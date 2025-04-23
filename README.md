#!/usr/bin/env bash
set -euxo pipefail

LOG_FILE="error.log"
: >"$LOG_FILE"
exec 2>>"$LOG_FILE"
exec > >(tee -a "$LOG_FILE")

# Validate GitLab token
if [[ -z "${TF_VAR_gitlab_token:-}" && -z "${GITLAB_TOKEN:-}" ]]; then
  echo "Error: GitLab token not set." >&2
  exit 1
fi

# MODE toggle: 'full' or 'incremental'
MODE="${MODE:-incremental}"

GITLAB_TOKEN="${TF_VAR_gitlab_token:-$GITLAB_TOKEN}"
GITLAB_URL="${GITLAB_URL:-https://gitlab.mda.mil}"
GROUP_ID="${GROUP_ID:-3482}"

CWD=$(pwd)
BUNDLE_DIR="${BUNDLE_DIR:-$CWD/gitlab_repos_bundles}"
LOCAL_REPO_BASE="${LOCAL_REPO_BASE:-$CWD/local_gitlab_repos}"
BUNDLE_BRANCHES="${BUNDLE_BRANCHES:-main staging}"
TAG_SUFFIX="transfer"

IFS=' ' read -r -a BUNDLE_BRANCHES_ARRAY <<< "$BUNDLE_BRANCHES"
mkdir -p "$BUNDLE_DIR" "$LOCAL_REPO_BASE"

get_projects_in_group() {
  curl --silent --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
    "$GITLAB_URL/api/v4/groups/$GROUP_ID/projects?include_subgroups=true&per_page=100" |
    jq -r '.[] | "\(.id);\(.path_with_namespace)"'
}

get_repo_https_url() {
  local repo_id=$1
  curl --silent --header "PRIVATE-TOKEN: $GITLAB_TOKEN" "$GITLAB_URL/api/v4/projects/$repo_id" |
    jq -r '.http_url_to_repo'
}

get_latest_commit() {
  git rev-parse HEAD
}

get_root_commit() {
  git rev-list --max-parents=0 HEAD | tail -n 1
}

get_last_commit_file() {
  local sanitized=$1
  local branch=$2
  echo "$BUNDLE_DIR/last_commit__${sanitized}__${branch}"
}

create_bundle() {
  local repo_path=$1
  local branch=$2
  local bundle_file=$3
  local sanitized=$4

  echo "[MODE: $MODE] Processing $sanitized on branch $branch"

  cd "$repo_path"
  git checkout "$branch" || git checkout -b "$branch" "origin/$branch"
  git pull origin "$branch"

  local latest_commit
  latest_commit=$(get_latest_commit)

  local last_commit_file
  last_commit_file=$(get_last_commit_file "$sanitized" "$branch")

  local last_commit=""
  if [[ -f "$last_commit_file" ]]; then
    last_commit=$(cat "$last_commit_file")
    if ! git cat-file -e "$last_commit" 2>/dev/null; then
      last_commit=$(get_root_commit)
    fi
  else
    last_commit=$(get_root_commit)
  fi

  if [[ "$MODE" == "incremental" ]]; then
    if [[ "$latest_commit" == "$last_commit" ]]; then
      echo "[SKIPPED] No new commits for $sanitized on $branch"
      cd - >/dev/null
      return 0
    fi

    local tag="v$(date +%Y%m%d%H%M%S)-$TAG_SUFFIX"
    echo "[TAG] Creating transfer tag: $tag at $latest_commit"
    git tag -a "$tag" "$latest_commit" -m "Transfer tag $tag"
    git bundle create "$bundle_file" "$last_commit..$latest_commit"
    echo "$latest_commit" > "$last_commit_file"
    echo "[BUNDLE] Incremental bundle created at $bundle_file"
  else
    echo "[BUNDLE] Creating full bundle for $sanitized on $branch"
    git bundle create "$bundle_file" --all
    echo "$latest_commit" > "$last_commit_file"
  fi

  cd - >/dev/null
}

process_repo() {
  local repo_id=$1
  local repo_namespace=$2

  repo_url=$(get_repo_https_url "$repo_id")
  sanitized_name=$(echo "$repo_namespace" | sed 's|/|__|g')
  local_path="$LOCAL_REPO_BASE/$sanitized_name"

  if [[ -d "$local_path/.git" ]]; then
    git -C "$local_path" fetch --all --tags
  else
    git clone "$repo_url" "$local_path"
  fi

  for branch in "${BUNDLE_BRANCHES_ARRAY[@]}"; do
    if git -C "$local_path" show-ref --verify --quiet "refs/remotes/origin/$branch"; then
      git -C "$local_path" checkout "$branch" || git -C "$local_path" checkout -b "$branch" "origin/$branch"
      bundle_path="$BUNDLE_DIR/${sanitized_name}__${branch}.bundle"
      create_bundle "$local_path" "$branch" "$bundle_path" "$sanitized_name"
    else
      echo "[WARNING] Branch $branch not found in $repo_namespace"
    fi
  done
}

