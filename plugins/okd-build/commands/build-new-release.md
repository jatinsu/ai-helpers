---
description: Build OKD SCOS release from builds.json image manifest
argument-hint: <builds-json-path> [--registry=<registry>] [--base-release=<image>] [--bash]
---

## Name
okd-build:build-new-release

## Synopsis
```
/okd-build:build-new-release <builds-json-path> [--registry=<registry>] [--base-release=<image>] [--bash]
```

## Description
The `okd-build:build-new-release` command automates the migration of OpenShift Container Platform (OCP) operators from RHEL-based images to OKD Stream CoreOS (SCOS) by processing a builds.json image manifest file. It extracts source repository metadata using skopeo, clones operator repositories, transforms Dockerfiles for SCOS compatibility, builds container images, and orchestrates the creation of a custom OKD release payload.

This command streamlines the process of rebuilding an entire OKD release (~190 components) by:
- Extracting VCS metadata from image labels using `skopeo inspect`
- Cloning source repositories from discovered URLs
- Discovering and transforming Dockerfiles from RHEL to SCOS base images
- Building operators with appropriate SCOS tags
- Handling unbuildable components gracefully (using original digests)
- Creating a custom release payload using `oc adm release new`
- Optionally generating a bash script for manual execution with `--bash` flag

## Implementation

### Phase 1: Input Validation & Environment Setup

#### 1.1 Validate Command Arguments
```bash
# Check that builds.json path is provided
if [ -z "$builds_json_path" ]; then
  echo "Error: Missing required argument <builds-json-path>"
  echo "Usage: /okd-build:build-new-release <builds-json-path> [OPTIONS]"
  exit 1
fi

# Check builds.json exists and is readable
if [ ! -f "$builds_json_path" ]; then
  echo "Error: builds.json not found at: $builds_json_path"
  exit 1
fi

# Validate JSON syntax
if ! jq empty "$builds_json_path" 2>/dev/null; then
  echo "Error: Invalid JSON in builds.json"
  echo "Please ensure the file contains valid JSON"
  exit 1
fi

# Count total components
TOTAL_COMPONENTS=$(jq 'length' "$builds_json_path")
echo "Found $TOTAL_COMPONENTS components in builds.json"
echo ""
```

#### 1.2 Check Prerequisites
```bash
# Required tools for all modes
REQUIRED_TOOLS=("skopeo" "git" "jq" "curl")

# Additional tools for direct execution mode
if [ "$bash_mode" != "true" ]; then
  REQUIRED_TOOLS+=("oc")
fi

# Check each required tool
MISSING_TOOLS=()
for tool in "${REQUIRED_TOOLS[@]}"; do
  if ! which "$tool" > /dev/null 2>&1; then
    MISSING_TOOLS+=("$tool")
  fi
done

# Report missing tools
if [ ${#MISSING_TOOLS[@]} -gt 0 ]; then
  echo "Error: Missing required tools: ${MISSING_TOOLS[*]}"
  echo ""
  echo "Installation instructions:"
  for tool in "${MISSING_TOOLS[@]}"; do
    case "$tool" in
      skopeo)
        echo "  skopeo: https://github.com/containers/skopeo/blob/main/install.md"
        ;;
      git)
        echo "  git: https://git-scm.com/downloads"
        ;;
      jq)
        echo "  jq: https://stedolan.github.io/jq/download/"
        ;;
      curl)
        echo "  curl: Usually pre-installed on most systems"
        ;;
      oc)
        echo "  oc: https://docs.openshift.com/container-platform/latest/cli_reference/openshift_cli/getting-started-cli.html"
        ;;
    esac
  done
  exit 1
fi

# Determine container engine (podman preferred, docker fallback)
if which podman > /dev/null 2>&1; then
  CONTAINER_ENGINE="podman"
  echo "Using container engine: podman"
elif which docker > /dev/null 2>&1; then
  CONTAINER_ENGINE="docker"
  echo "Using container engine: docker"
else
  echo "Error: No container engine found (need podman or docker)"
  exit 1
fi
echo ""
```

#### 1.3 Setup Working Directory
```bash
# Create unique session ID using timestamp
SESSION_ID=$(date +%Y%m%d-%H%M%S)
WORK_DIR=".work/okd-build/${SESSION_ID}"

# Create directory structure
mkdir -p "${WORK_DIR}"/{repos,build_logs,metadata}

echo "Working directory: ${WORK_DIR}"
echo ""

# Copy builds.json to working directory for reference
cp "$builds_json_path" "${WORK_DIR}/builds.json"
```

#### 1.4 Initialize Tracking Files
```bash
# Create JSON tracking files for state management
echo '{}' > "${WORK_DIR}/metadata_extraction.json"    # Skopeo inspection results
echo '[]' > "${WORK_DIR}/buildable.json"              # Components with Dockerfiles
echo '[]' > "${WORK_DIR}/unbuildable.json"            # Components without Dockerfiles
echo '{}' > "${WORK_DIR}/build_results.json"          # Build success/failure status
echo '{}' > "${WORK_DIR}/image_replacements.json"     # New digest mappings
```

#### 1.5 Set Default Values and Validate Configuration
```bash
# Set registry default if not provided
if [ -z "$registry" ]; then
  registry="quay.io/${USER}"
  echo "Using default registry: ${registry}"
fi

# Validate registry authentication (skip in bash mode)
if [ "$bash_mode" != "true" ]; then
  REGISTRY_HOST=$(echo "${registry}" | cut -d'/' -f1)
  if ! ${CONTAINER_ENGINE} login --get-login "${REGISTRY_HOST}" > /dev/null 2>&1; then
    echo "Warning: Not logged into ${REGISTRY_HOST}"
    echo "To authenticate, run: ${CONTAINER_ENGINE} login ${REGISTRY_HOST}"
    echo ""
    echo "Continue without authentication? [y/N]"
    read -r response
    if [[ ! "$response" =~ ^[Yy]$ ]]; then
      echo "Aborted. Please authenticate and try again."
      exit 1
    fi
  fi
fi

# Fetch base-release if not provided
if [ -z "$base_release" ]; then
  echo "Fetching latest SCOS release from OKD API..."
  SCOS_VERSION=$(curl -s https://amd64.origin.releases.ci.openshift.org/api/v1/releasestreams/accepted | jq -r '.["4-scos-next"][0]')

  if [ -z "$SCOS_VERSION" ] || [ "$SCOS_VERSION" = "null" ]; then
    echo "Error: Failed to fetch latest SCOS release from OKD API"
    echo "Please specify --base-release manually"
    exit 1
  fi

  base_release="quay.io/okd/scos-release:${SCOS_VERSION}"
  echo "Using base release: ${base_release}"
else
  echo "Using specified base release: ${base_release}"
fi
echo ""
```

---

### Phase 2: Metadata Extraction with Skopeo

#### 2.1 Display Progress Header
```bash
echo "========================================"
echo "Phase 2: Extracting Source Metadata"
echo "========================================"
echo ""
echo "Processing $TOTAL_COMPONENTS components..."
echo "This may take several minutes..."
echo ""
```

#### 2.2 Iterate Through builds.json and Extract Metadata
```bash
# Counter for progress tracking
COUNTER=0

# Use jq to iterate through each component
jq -r 'to_entries[] | "\(.key)|\(.value)"' "${WORK_DIR}/builds.json" | \
while IFS='|' read -r component digest; do
  ((COUNTER++))
  echo "[$COUNTER/$TOTAL_COMPONENTS] ${component}"

  # Run skopeo inspect to get image metadata
  METADATA_FILE="${WORK_DIR}/metadata/${component}.json"

  echo "  Inspecting docker://${digest}..."
  if ! skopeo inspect "docker://${digest}" > "$METADATA_FILE" 2>&1; then
    echo "  ⚠️  Skopeo inspect failed"

    # Record failure in unbuildable list
    jq --arg comp "$component" \
       --arg reason "skopeo_failed" \
       --arg digest "$digest" \
       '. += [{"component": $comp, "reason": $reason, "digest": $digest}]' \
       "${WORK_DIR}/unbuildable.json" > "${WORK_DIR}/unbuildable.json.tmp"
    mv "${WORK_DIR}/unbuildable.json.tmp" "${WORK_DIR}/unbuildable.json"

    continue
  fi

  # Extract VCS metadata from labels/annotations
  # Try multiple possible label names for source URL
  VCS_URL=$(jq -r '
    .Labels["io.openshift.build.source-location"] //
    .Labels["vcs-url"] //
    .Labels["org.opencontainers.image.source"] //
    .Annotations["io.openshift.build.source-location"] //
    empty
  ' "$METADATA_FILE")

  # Extract VCS ref (branch/tag)
  VCS_REF=$(jq -r '
    .Labels["io.openshift.build.commit.ref"] //
    .Labels["vcs-ref"] //
    .Labels["io.openshift.build.commit.id"] //
    .Annotations["io.openshift.build.commit.ref"] //
    "master"
  ' "$METADATA_FILE")

  # Handle null/empty VCS_REF
  if [ "$VCS_REF" = "null" ] || [ -z "$VCS_REF" ]; then
    VCS_REF="master"
  fi

  # Validate VCS URL exists
  if [ -z "$VCS_URL" ] || [ "$VCS_URL" = "null" ]; then
    echo "  ⚠️  No VCS URL found in image metadata"

    # Record as unbuildable
    jq --arg comp "$component" \
       --arg reason "no_vcs_url" \
       --arg digest "$digest" \
       '. += [{"component": $comp, "reason": $reason, "digest": $digest}]' \
       "${WORK_DIR}/unbuildable.json" > "${WORK_DIR}/unbuildable.json.tmp"
    mv "${WORK_DIR}/unbuildable.json.tmp" "${WORK_DIR}/unbuildable.json"

    continue
  fi

  echo "  ✓ VCS: ${VCS_URL}"
  echo "  ✓ Ref: ${VCS_REF}"

  # Record successful metadata extraction
  jq --arg comp "$component" \
     --arg vcs "$VCS_URL" \
     --arg ref "$VCS_REF" \
     --arg digest "$digest" \
     '.[$comp] = {"vcs_url": $vcs, "vcs_ref": $ref, "original_digest": $digest}' \
     "${WORK_DIR}/metadata_extraction.json" > "${WORK_DIR}/metadata_extraction.json.tmp"
  mv "${WORK_DIR}/metadata_extraction.json.tmp" "${WORK_DIR}/metadata_extraction.json"

done
```

#### 2.3 Summary of Metadata Extraction
```bash
echo ""
echo "Metadata extraction complete:"
EXTRACTED_COUNT=$(jq 'length' "${WORK_DIR}/metadata_extraction.json")
FAILED_COUNT=$(jq 'length' "${WORK_DIR}/unbuildable.json")
echo "  Successfully extracted: ${EXTRACTED_COUNT}"
echo "  Failed extraction: ${FAILED_COUNT}"
echo ""
```

---

### Phase 3: Repository Cloning & Dockerfile Discovery

#### 3.1 Display Progress Header
```bash
echo "========================================"
echo "Phase 3: Cloning Repositories"
echo "========================================"
echo ""
```

#### 3.2 Clone Repositories
```bash
# Iterate through successfully extracted metadata
jq -r 'to_entries[] | "\(.key)|\(.value.vcs_url)|\(.value.vcs_ref)"' \
   "${WORK_DIR}/metadata_extraction.json" | \
while IFS='|' read -r component vcs_url vcs_ref; do
  echo "Cloning: ${component}"
  echo "  Repository: ${vcs_url}"
  echo "  Ref: ${vcs_ref}"

  REPO_DIR="${WORK_DIR}/repos/${component}"
  CLONE_LOG="${WORK_DIR}/build_logs/${component}_clone.log"

  # Try to clone with specified ref
  if ! git clone --depth 1 --branch "$vcs_ref" "$vcs_url" "$REPO_DIR" > "$CLONE_LOG" 2>&1; then
    # Try master -> main fallback if original ref was master
    if [ "$vcs_ref" = "master" ]; then
      echo "  Branch 'master' not found, trying 'main'..."
      if git clone --depth 1 --branch "main" "$vcs_url" "$REPO_DIR" >> "$CLONE_LOG" 2>&1; then
        echo "  ✓ Cloned from 'main' branch"

        # Update vcs_ref in metadata to reflect actual branch used
        jq --arg comp "$component" \
           '.[$comp].vcs_ref = "main"' \
           "${WORK_DIR}/metadata_extraction.json" > "${WORK_DIR}/metadata_extraction.json.tmp"
        mv "${WORK_DIR}/metadata_extraction.json.tmp" "${WORK_DIR}/metadata_extraction.json"
      else
        echo "  ⚠️  Clone failed (tried 'master' and 'main')"

        # Record clone failure
        jq --arg comp "$component" \
           --arg reason "clone_failed" \
           --arg vcs "$vcs_url" \
           --arg ref "$vcs_ref" \
           '. += [{"component": $comp, "reason": $reason, "vcs_url": $vcs, "vcs_ref": $ref}]' \
           "${WORK_DIR}/unbuildable.json" > "${WORK_DIR}/unbuildable.json.tmp"
        mv "${WORK_DIR}/unbuildable.json.tmp" "${WORK_DIR}/unbuildable.json"
        continue
      fi
    else
      echo "  ⚠️  Clone failed"

      # Record clone failure
      jq --arg comp "$component" \
         --arg reason "clone_failed" \
         --arg vcs "$vcs_url" \
         --arg ref "$vcs_ref" \
         '. += [{"component": $comp, "reason": $reason, "vcs_url": $vcs, "vcs_ref": $ref}]' \
         "${WORK_DIR}/unbuildable.json" > "${WORK_DIR}/unbuildable.json.tmp"
      mv "${WORK_DIR}/unbuildable.json.tmp" "${WORK_DIR}/unbuildable.json"
      continue
    fi
  else
    echo "  ✓ Cloned successfully"
  fi

  # Get actual commit SHA for tagging images
  COMMIT_SHA=$(cd "$REPO_DIR" && git rev-parse --short HEAD)
  echo "  Commit: ${COMMIT_SHA}"

  # Update metadata with commit SHA
  jq --arg comp "$component" \
     --arg sha "$COMMIT_SHA" \
     '.[$comp].commit_sha = $sha' \
     "${WORK_DIR}/metadata_extraction.json" > "${WORK_DIR}/metadata_extraction.json.tmp"
  mv "${WORK_DIR}/metadata_extraction.json.tmp" "${WORK_DIR}/metadata_extraction.json"
done
```