main() {
  REPOSITORIES=$(get_projects_in_group)
  if [[ -z "$REPOSITORIES" ]]; then
    echo "No repositories found."
    exit 1
  fi

  echo "$REPOSITORIES" | while IFS=';' read -r repo_id repo_namespace; do
    [[ -z "$repo_id" ]] && continue
    process_repo "$repo_id" "$repo_namespace"
  done

  echo "------- Summary of Bundles --------"
  if ls -1 "$BUNDLE_DIR"/*.bundle 2>/dev/null; then
    echo "Bundles created:"
    ls -l "$BUNDLE_DIR"/*.bundle
  else
    echo "No bundles were created."
  fi
}

main





-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
#!/usr/bin/bash

LOG_FILE="error.log"

# Truncate the log file at the start of the script
true > "$LOG_FILE"

# Redirect stderr for the entire script
exec 2>> "$LOG_FILE"

# Redirect stdout to both the terminal and log file
exec > >(tee -a "$LOG_FILE")


# Check if the GitLab access token is set in the environment, exit if not
if [ -z "$GITLAB_TOKEN" ]; then
  echo "Error: GITLAB_TOKEN environment variable is not set." >&2
  echo "Please set your GitLab token using: export GITLAB_TOKEN='your_personal_access_token'" >&2
  exit 1
fi

# Set the current working directory
CWD=$(pwd)

#GITLAB_TOKEN="$TF_VAR_gitlab_token"
#GITLAB_TOKEN="d9z9AcayqfEsGe5sv9RG"

# Base GitLab URL (destination GitLab, airgapped environment)
DEST_GITLAB_URL="${DEST_GITLAB_URL:-"https://gitlab.mda.mil"}"

# Directory where the bundles are located
BUNDLE_DIR="${BUNDLE_DIR:-"${CWD}/gitlab_repos_bundles"}"

# Directory where local repositories will be stored
LOCAL_REPO_BASE="${LOCAL_REPO_BASE:-"${CWD}/local_gitlab_repos"}"

# Get today's date for branch prefix (YYYYMMDD)
TODAY=$(date +%Y%m%d)
BRANCH_PREFIX="transfer-${TODAY}/"

# Create the directory for local repositories
mkdir -p "$LOCAL_REPO_BASE"

# Declare an associative array to hold ignored files for repository groups
#declare -A ignored_files
#declare -A ignored_subgroups

# Example: Set ignored files/patterns for individual or multiple repositories
#ignored_files["repo1"]="*.log *.tmp"
#ignored_files["repo2,repo5,repo8"]="README.md config.json *.bak"
#ignored_files["repo3,repo12"]="docs/*"

# Common ignored files for all repositories
#common_ignored="*.swp .gitignore"

# Example: Set ignored files/patterns for subgroups by subgroup ID or namespace
#ignored_subgroups["123324"]="**/providers.tf"
#ignored_subgroups["my-top-level-group/infrastructure-group/terraform-subgroup"]="**/providers.tf"

# Function to get the ignored files for a repository or subgroup
get_ignored_files() {
    local repo_name=$1
    local repo_namespace=$2
    local subgroup_id=$3

    # Check if there are any specific ignored files for the repository
    for repo_group in "${!ignored_files[@]}"; do
        if [[ ",${repo_group}," == *",$repo_name,"* ]]; then
            echo "${ignored_files[$repo_group]} $common_ignored"
            return
        fi
    done

    # Check if there are any specific ignored files for the subgroup by ID or namespace
    if [ -n "${ignored_subgroups[$subgroup_id]}" ]; then
        echo "${ignored_subgroups[$subgroup_id]} $common_ignored"
        return
    fi
    if [ -n "${ignored_subgroups[$repo_namespace]}" ]; then
        echo "${ignored_subgroups[$repo_namespace]} $common_ignored"
        return
    fi

    # Return only common ignored files if no specific exclusions were found
    echo "$common_ignored"
}

# Function to URL-encode a string
urlencode() {
    local string="$1"
    local strlen=${#string}
    local encoded=""

    for (( pos=0 ; pos<strlen ; pos++ )); do
        c=${string:$pos:1}
        case "$c" in
            [a-zA-Z0-9.~_-]) encoded+="$c" ;;
            *) encoded+=$(printf '%%%02X' "'$c") ;;
        esac
    done
    echo "$encoded"
}