#### 3.3 Dockerfile Discovery
```bash
echo ""
echo "Discovering Dockerfiles..."
echo ""

# Iterate through cloned repositories
jq -r 'to_entries[] | .key' "${WORK_DIR}/metadata_extraction.json" | \
while read -r component; do
  REPO_DIR="${WORK_DIR}/repos/${component}"

  # Skip if directory doesn't exist (clone failed)
  [ ! -d "$REPO_DIR" ] && continue

  echo "Searching: ${component}"

  # Search for Dockerfiles in priority order
  DOCKERFILE=""

  # Priority 1: Root Dockerfile
  if [ -f "${REPO_DIR}/Dockerfile" ]; then
    DOCKERFILE="${REPO_DIR}/Dockerfile"
    echo "  ✓ Found: Dockerfile"

  # Priority 2: Dockerfile.rhel (common for OCP operators)
  elif [ -f "${REPO_DIR}/Dockerfile.rhel" ]; then
    DOCKERFILE="${REPO_DIR}/Dockerfile.rhel"
    echo "  ✓ Found: Dockerfile.rhel"

  # Priority 3: Dockerfile.rhel9
  elif [ -f "${REPO_DIR}/Dockerfile.rhel9" ]; then
    DOCKERFILE="${REPO_DIR}/Dockerfile.rhel9"
    echo "  ✓ Found: Dockerfile.rhel9"

  # Priority 4: openshift/Dockerfile
  elif [ -f "${REPO_DIR}/openshift/Dockerfile" ]; then
    DOCKERFILE="${REPO_DIR}/openshift/Dockerfile"
    echo "  ✓ Found: openshift/Dockerfile"

  # Priority 5: build/Dockerfile
  elif [ -f "${REPO_DIR}/build/Dockerfile" ]; then
    DOCKERFILE="${REPO_DIR}/build/Dockerfile"
    echo "  ✓ Found: build/Dockerfile"

  # Priority 6: images/Dockerfile
  elif [ -f "${REPO_DIR}/images/Dockerfile" ]; then
    DOCKERFILE="${REPO_DIR}/images/Dockerfile"
    echo "  ✓ Found: images/Dockerfile"

  # Priority 7: Find most recently modified Dockerfile
  else
    DOCKERFILE=$(find "$REPO_DIR" -name "Dockerfile*" -type f -printf '%T@ %p\n' 2>/dev/null | \
                 sort -rn | head -1 | cut -d' ' -f2-)

    if [ -n "$DOCKERFILE" ]; then
      REL_PATH=$(echo "$DOCKERFILE" | sed "s|${REPO_DIR}/||")
      echo "  ✓ Found: ${REL_PATH} (most recent)"
    fi
  fi

  # Record result
  if [ -n "$DOCKERFILE" ] && [ -f "$DOCKERFILE" ]; then
    # Found a Dockerfile - add to buildable list
    REL_DOCKERFILE=$(echo "$DOCKERFILE" | sed "s|${REPO_DIR}/||")

    jq --arg comp "$component" \
       --arg df "$REL_DOCKERFILE" \
       '. += [{"component": $comp, "dockerfile": $df}]' \
       "${WORK_DIR}/buildable.json" > "${WORK_DIR}/buildable.json.tmp"
    mv "${WORK_DIR}/buildable.json.tmp" "${WORK_DIR}/buildable.json"

    # Also update metadata
    jq --arg comp "$component" \
       --arg df "$REL_DOCKERFILE" \
       '.[$comp].dockerfile = $df' \
       "${WORK_DIR}/metadata_extraction.json" > "${WORK_DIR}/metadata_extraction.json.tmp"
    mv "${WORK_DIR}/metadata_extraction.json.tmp" "${WORK_DIR}/metadata_extraction.json"
  else
    # No Dockerfile found
    echo "  ⚠️  No Dockerfile found"

    jq --arg comp "$component" \
       --arg reason "no_dockerfile" \
       '. += [{"component": $comp, "reason": $reason}]' \
       "${WORK_DIR}/unbuildable.json" > "${WORK_DIR}/unbuildable.json.tmp"
    mv "${WORK_DIR}/unbuildable.json.tmp" "${WORK_DIR}/unbuildable.json"
  fi
done
```

#### 3.4 Summary of Repository Cloning
```bash
echo ""
echo "Repository cloning complete:"
BUILDABLE_COUNT=$(jq 'length' "${WORK_DIR}/buildable.json")
UNBUILDABLE_COUNT=$(jq 'length' "${WORK_DIR}/unbuildable.json")
echo "  Buildable components: ${BUILDABLE_COUNT}"
echo "  Unbuildable components: ${UNBUILDABLE_COUNT}"
echo ""
```

---

### Phase 4: Dockerfile Transformation (RHEL → SCOS)

#### 4.1 Display Progress Header
```bash
echo "========================================"
echo "Phase 4: Transforming Dockerfiles"
echo "========================================"
echo ""
echo "Transforming RHEL base images to SCOS..."
echo ""
```

#### 4.2 Transform Each Dockerfile
```bash
# Iterate through buildable components
jq -r '.[] | "\(.component)|\(.dockerfile)"' "${WORK_DIR}/buildable.json" | \
while IFS='|' read -r component dockerfile; do
  echo "Transforming: ${component}"

  REPO_DIR="${WORK_DIR}/repos/${component}"
  DOCKERFILE_PATH="${REPO_DIR}/${dockerfile}"

  echo "  File: ${dockerfile}"

  # Create backup before modification
  cp "$DOCKERFILE_PATH" "${DOCKERFILE_PATH}.backup"

  # Perform SCOS transformation
  # Pattern: FROM registry.ci.openshift.org/ocp/4.XX:base-rhel[89]
  # Replacement: FROM registry.ci.openshift.org/origin/scos-4.XX:base-stream[89]

  TRANSFORM_COUNT=0

  # Transform RHEL9 base images
  # IMPORTANT: Only transform base images, NOT builder or golang images
  if grep -q "FROM registry.ci.openshift.org/ocp/[0-9.]*:base-rhel9" "$DOCKERFILE_PATH"; then
    sed -i 's|FROM registry\.ci\.openshift\.org/ocp/\([0-9.]*\):base-rhel9|FROM registry.ci.openshift.org/origin/scos-\1:base-stream9|g' "$DOCKERFILE_PATH"
    ((TRANSFORM_COUNT++))
    echo "  ✓ Transformed RHEL9 base image"
  fi

  # Transform RHEL8 base images
  if grep -q "FROM registry.ci.openshift.org/ocp/[0-9.]*:base-rhel8" "$DOCKERFILE_PATH"; then
    sed -i 's|FROM registry\.ci\.openshift\.org/ocp/\([0-9.]*\):base-rhel8|FROM registry.ci.openshift.org/origin/scos-\1:base-stream8|g' "$DOCKERFILE_PATH"
    ((TRANSFORM_COUNT++))
    echo "  ✓ Transformed RHEL8 base image"
  fi

  # Check if any transformations were made
  if [ $TRANSFORM_COUNT -eq 0 ]; then
    echo "  ℹ️  No base images to transform (may already be SCOS or use different base)"
  fi

  # Generate diff for verification
  if ! diff -u "${DOCKERFILE_PATH}.backup" "$DOCKERFILE_PATH" > "${WORK_DIR}/build_logs/${component}_dockerfile.diff" 2>&1; then
    echo "  Diff saved to: ${WORK_DIR}/build_logs/${component}_dockerfile.diff"
  fi
done

echo ""
echo "Dockerfile transformation complete"
echo ""
```

---

### Phase 5: User Review & Confirmation

#### 5.1 Display Unbuildable Components Report
```bash
echo "========================================"
echo "Build Plan Review"
echo "========================================"
echo ""

# Show unbuildable components
UNBUILDABLE_COUNT=$(jq 'length' "${WORK_DIR}/unbuildable.json")

if [ $UNBUILDABLE_COUNT -gt 0 ]; then
  echo "⚠️  Unbuildable Components: ${UNBUILDABLE_COUNT}"
  echo ""

  # Group by reason
  echo "Breakdown by reason:"

  NO_VCS=$(jq '[.[] | select(.reason == "no_vcs_url")] | length' "${WORK_DIR}/unbuildable.json")
  CLONE_FAILED=$(jq '[.[] | select(.reason == "clone_failed")] | length' "${WORK_DIR}/unbuildable.json")
  NO_DOCKERFILE=$(jq '[.[] | select(.reason == "no_dockerfile")] | length' "${WORK_DIR}/unbuildable.json")
  SKOPEO_FAILED=$(jq '[.[] | select(.reason == "skopeo_failed")] | length' "${WORK_DIR}/unbuildable.json")

  echo "  No VCS URL in metadata: ${NO_VCS}"
  echo "  Clone failed: ${CLONE_FAILED}"
  echo "  No Dockerfile found: ${NO_DOCKERFILE}"
  echo "  Skopeo inspection failed: ${SKOPEO_FAILED}"
  echo ""

  # Show detailed list (first 20)
  echo "Components (showing first 20):"
  jq -r '.[:20][] | "  • \(.component) - \(.reason)"' "${WORK_DIR}/unbuildable.json"

  if [ $UNBUILDABLE_COUNT -gt 20 ]; then
    echo "  ... and $((UNBUILDABLE_COUNT - 20)) more"
  fi
  echo ""

  # Save full list to file
  echo "Full list saved to: ${WORK_DIR}/unbuildable_components.txt"
  jq -r '.[] | "\(.component)\t\(.reason)"' "${WORK_DIR}/unbuildable.json" > "${WORK_DIR}/unbuildable_components.txt"
  echo ""
fi
```

#### 5.2 Display Buildable Components Summary
```bash
BUILDABLE_COUNT=$(jq 'length' "${WORK_DIR}/buildable.json")

echo "✓ Buildable Components: ${BUILDABLE_COUNT}"
echo ""
echo "These components will be built from source with SCOS transformations."
echo ""

# Show sample (first 10)
if [ $BUILDABLE_COUNT -gt 0 ]; then
  echo "Sample components (showing first 10):"
  jq -r '.[:10][] | "  • \(.component) - \(.dockerfile)"' "${WORK_DIR}/buildable.json"

  if [ $BUILDABLE_COUNT -gt 10 ]; then
    echo "  ... and $((BUILDABLE_COUNT - 10)) more"
  fi
  echo ""

  # Save full list
  echo "Full list saved to: ${WORK_DIR}/buildable_components.txt"
  jq -r '.[] | "\(.component)\t\(.dockerfile)"' "${WORK_DIR}/buildable.json" > "${WORK_DIR}/buildable_components.txt"
  echo ""
fi
```

#### 5.3 Ask for User Confirmation
```bash
echo "========================================"
echo "Action Required"
echo "========================================"
echo ""
echo "Build Plan:"
echo "  • ${BUILDABLE_COUNT} components will be built from source"
echo "  • ${UNBUILDABLE_COUNT} components will use original digests from builds.json"
echo ""
echo "IMPORTANT: Unbuildable components will NOT block release creation."
echo "The final release will include:"
echo "  - New digests for successfully built components"
echo "  - Original digests for unbuildable/failed components"
echo ""

if [ "$bash_mode" = "true" ]; then
  echo "Bash script mode: Will generate build script"
  echo ""
  echo "Proceed with script generation? [y/N]"
else
  echo "Direct execution mode: Will build and create release"
  echo ""
  echo "Proceed with builds? [y/N]"
fi

read -r response
if [[ ! "$response" =~ ^[Yy]$ ]]; then
  echo ""
  echo "Aborted by user"
  echo ""
  echo "Working directory preserved: ${WORK_DIR}"
  echo "You can review:"
  echo "  - Buildable: ${WORK_DIR}/buildable.json"
  echo "  - Unbuildable: ${WORK_DIR}/unbuildable.json"
  echo "  - Metadata: ${WORK_DIR}/metadata_extraction.json"
  exit 0
fi

echo ""
```

---

### Phase 6: Build Execution (Two Modes)

#### 6.1 Bash Script Generation Mode (--bash flag)