# Function to create a branch in the destination GitLab if it doesn't exist
create_branch_if_missing() {
    local project_id=$1
    local branch_name=$2
    local ref=$3  # Reference (usually 'main' or 'master')

    # Check if the branch exists in the destination GitLab
    branch_exists=$(curl --silent --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
        "$DEST_GITLAB_URL/api/v4/projects/$project_id/repository/branches/$(urlencode "$branch_name")" | jq -r '.name')

    # If the branch does not exist, create it
    if [ "$branch_exists" == "null" ]; then
        echo "Branch $branch_name does not exist. Creating it from $ref..." >&2
        curl --silent --request POST --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
            "$DEST_GITLAB_URL/api/v4/projects/$project_id/repository/branches" \
            --data-urlencode "branch=$branch_name" \
            --data-urlencode "ref=$ref" > /dev/null
        echo "Branch $branch_name created." >&2
    else
        echo "Branch $branch_name already exists." >&2
    fi
}

# Function to submit a merge request
submit_merge_request() {
    local project_id=$1
    local source_branch=$2
    local target_branch=$3
    local repo_name=$4

    echo "Submitting merge request from $source_branch to $target_branch for project $repo_name..." >&2

    # Submit the merge request
    curl --silent --request POST --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
        "$DEST_GITLAB_URL/api/v4/projects/$project_id/merge_requests" \
        --data-urlencode "source_branch=$source_branch" \
        --data-urlencode "target_branch=$target_branch" \
        --data-urlencode "title=Merge from $source_branch into $target_branch" \
        --data-urlencode "description=Automated merge request for repository $repo_name" \
        --data "remove_source_branch=true" > /dev/null

    echo "Merge request submitted from $source_branch to $target_branch." >&2
}

# Function to get the namespace ID on the destination GitLab
get_namespace_id() {
    local namespace_path=$1

    # If the namespace is empty or root, return null
    if [ -z "$namespace_path" ] || [ "$namespace_path" == "." ]; then
        echo "null"
        return
    fi

    namespace=$(curl --silent --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
        "$DEST_GITLAB_URL/api/v4/namespaces?search=$(urlencode "$namespace_path")" | jq -r '.[] | select(.full_path=="'"$namespace_path"'")')

    namespace_id=$(echo "$namespace" | jq -r '.id')

    if [ -z "$namespace_id" ] || [ "$namespace_id" == "null" ]; then
        echo "Namespace $namespace_path does not exist. Creating it..." >&2

        # Create the namespace (group)
        namespace=$(curl --silent --request POST --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
            "$DEST_GITLAB_URL/api/v4/groups" \
            --data-urlencode "name=$(basename "$namespace_path")" \
            --data-urlencode "path=$(basename "$namespace_path")" \
            --data-urlencode "parent_id=$(get_namespace_id "$(dirname "$namespace_path")")" \
            --data "visibility=private")

        namespace_id=$(echo "$namespace" | jq -r '.id')

        if [ -z "$namespace_id" ] || [ "$namespace_id" == "null" ]; then
            echo "Error: Could not create namespace $namespace_path." >&2
            echo "Check your permissions." >&2
            exit 1
        fi
    fi

    echo "$namespace_id"
}