```bash
if [ "$bash_mode" = "true" ]; then
  echo "========================================"
  echo "Generating Build Script"
  echo "========================================"
  echo ""

  SCRIPT_PATH="./build-okd-release-${SESSION_ID}.sh"

  # Generate script header
  cat > "$SCRIPT_PATH" << 'SCRIPT_HEADER'
#!/bin/bash
set -e

# Generated by /okd-build:build-new-release
# Generated on: $(date)

echo "========================================"
echo "OKD SCOS Release Builder"
echo "========================================"
echo ""

# Configuration (will be replaced with actual values)
WORK_DIR="WORK_DIR_PLACEHOLDER"
REGISTRY="REGISTRY_PLACEHOLDER"
BASE_RELEASE="BASE_RELEASE_PLACEHOLDER"
CONTAINER_ENGINE="CONTAINER_ENGINE_PLACEHOLDER"
SESSION_ID="SESSION_ID_PLACEHOLDER"
BUILDABLE_COUNT=BUILDABLE_COUNT_PLACEHOLDER

# Counters
BUILD_SUCCESS=0
BUILD_FAILED=0

echo "Build configuration:"
echo "  Working directory: ${WORK_DIR}"
echo "  Registry: ${REGISTRY}"
echo "  Base release: ${BASE_RELEASE}"
echo "  Container engine: ${CONTAINER_ENGINE}"
echo "  Components to build: ${BUILDABLE_COUNT}"
echo ""
echo "Starting builds..."
echo ""

SCRIPT_HEADER

  # Replace placeholders with actual values
  sed -i "s|WORK_DIR_PLACEHOLDER|${WORK_DIR}|g" "$SCRIPT_PATH"
  sed -i "s|REGISTRY_PLACEHOLDER|${registry}|g" "$SCRIPT_PATH"
  sed -i "s|BASE_RELEASE_PLACEHOLDER|${base_release}|g" "$SCRIPT_PATH"
  sed -i "s|CONTAINER_ENGINE_PLACEHOLDER|${CONTAINER_ENGINE}|g" "$SCRIPT_PATH"
  sed -i "s|SESSION_ID_PLACEHOLDER|${SESSION_ID}|g" "$SCRIPT_PATH"
  sed -i "s|BUILDABLE_COUNT_PLACEHOLDER|${BUILDABLE_COUNT}|g" "$SCRIPT_PATH"

  # Add build commands for each component
  jq -r '.[] | "\(.component)|\(.dockerfile)"' "${WORK_DIR}/buildable.json" | \
  while IFS='|' read -r component dockerfile; do
    # Get metadata
    vcs_url=$(jq -r --arg comp "$component" '.[$comp].vcs_url' "${WORK_DIR}/metadata_extraction.json")
    vcs_ref=$(jq -r --arg comp "$component" '.[$comp].vcs_ref' "${WORK_DIR}/metadata_extraction.json")
    commit_sha=$(jq -r --arg comp "$component" '.[$comp].commit_sha' "${WORK_DIR}/metadata_extraction.json")

    # Add build section to script
    cat >> "$SCRIPT_PATH" << SCRIPT_BUILD
echo "========================================"
echo "Building: ${component}"
echo "========================================"
echo "Repository: ${vcs_url}"
echo "Ref: ${vcs_ref}"
echo "Commit: ${commit_sha}"
echo "Dockerfile: ${dockerfile}"
echo ""

REPO_DIR="\${WORK_DIR}/repos/${component}"
BUILD_LOG="\${WORK_DIR}/build_logs/${component}_build.log"
DOCKERFILE_PATH="\${REPO_DIR}/${dockerfile}"
BUILD_CONTEXT="\$(dirname "\${DOCKERFILE_PATH}")"

# If Dockerfile is in root, use repo root as context
if [ "\${BUILD_CONTEXT}" = "\${REPO_DIR}" ]; then
  BUILD_CONTEXT="\${REPO_DIR}"
fi

IMAGE_TAG="\${REGISTRY}/${component}:${commit_sha}"
IMAGE_LATEST="\${REGISTRY}/${component}:latest"

echo "Building image..."
if \${CONTAINER_ENGINE} build \\
  -t "\${IMAGE_TAG}" \\
  -f "\${DOCKERFILE_PATH}" \\
  --build-arg TAGS=scos \\
  "\${BUILD_CONTEXT}" \\
  > "\${BUILD_LOG}" 2>&1; then

  echo "✓ Build successful"

  # Tag as latest
  \${CONTAINER_ENGINE} tag "\${IMAGE_TAG}" "\${IMAGE_LATEST}"

  # Push both tags
  echo "Pushing to registry..."
  \${CONTAINER_ENGINE} push "\${IMAGE_TAG}" >> "\${BUILD_LOG}" 2>&1
  \${CONTAINER_ENGINE} push "\${IMAGE_LATEST}" >> "\${BUILD_LOG}" 2>&1

  # Get image digest using skopeo
  IMAGE_DIGEST=\$(skopeo inspect "docker://\${IMAGE_TAG}" 2>/dev/null | jq -r '.Digest')

  if [ -n "\${IMAGE_DIGEST}" ] && [ "\${IMAGE_DIGEST}" != "null" ]; then
    IMAGE_WITH_DIGEST="\${REGISTRY}/${component}@\${IMAGE_DIGEST}"
    echo "✓ Pushed: \${IMAGE_WITH_DIGEST}"

    # Record in replacements file
    jq --arg comp "${component}" --arg img "\${IMAGE_WITH_DIGEST}" \\
       '.[\$comp] = \$img' \\
       "\${WORK_DIR}/image_replacements.json" > "\${WORK_DIR}/image_replacements.json.tmp"
    mv "\${WORK_DIR}/image_replacements.json.tmp" "\${WORK_DIR}/image_replacements.json"

    ((BUILD_SUCCESS++))
  else
    echo "⚠️  Could not get image digest"
    ((BUILD_FAILED++))
  fi
else
  echo "❌ Build failed"
  echo "See log: \${BUILD_LOG}"
  ((BUILD_FAILED++))
fi

echo ""

SCRIPT_BUILD
  done

  # Add release creation section
  cat >> "$SCRIPT_PATH" << 'SCRIPT_RELEASE'

echo "========================================"
echo "Build Summary"
echo "========================================"
echo "Successful: ${BUILD_SUCCESS}"
echo "Failed: ${BUILD_FAILED}"
echo ""

if [ ${BUILD_SUCCESS} -eq 0 ]; then
  echo "❌ No components were successfully built"
  echo "Cannot create release"
  exit 1
fi

echo "========================================"
echo "Creating Release Image"
echo "========================================"
echo ""

# Merge new digests with original digests from builds.json
echo "Merging component digests..."

# Create final mappings: use new digests where available, otherwise use original
jq -s '.[0] as $original | .[1] as $new |
  $original | to_entries | map(
    .key as $comp |
    if $new[$comp] then {key: $comp, value: $new[$comp]}
    else {key: $comp, value: .value}
    end
  ) | from_entries' \
  "${WORK_DIR}/builds.json" \
  "${WORK_DIR}/image_replacements.json" \
  > "${WORK_DIR}/final_mappings.json"

FINAL_COUNT=$(jq 'length' "${WORK_DIR}/final_mappings.json")
echo "Total components in release: ${FINAL_COUNT}"
echo "Newly built: ${BUILD_SUCCESS}"
echo "Using original digests: $((FINAL_COUNT - BUILD_SUCCESS))"
echo ""

# Check if cluster-version-operator was built
CVO_IMAGE=$(jq -r '.["cluster-version-operator"] // empty' "${WORK_DIR}/image_replacements.json")

# Build oc adm release new command
RELEASE_CMD="oc adm release new"
RELEASE_CMD="$RELEASE_CMD --from-release=${BASE_RELEASE}"

if [ -n "$CVO_IMAGE" ]; then
  echo "Note: cluster-version-operator was rebuilt"
  echo "Using --to-image-base flag"
  RELEASE_CMD="$RELEASE_CMD --to-image-base=${CVO_IMAGE}"
fi

# Add all component mappings
while IFS= read -r line; do
  component=$(echo "$line" | jq -r '.key')
  image=$(echo "$line" | jq -r '.value')
  RELEASE_CMD="$RELEASE_CMD ${component}=${image}"
done < <(jq -r 'to_entries[] | @json' "${WORK_DIR}/final_mappings.json")

# Add output image and flags
OUTPUT_IMAGE="${REGISTRY}/okd-release:custom-${SESSION_ID}"
RELEASE_CMD="$RELEASE_CMD --to-image=${OUTPUT_IMAGE}"
RELEASE_CMD="$RELEASE_CMD --keep-manifest-list"
RELEASE_CMD="$RELEASE_CMD --allow-missing-images"

# Save command for reference
echo "$RELEASE_CMD" > "${WORK_DIR}/release_command.txt"

echo "Executing release creation..."
echo ""

if eval "$RELEASE_CMD" > "${WORK_DIR}/release_creation.log" 2>&1; then
  echo "========================================"
  echo "✓ Release Created Successfully"
  echo "========================================"
  echo ""
  echo "Release image: ${OUTPUT_IMAGE}"
  echo ""
  echo "To use this release:"
  echo "  oc adm upgrade --to-image=${OUTPUT_IMAGE}"
  echo ""
else
  echo "❌ Release creation failed"
  echo "See log: ${WORK_DIR}/release_creation.log"
  exit 1
fi

SCRIPT_RELEASE

  # Make script executable
  chmod +x "$SCRIPT_PATH"

  echo "✓ Build script generated: ${SCRIPT_PATH}"
  echo ""
  echo "To execute:"
  echo "  ${SCRIPT_PATH}"
  echo ""
  echo "The script will:"
  echo "  1. Build ${BUILDABLE_COUNT} components"
  echo "  2. Push images to ${registry}"
  echo "  3. Create release image using ${base_release}"
  echo ""

  exit 0
fi
```

#### 6.2 Direct Execution Mode

```bash
echo "========================================"
echo "Phase 6: Building Components"
echo "========================================"
echo ""

# Initialize counters
BUILD_SUCCESS=0
BUILD_FAILED=0
BUILD_INDEX=0

# Build each component
jq -r '.[] | "\(.component)|\(.dockerfile)"' "${WORK_DIR}/buildable.json" | \
while IFS='|' read -r component dockerfile; do
  ((BUILD_INDEX++))

  echo ""
  echo "[${BUILD_INDEX}/${BUILDABLE_COUNT}] ${component}"
  echo "========================================"

  # Get metadata
  vcs_url=$(jq -r --arg comp "$component" '.[$comp].vcs_url' "${WORK_DIR}/metadata_extraction.json")
  vcs_ref=$(jq -r --arg comp "$component" '.[$comp].vcs_ref' "${WORK_DIR}/metadata_extraction.json")
  commit_sha=$(jq -r --arg comp "$component" '.[$comp].commit_sha' "${WORK_DIR}/metadata_extraction.json")

  echo "Repository: ${vcs_url}"
  echo "Ref: ${vcs_ref}"
  echo "Commit: ${commit_sha}"
  echo "Dockerfile: ${dockerfile}"
  echo ""

  REPO_DIR="${WORK_DIR}/repos/${component}"
  BUILD_LOG="${WORK_DIR}/build_logs/${component}_build.log"
  DOCKERFILE_PATH="${REPO_DIR}/${dockerfile}"

  # Determine build context (directory containing Dockerfile)
  BUILD_CONTEXT=$(dirname "$DOCKERFILE_PATH")
  if [ "$BUILD_CONTEXT" = "$REPO_DIR" ]; then
    BUILD_CONTEXT="$REPO_DIR"
  fi

  IMAGE_TAG="${registry}/${component}:${commit_sha}"
  IMAGE_LATEST="${registry}/${component}:latest"

  # Build image
  echo "Building image..."
  if ${CONTAINER_ENGINE} build \
    -t "$IMAGE_TAG" \
    -f "$DOCKERFILE_PATH" \
    --build-arg TAGS=scos \
    "$BUILD_CONTEXT" \
    > "$BUILD_LOG" 2>&1; then

    echo "✓ Build successful"

    # Tag as latest
    ${CONTAINER_ENGINE} tag "$IMAGE_TAG" "$IMAGE_LATEST"

    # Push to registry
    echo "Pushing to registry..."
    if ${CONTAINER_ENGINE} push "$IMAGE_TAG" >> "$BUILD_LOG" 2>&1 && \
       ${CONTAINER_ENGINE} push "$IMAGE_LATEST" >> "$BUILD_LOG" 2>&1; then

      # Get image digest using skopeo
      IMAGE_DIGEST=$(skopeo inspect "docker://${IMAGE_TAG}" 2>/dev/null | jq -r '.Digest')

      if [ -n "$IMAGE_DIGEST" ] && [ "$IMAGE_DIGEST" != "null" ]; then
        IMAGE_WITH_DIGEST="${registry}/${component}@${IMAGE_DIGEST}"
        echo "✓ Pushed: ${IMAGE_WITH_DIGEST}"

        # Record in replacements
        jq --arg comp "$component" --arg img "$IMAGE_WITH_DIGEST" \
           '.[$comp] = $img' \
           "${WORK_DIR}/image_replacements.json" > "${WORK_DIR}/image_replacements.json.tmp"
        mv "${WORK_DIR}/image_replacements.json.tmp" "${WORK_DIR}/image_replacements.json"

        # Record success
        jq --arg comp "$component" --arg result "success" --arg img "$IMAGE_WITH_DIGEST" \
           '.[$comp] = {"result": $result, "image": $img}' \
           "${WORK_DIR}/build_results.json" > "${WORK_DIR}/build_results.json.tmp"
        mv "${WORK_DIR}/build_results.json.tmp" "${WORK_DIR}/build_results.json"

        ((BUILD_SUCCESS++))
      else
        echo "⚠️  Could not get image digest"
        ((BUILD_FAILED++))

        # Record failure
        jq --arg comp "$component" --arg result "failed" --arg reason "no_digest" \
           '.[$comp] = {"result": $result, "reason": $reason}' \
           "${WORK_DIR}/build_results.json" > "${WORK_DIR}/build_results.json.tmp"
        mv "${WORK_DIR}/build_results.json.tmp" "${WORK_DIR}/build_results.json"
      fi
    else
      echo "❌ Push failed"
      echo "See log: ${BUILD_LOG}"
      ((BUILD_FAILED++))

      # Record failure
      jq --arg comp "$component" --arg result "failed" --arg reason "push_failed" \
         '.[$comp] = {"result": $result, "reason": $reason, "log": "'$BUILD_LOG'"}' \
         "${WORK_DIR}/build_results.json" > "${WORK_DIR}/build_results.json.tmp"
      mv "${WORK_DIR}/build_results.json.tmp" "${WORK_DIR}/build_results.json"
    fi
  else
    echo "❌ Build failed"
    echo "See log: ${BUILD_LOG}"
    ((BUILD_FAILED++))

    # Record failure
    jq --arg comp "$component" --arg result "failed" --arg reason "build_failed" \
       '.[$comp] = {"result": $result, "reason": $reason, "log": "'$BUILD_LOG'"}' \
       "${WORK_DIR}/build_results.json" > "${WORK_DIR}/build_results.json.tmp"
    mv "${WORK_DIR}/build_results.json.tmp" "${WORK_DIR}/build_results.json"

    # Ask if should continue
    echo ""
    echo "Build failed for ${component}"
    echo "Continue with remaining components? [Y/n]"
    read -r response
    if [[ "$response" =~ ^[Nn]$ ]]; then
      echo "Aborted by user"
      exit 1
    fi
  fi
done

echo ""
echo "========================================"
echo "Build Summary"
echo "========================================"
echo "Successful: ${BUILD_SUCCESS}"
echo "Failed: ${BUILD_FAILED}"
echo ""
```

---

### Phase 7: Release Creation (Direct Mode Only)