# Function to push repository branches to "transfer-YYYYMMDD/<branch>" on the destination GitLab
push_to_transfer_branch() {
    local repo_namespace=$1
    local branch=$2

    echo "Processing repository: $repo_namespace" >&2

    # Replace slashes in repo namespace with double underscores
    local sanitized_repo_namespace=$(echo "$repo_namespace" | sed 's/\//__/g')

    # Define local repository path
    local local_repo_path="$LOCAL_REPO_BASE/$sanitized_repo_namespace"

    # Change to the repository directory
    cd "$local_repo_path" || { echo "Repository $repo_namespace not found locally." >&2; return; }

    # Fetch subgroup ID (assuming it's stored in a metadata file or via API)
    subgroup_id=$(curl --silent --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
        "$DEST_GITLAB_URL/api/v4/projects/$(urlencode "$repo_namespace")" | jq -r '.namespace.id')

    # Get the list of ignored files/patterns for the current repository or subgroup
    ignored_patterns=$(get_ignored_files "$repo_namespace" "$repo_namespace" "$subgroup_id")

    echo "Ignoring the following files in $repo_namespace: $ignored_patterns" >&2

    # Mark the ignored files as skipped for staging
    for pattern in $ignored_patterns; do
        git update-index --assume-unchanged "$pattern" 2>/dev/null
    done

    # Set the destination GitLab repository URL
    dest_repo_url="$DEST_GITLAB_URL/$repo_namespace.git"

    # Ensure the repository exists on the destination GitLab
    project=$(curl --silent --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
        "$DEST_GITLAB_URL/api/v4/projects/$(urlencode "$repo_namespace")")
    project_id=$(echo "$project" | jq -r '.id')

    if [ "$project_id" == "null" ] || [ -z "$project_id" ]; then
        echo "Project $repo_namespace does not exist on the destination GitLab. Creating it..." >&2

        # Create the project (assuming you have permissions to do so)
        project=$(curl --silent --request POST --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
            "$DEST_GITLAB_URL/api/v4/projects" \
            --data-urlencode "name=$(basename "$repo_namespace")" \
            --data-urlencode "path=$(basename "$repo_namespace")" \
            --data-urlencode "namespace_id=$(get_namespace_id "$(dirname "$repo_namespace")")" \
            --data "visibility=private")
        project_id=$(echo "$project" | jq -r '.id')

        if [ "$project_id" == "null" ] || [ -z "$project_id" ]; then
            echo "Error: Could not create project $repo_namespace on the destination GitLab." >&2
            return
        fi

        echo "Project $repo_namespace created with ID $project_id." >&2
    else
        echo "Project $repo_namespace exists with ID $project_id." >&2
    fi

    # Set the remote URL to the destination GitLab
    git remote rm origin >/dev/null 2>&1
    git remote add origin "$dest_repo_url"

    # Push the branch to the destination GitLab under the transfer prefix
    transfer_branch="${BRANCH_PREFIX}${branch}"
    echo "Pushing branch $branch to $transfer_branch on destination GitLab..." >&2
    git push origin "$branch:$transfer_branch"

    # Create the target branch in the destination if it doesn't exist
    create_branch_if_missing "$project_id" "$branch" "$transfer_branch"

    # Submit the merge request from transfer-YYYYMMDD/<branch> to the original <branch>
    submit_merge_request "$project_id" "$transfer_branch" "$branch" "$repo_namespace"

    # Return to the main directory
    cd - >/dev/null || exit 1
}

# Loop through all .bundle files and process each repository
for bundle_file in "$BUNDLE_DIR"/*.bundle; do
    # Extract repository namespace and branch from the bundle filename
    filename=$(basename "$bundle_file")
    repo_info=$(echo "$filename" | sed 's/__/\//g' | sed 's/\.bundle$//')
    repo_namespace=$(echo "$repo_info" | cut -d'__' -f1)
    branch=$(echo "$repo_info" | cut -d'__' -f2)

    # Replace double underscores with slashes to get the original namespace, Replace any leading slashes, Replace any multiple slashes with single slash, Replace any trailing slashes
    repo_namespace=$(echo "$repo_namespace" | sed -e 's/__/\/\//g' -e 's#^/##' -e 's#//*#/#g' -e 's#/$##')

    # Define local repository path
    sanitized_repo_namespace=$(echo "$repo_namespace" | sed 's/\//__/g')
    local_repo_path="$LOCAL_REPO_BASE/$sanitized_repo_namespace"

    # Initialize or update the repository
    if [ -d "$local_repo_path/.git" ]; then
        echo "Repository $repo_namespace exists locally. Fetching bundle..." >&2
        cd "$local_repo_path" || exit 1
        if ! git fetch "$bundle_file"; then
            echo "Error: Failed to fetch bundle for $repo_namespace" >&2
            continue
        fi
    else
        echo "Initializing repository $repo_namespace..." >&2
        mkdir -p "$local_repo_path"
        cd "$local_repo_path" || exit 1
        git init
        if ! git fetch "$bundle_file"; then
            echo "Error: Failed to fetch bundle for $repo_namespace" >&2
            continue
        fi
    fi

    # Update the branch
    if ! git checkout -B "$branch" "FETCH_HEAD"; then
        echo "Error: Failed to update branch $branch for $repo_namespace" >&2
        cd - >/dev/null || exit 1
        continue
    fi

    # Push to the destination GitLab and submit merge request
    push_to_transfer_branch "$repo_namespace" "$branch"

    # Return to the main directory
    cd - >/dev/null || exit 1
done

echo "All repositories have been processed and pushed to the destination GitLab." >&2