```bash
if [ "$bash_mode" != "true" ]; then
  echo "========================================"
  echo "Phase 7: Creating Release Image"
  echo "========================================"
  echo ""

  # Verify we have successful builds
  SUCCESS_COUNT=$(jq 'length' "${WORK_DIR}/image_replacements.json")

  if [ "$SUCCESS_COUNT" -eq 0 ]; then
    echo "❌ No components were successfully built"
    echo "Cannot create release without any new images"
    exit 1
  fi

  echo "Creating release with ${SUCCESS_COUNT} new component images"
  echo ""

  # Merge new digests with original digests from builds.json
  echo "Merging component digests..."

  jq -s '.[0] as $original | .[1] as $new |
    $original | to_entries | map(
      .key as $comp |
      if $new[$comp] then {key: $comp, value: $new[$comp]}
      else {key: $comp, value: .value}
      end
    ) | from_entries' \
    "${WORK_DIR}/builds.json" \
    "${WORK_DIR}/image_replacements.json" \
    > "${WORK_DIR}/final_mappings.json"

  FINAL_COUNT=$(jq 'length' "${WORK_DIR}/final_mappings.json")
  echo "Total components in release: ${FINAL_COUNT}"
  echo "Newly built: ${SUCCESS_COUNT}"
  echo "Using original digests: $((FINAL_COUNT - SUCCESS_COUNT))"
  echo ""

  # Build oc adm release new command
  echo "Generating release command..."

  RELEASE_TAG="custom-${SESSION_ID}"
  OUTPUT_RELEASE="${registry}/okd-release:${RELEASE_TAG}"

  # Start building command
  RELEASE_CMD="oc adm release new"
  RELEASE_CMD="$RELEASE_CMD --from-release=${base_release}"

  # Check if cluster-version-operator was rebuilt
  # IMPORTANT: --to-image-base is ONLY required when CVO is being overwritten
  CVO_IMAGE=$(jq -r '.["cluster-version-operator"] // empty' "${WORK_DIR}/image_replacements.json")

  if [ -n "$CVO_IMAGE" ]; then
    echo "Note: cluster-version-operator was rebuilt"
    echo "Using --to-image-base flag (required when overwriting CVO)"
    RELEASE_CMD="$RELEASE_CMD --to-image-base=${CVO_IMAGE}"
  fi

  # Add all component mappings
  while IFS= read -r line; do
    component=$(echo "$line" | jq -r '.key')
    image=$(echo "$line" | jq -r '.value')
    RELEASE_CMD="$RELEASE_CMD ${component}=${image}"
  done < <(jq -r 'to_entries[] | @json' "${WORK_DIR}/final_mappings.json")

  # Add output and flags
  RELEASE_CMD="$RELEASE_CMD --to-image=${OUTPUT_RELEASE}"
  RELEASE_CMD="$RELEASE_CMD --keep-manifest-list"
  RELEASE_CMD="$RELEASE_CMD --allow-missing-images"

  # Save command to file for reference
  echo "$RELEASE_CMD" > "${WORK_DIR}/release_command.txt"

  echo "Release command saved to: ${WORK_DIR}/release_command.txt"
  echo ""

  # Display command preview (formatted for readability)
  echo "Command preview:"
  echo "  oc adm release new \\"
  echo "    --from-release=${base_release} \\"
  [ -n "$CVO_IMAGE" ] && echo "    --to-image-base=${CVO_IMAGE} \\"
  echo "    [${FINAL_COUNT} component mappings] \\"
  echo "    --to-image=${OUTPUT_RELEASE} \\"
  echo "    --keep-manifest-list \\"
  echo "    --allow-missing-images"
  echo ""

  # Ask for confirmation
  echo "Execute release creation now? [y/N]"
  read -r response

  if [[ "$response" =~ ^[Yy]$ ]]; then
    echo "Creating release..."
    echo ""

    RELEASE_LOG="${WORK_DIR}/release_creation.log"

    if eval "$RELEASE_CMD" > "$RELEASE_LOG" 2>&1; then
      echo "✓ Release created successfully"
      echo ""
      echo "========================================"
      echo "Release Image Created"
      echo "========================================"
      echo ""
      echo "Release image: ${OUTPUT_RELEASE}"
      echo ""
      echo "To use this release:"
      echo "  oc adm upgrade --to-image=${OUTPUT_RELEASE}"
      echo ""
    else
      echo "❌ Release creation failed"
      echo "See log: ${RELEASE_LOG}"
      echo ""
      echo "You can retry manually using:"
      echo "  bash -c \"\$(cat ${WORK_DIR}/release_command.txt)\""
      echo ""
      exit 1
    fi
  else
    echo "Skipped release creation"
    echo ""
    echo "To create release manually:"
    echo "  bash -c \"\$(cat ${WORK_DIR}/release_command.txt)\""
    echo ""
  fi
fi
```

---

### Phase 8: Final Summary & Output

```bash
echo ""
echo "========================================"
echo "✓ Workflow Complete"
echo "========================================"
echo ""
echo "Summary:"
echo "  Total components in builds.json: ${TOTAL_COMPONENTS}"

if [ "$bash_mode" = "true" ]; then
  echo "  Buildable components: ${BUILDABLE_COUNT}"
  echo "  Unbuildable components: ${UNBUILDABLE_COUNT}"
  echo ""
  echo "Build script: ${SCRIPT_PATH}"
  echo ""
  echo "To execute:"
  echo "  ${SCRIPT_PATH}"
  echo ""
else
  SUCCESS_COUNT=$(jq 'length' "${WORK_DIR}/image_replacements.json")
  echo "  Successfully built: ${SUCCESS_COUNT}"
  echo "  Build failures: ${BUILD_FAILED}"
  echo "  Unbuildable (no source/Dockerfile): ${UNBUILDABLE_COUNT}"
  echo "  Using original digests: $((TOTAL_COMPONENTS - SUCCESS_COUNT))"
  echo ""
  echo "Release image: ${OUTPUT_RELEASE}"
  echo ""
  echo "Pull command:"
  echo "  oc adm upgrade --to-image=${OUTPUT_RELEASE}"
  echo ""
fi

echo "Working directory: ${WORK_DIR}"
echo ""
echo "Key files:"
echo "  - Buildable components: ${WORK_DIR}/buildable_components.txt"
echo "  - Unbuildable components: ${WORK_DIR}/unbuildable_components.txt"
echo "  - Build results: ${WORK_DIR}/build_results.json"
echo "  - Image replacements: ${WORK_DIR}/image_replacements.json"
if [ "$bash_mode" != "true" ]; then
  echo "  - Final mappings: ${WORK_DIR}/final_mappings.json"
  echo "  - Release command: ${WORK_DIR}/release_command.txt"
  echo "  - Release log: ${WORK_DIR}/release_creation.log"
fi
echo ""
```

## Return Value

- **Format**: Summary report with build status and release information, or bash script location

The command outputs depend on the execution mode:

### Normal Execution Mode (without --bash)

1. **Discovery Summary**:
   - Metadata extraction results
   - Repository cloning status
   - Dockerfile discovery results

2. **Build Plan Review**:
   - Buildable components count and list
   - Unbuildable components count with breakdown by reason
   - User confirmation prompt

3. **Build Results**:
   - Status for each component (Success/Failed)
   - Image references with digests for successful builds
   - Error logs for failed builds

4. **Release Information**:
   - Complete `oc adm release new` command
   - Target release image reference
   - Usage instructions

**Example output:**
```
========================================
✓ Workflow Complete
========================================

Summary:
  Total components in builds.json: 193
  Successfully built: 145
  Build failures: 12
  Unbuildable (no source/Dockerfile): 36
  Using original digests: 48

Release image: quay.io/myuser/okd-release:custom-20250122-143022

Pull command:
  oc adm upgrade --to-image=quay.io/myuser/okd-release:custom-20250122-143022

Working directory: .work/okd-build/20250122-143022

Key files:
  - Buildable components: .work/okd-build/20250122-143022/buildable_components.txt
  - Unbuildable components: .work/okd-build/20250122-143022/unbuildable_components.txt
  - Build results: .work/okd-build/20250122-143022/build_results.json
  - Image replacements: .work/okd-build/20250122-143022/image_replacements.json
  - Final mappings: .work/okd-build/20250122-143022/final_mappings.json
  - Release command: .work/okd-build/20250122-143022/release_command.txt
  - Release log: .work/okd-build/20250122-143022/release_creation.log
```

### Bash Script Generation Mode (with --bash)

1. **Discovery Summary**:
   - Metadata extraction results
   - Repository cloning status
   - Dockerfile discovery results

2. **Build Plan Review**:
   - Buildable components count and list
   - Unbuildable components count with breakdown by reason
   - User confirmation prompt

3. **Script Location**:
   - Path to generated script: `build-okd-release-<session-id>.sh`
   - Execution instructions
   - Script overview

**Example output:**
```
========================================
✓ Workflow Complete
========================================

Summary:
  Total components in builds.json: 193
  Buildable components: 157
  Unbuildable components: 36

Build script: ./build-okd-release-20250122-143022.sh

To execute:
  ./build-okd-release-20250122-143022.sh

The script will:
  1. Build 157 components
  2. Push images to quay.io/myuser
  3. Create release image using quay.io/okd/scos-release:4.21.0-okd-scos.ec.13

Working directory: .work/okd-build/20250122-143022

Key files:
  - Buildable components: .work/okd-build/20250122-143022/buildable_components.txt
  - Unbuildable components: .work/okd-build/20250122-143022/unbuildable_components.txt
  - Build results: .work/okd-build/20250122-143022/build_results.json
  - Image replacements: .work/okd-build/20250122-143022/image_replacements.json
```

## Examples

1. **Build release from builds.json with default settings**:
   ```
   /okd-build:build-new-release builds.json
   ```
   Uses default registry (`quay.io/${USER}`), auto-fetches latest SCOS release, and executes builds directly.

2. **Build and push to custom registry**:
   ```
   /okd-build:build-new-release builds.json --registry=quay.io/myteam
   ```
   Builds operators and pushes to specified registry.

3. **Build with custom base release**:
   ```
   /okd-build:build-new-release builds.json --base-release=quay.io/okd/scos-release:4.22.0-okd-scos.ec.1
   ```
   Uses a specific OKD release as the base for the custom release payload.

4. **Generate bash script instead of executing**:
   ```
   /okd-build:build-new-release builds.json --bash
   ```
   Creates `build-okd-release-<timestamp>.sh` script for manual review and execution.

5. **Full customization**:
   ```
   /okd-build:build-new-release builds.json --registry=quay.io/myteam --base-release=quay.io/okd/scos-release:4.22.0-okd-scos.ec.1 --bash
   ```
   Generates a customized build script with specified registry and base release.

## Arguments

- `<builds-json-path>` (Required): String. Path to the builds.json file containing component-to-digest mappings. The file must be valid JSON with format: `{"component-name": "registry/namespace/repo@sha256:digest"}`. Example: `{"cluster-version-operator": "quay.io/redhat-user-workloads/...@sha256:..."}`

- `--registry=<registry>` (Optional): String. Target registry for pushing built images. Format: `registry.example.com/namespace`. Default: `quay.io/${USER}` where `${USER}` is the current system user. Requires authentication to the specified registry before execution.

- `--base-release=<release-image>` (Optional): String. Base OKD release image to use for creating the custom release payload. This should be a fully qualified image reference. If not provided, the latest SCOS release is automatically fetched from the OKD release stream API (`https://amd64.origin.releases.ci.openshift.org/api/v1/releasestreams/accepted`) by querying the first element of the `4-scos-next` array. Examples:
  - `quay.io/okd/scos-release:4.21.0-okd-scos.ec.13` (auto-fetched if not specified)
  - `quay.io/okd/scos-release:4.22.0-okd-scos.ec.1`
  - Custom release images from your registry

- `--bash` (Optional): Boolean flag. When present, generates a bash script (`build-okd-release-<session-id>.sh`) instead of executing builds directly. The script will include all build commands, image tagging/pushing, digest extraction using `skopeo inspect`, and the final `oc adm release new` command. This allows for manual review and customization before execution.

## Prerequisites

1. **skopeo**: Container image inspection tool
   - Check if installed: `which skopeo`
   - Installation: https://github.com/containers/skopeo/blob/main/install.md
   - Required for extracting VCS metadata from image labels and getting image digests

2. **git**: Version control system
   - Check if installed: `which git`
   - Installation: https://git-scm.com/downloads
   - Required for cloning operator source repositories

3. **jq**: JSON processor
   - Check if installed: `which jq`
   - Installation: https://stedolan.github.io/jq/download/
   - Required for parsing builds.json, image metadata, and tracking files

4. **curl**: HTTP client
   - Check if installed: `which curl`
   - Installation: Usually pre-installed on most systems
   - Required for fetching latest SCOS release from OKD API when `--base-release` is not specified

5. **Podman or Docker**: Container build tool
   - Check if installed: `which podman` or `which docker`
   - Podman installation: https://podman.io/getting-started/installation
   - Docker installation: https://docs.docker.com/get-docker/
   - Required for building and pushing container images (not needed in `--bash` mode)

6. **oc CLI** (for release orchestration): OpenShift CLI
   - Check if installed: `which oc`
   - Installation: https://docs.openshift.com/container-platform/latest/cli_reference/openshift_cli/getting-started-cli.html
   - Required for creating release payloads (not needed in `--bash` mode)

7. **Registry Authentication**: Credentials for target registry
   - Login: `podman login quay.io` (or `docker login quay.io`)
   - Required for pushing built images to registry

## Notes

- **Input file format**: The builds.json file must be a flat JSON object mapping component names to image digests. Each value must be a fully-qualified image reference with digest (format: `registry/namespace/repo@sha256:digest`).

- **Working directory**: All operations create a timestamped working directory at `.work/okd-build/<session-id>/` containing cloned repositories, build logs, tracking files, and metadata. This directory is preserved after execution for review and debugging. Manual cleanup is recommended after successful release creation.

- **Disk space requirements**: Processing ~190 components requires approximately 95GB of disk space for cloned repositories (average ~500MB per operator). Ensure sufficient disk space is available before execution.

- **Time estimates**:
  - Metadata extraction + cloning: 30-60 minutes
  - Building: Highly variable (2-5 minutes per operator average)
  - Total for 190 components: Several hours for complete execution

- **Unbuildable components**: Components that cannot be built (no VCS URL, clone failures, missing Dockerfiles) will NOT block release creation. The final release will use original digests from builds.json for these components. A detailed report is provided before execution begins.

- **SCOS transformation specifics**: The command ONLY transforms base image lines matching the pattern `FROM registry.ci.openshift.org/ocp/X.XX:base-rhel[89]`. Builder images (e.g., golang builders) and multi-stage build images are NOT modified. This ensures builds remain functional while migrating to SCOS base images.

- **cluster-version-operator special case**: When `cluster-version-operator` IS being overwritten (included in built components), the `oc adm release new` command **MUST include** the `--to-image-base` flag pointing to the cluster-version-operator image with digest. When `cluster-version-operator` IS NOT being overwritten, the command should omit the `--to-image-base` flag entirely. This is required for proper release creation.

- **Error handling philosophy**: The command uses graceful degradation with multiple user checkpoints. Failed builds do not stop the overall process - the command continues with remaining operators. The `--allow-missing-images` flag in the release command permits partial operator updates.

- **State preservation**: All tracking JSON files (`metadata_extraction.json`, `buildable.json`, `unbuildable.json`, `build_results.json`, `image_replacements.json`) allow manual intervention. If execution is interrupted or aborted, the working directory is preserved for review and manual recovery.

- **Bash script mode benefits**: Using `--bash` flag generates a complete, executable script that can be:
  - Reviewed and customized before execution
  - Version-controlled for reproducibility
  - Executed multiple times with same configuration
  - Modified to skip specific components or adjust build parameters

- **Parallel execution**: In v1, builds are executed sequentially to ensure reliability and resource management. Future versions may add `--parallel` flag for concurrent builds.

- **Security considerations**:
  - VCS URLs are extracted from image metadata - verify source authenticity
  - Private repositories require SSH key or credential configuration
  - Registry push operations require proper authentication
  - Review Dockerfile transformations using diff files before execution
