Test and improve gitlab transfer script


#!/usr/bin/bash

LOG_FILE="error.log"

# Truncate the log file at the start of the script
true >"$LOG_FILE"

# Redirect stderr for the entire script
exec 2>>"$LOG_FILE"

# Redirect stdout to both the terminal and log file
exec > >(tee -a "$LOG_FILE")

# Check if the GitLab access token is set in the environment, exit if not
#if [ -z "$TF_VAR_gitlab_token" ] && [ -z "$GITLAB_TOKEN" ] ; then
if [ -z "$TF_VAR_gitlab_token" ]; then
  echo "Error: GITLAB_TOKEN environment variable is not set."
  echo "Please set your GitLab token using: export GITLAB_TOKEN='your_personal_access_token'"
  exit 1
fi

# Set the current working directory
CWD=$(pwd)

GITLAB_TOKEN="$TF_VAR_gitlab_token"
# Base GitLab URL (replace with your GitLab instance if using self-hosted GitLab, or override via environment)
GITLAB_URL="${GITLAB_URL:-"https://gitlab.mda.mil"}"

# Group ID or Group Path (can be the group's path like 'your_group' or numerical ID, override via environment)
GROUP_ID="${GROUP_ID:-"3482"}"

# Directory where the repositories will be bundled (override via environment)
BUNDLE_DIR="${BUNDLE_DIR:-"${CWD}/gitlab_repos_bundles"}"

# Directory where local repositories will be stored
LOCAL_REPO_BASE="${LOCAL_REPO_BASE:-"${CWD}/local_gitlab_repos"}"

# Use a fixed suffix for transfer tags (override via environment)
TAG_SUFFIX="${TAG_SUFFIX:-transfer}"

# List of subgroups to skip (override via environment). Example: "subgroup_id_1,subgroup_id_2"
DEFAULT_SKIP_SUBGROUPS="3483,4517"
SKIP_SUBGROUPS="${SKIP_SUBGROUPS:-$DEFAULT_SKIP_SUBGROUPS}"

# Branches to track (override via environment). Example: "main staging"
DEFAULT_BUNDLE_BRANCHES="main staging"
BUNDLE_BRANCHES="${BUNDLE_BRANCHES:-$DEFAULT_BUNDLE_BRANCHES}"

# Convert space-separated BUNDLE_BRANCHES into an array
IFS=' ' read -r -a BUNDLE_BRANCHES_ARRAY <<<"$BUNDLE_BRANCHES"

# Convert comma-separated SKIP_SUBGROUPS into an array
IFS=',' read -r -a SKIP_SUBGROUPS_ARRAY <<<"$SKIP_SUBGROUPS"

# Create directories to store the bundled repos and local repos
mkdir -p "$BUNDLE_DIR"
mkdir -p "$LOCAL_REPO_BASE"

# Function to check if a subgroup should be skipped
should_skip_subgroup() {
    local subgroup_id=$1
    for skip_id in "${SKIP_SUBGROUPS_ARRAY[@]}"; do
        if [ "$skip_id" == "$subgroup_id" ]; then
            return 0 # Skip this subgroup
        fi
    done
    return 1 # Do not skip
}

# Function to verify a single bundle and list its contents
verify_bundle() {
    local bundle_path=$1

    if git bundle verify "$bundle_path" >/dev/null 2>&1; then
        echo "Bundle $bundle_path is valid."

        # List the heads (branches, tags) in the bundle
        echo "Listing heads (branches and tags) for bundle $bundle_path:"
        git bundle list-heads "$bundle_path"
    else
        echo "Error: Bundle $bundle_path is invalid." >&2
        return 1
    fi
}

# Function to verify all bundles in the BUNDLE_DIR
verify_all_bundles() {
    echo "Verifying all bundles in $BUNDLE_DIR..." >&2
    for bundle_file in "$BUNDLE_DIR"/*.bundle; do
        if [ -f "$bundle_file" ]; then
            echo "Bundle path found for $repo_namespace at $bundle_file" >&2
            verify_bundle "$bundle_file"
        fi
    done
}

# Function to get repository HTTPS URL via GitLab API
get_repo_https_url() {
    local repo_id=$1
    local response
    local url

    # Check if the repo_id is provided
    if [ -z "$repo_id" ]; then
        echo "Error: repo_id not provided to get_repo_https_url" >&2
        return 1
    fi
    echo "Fetching HTTPS URL for repo_id $repo_id" >&2

    # Fetch the repo data from GitLab API
    response=$(curl --silent --header "PRIVATE-TOKEN: $GITLAB_TOKEN" "$GITLAB_URL/api/v4/projects/$repo_id")
    #echo "$response" > api_response.log

    # Debugging: Print the raw API response
    echo "Raw API Response: $response" >&2

    # Extract the HTTPS URL from the response using jq
    url=$(echo "$response" | jq -r '.http_url_to_repo' | tr -d '\n')

    # Debugging: Print the extracted URL
    echo "Extracted URL: $url" >&2

    # Ensure the URL is valid before returning it
    if [[ -z "$url" ]]; then
        echo "Error: No valid URL found in the response for repo_id $repo_id." >&2
        return 1
    elif [[ ! "$url" =~ ^https:// ]]; then
        echo "Error: Invalid HTTPS URL for repo_id $repo_id: $url" >&2
        return 1
    fi

    # Output the URL to stdout
    echo "$url"
}

# Function to fetch the highest existing tag (SemVer) for a repo
get_highest_version_tag() {
    local repo_path=$1
    local branch=$2

    # Check if the repo_path is provided
    if [ -z "$repo_path" ]; then
        echo "Error: repo_path not provided to get_highest_version_tag" >&2
        return 1
    fi

    cd "$repo_path" || {
        echo "Error: Cannot access directory $repo_path" >&2
        return 1
    }

    # Fetch all tags to ensure local tags are up-to-date with the remote
    git fetch --tags --force

    # Get all main branch tags (without branch suffixes)
    main_tags=$(git tag -l "v[0-9]*.[0-9]*.[0-9]*")

    # Get branch-specific tags (like v0.0.3-staging if branch is staging)
    branch_tags=$(git tag -l "v[0-9]*.[0-9]*.[0-9]*-$branch")

    # If no tags are found, return an empty string
    if [ -z "$main_tags" ] && [ -z "$branch_tags" ]; then
        echo ""
        return 0
    fi

    # Sort the main branch tags and pick the highest one
    highest_main_tag=$(echo "$main_tags" | sort -V | tail -n 1)

    # Sort the branch-specific tags and pick the highest one
    highest_branch_tag=$(echo "$branch_tags" | sort -V | tail -n 1)

    # Return the branch-specific tag if we are on a branch other than main
    if [[ "$branch" != "main" && -n "$highest_branch_tag" ]]; then
        echo "$highest_branch_tag"
    else
        # Return the highest main branch tag if we are on main or if no branch-specific tags exist
        echo "$highest_main_tag"
    fi
}


# Function to increment a SemVer tag (e.g., from v0.0.6 to v0.0.7)
increment_tag() {
    local tag=$1
    # Extract the version numbers
    if [ -z "$tag" ]; then
        # Start from v0.0.0 if no tag exists
        major=0
        minor=0
        patch=0
    else
        # Remove any suffix (e.g., -branch)
        version_numbers=$(echo "$tag" | sed 's/^v//' | cut -d'-' -f1)
        major=$(echo "$version_numbers" | cut -d'.' -f1)
        minor=$(echo "$version_numbers" | cut -d'.' -f2)
        patch=$(echo "$version_numbers" | cut -d'.' -f3)
    fi

    # Increment the patch version
    new_patch=$((patch + 1))

    # Return the new version tag
    echo "v$major.$minor.$new_patch"
}

# Function to create tags and incremental bundle
create_incremental_bundle_and_tags() {
    local repo_id=$1
    local repo_namespace=$2
    local branch=$3
    local repo_https_url=$4
    local local_repo_path=$5  # Pass the local_repo_path as a parameter

    # Create file paths using the sanitized name
    local sanitized_repo_namespace
    sanitized_repo_namespace=$(echo "$repo_namespace" | sed 's/\//__/g')
    local last_commit_file="$BUNDLE_DIR/last_commit__${sanitized_repo_namespace}__${branch}"
    local bundle_path="$BUNDLE_DIR/${sanitized_repo_namespace}__${branch}__incremental.bundle"

    # Ensure we are in the correct directory
    cd "$local_repo_path" || {
        echo "Error: Cannot access directory $local_repo_path" >&2
        return
    }

    # Get the latest commit on the branch
    local latest_commit
    latest_commit=$(git rev-parse "$branch") || {
        echo "Error: Failed to get the latest commit for branch $branch" >&2
        return 1
    }
    echo "Latest commit on branch $branch: $latest_commit" >&2

    # Get the highest version tag from the current branch (including branch-specific tags)
    local highest_tag
    highest_tag=$(get_highest_version_tag "$local_repo_path" "$branch") || {
        echo "Error: Failed to get the highest tag for branch $branch" >&2
        return 1
    }
    echo "Highest tag on branch $branch: $highest_tag" >&2

    # Read the last processed commit from the last_commit_file
    local last_commit=""
    if [ -f "$last_commit_file" ]; then
        last_commit=$(cat "$last_commit_file")
        # Verify that the last commit exists in the current repository
        if ! git cat-file -e "$last_commit" 2>/dev/null; then
            echo "Warning: Last processed commit $last_commit does not exist in the current repository history." >&2
            echo "Falling back to the root commit of the branch $branch." >&2
            last_commit=$(git rev-list --max-parents=0 "$branch")
        fi
    else
        # No last commit, use the root commit of the branch
        last_commit=$(git rev-list --max-parents=0 "$branch")
    fi
    echo "Last processed commit for branch $branch: $last_commit" >&2

    

    # Determine if this is the first time bundling
    local is_first_bundle=false
    if [ ! -f "$last_commit_file" ]; then
        is_first_bundle=true
        echo "First time bundling branch $branch for repository $repo_namespace"
    fi

    # Check if there are new commits to bundle
    if [ -f "$bundle_path" ] && [ -f "$last_commit_file" ] && [ "$last_commit" == "$latest_commit" ]; then
        echo "No new commits to bundle for branch $branch in repository $repo_namespace"
        return 0
    fi

    # Verify if there are valid commits between last_commit and latest_commit
    if [ ] [ ! "git rev-list "$last_commit..$latest_commit" | grep -q ." ]; then
        echo "No valid commits found between $last_commit and $latest_commit for branch $branch" >&2
        return 0
    fi

    # Check if the transfer tag already exists
    local transfer_tag
    if [ -n "$highest_tag" ]; then
        transfer_tag="${highest_tag}-${TAG_SUFFIX}"
        echo "Transfer tag $transfer_tag using highest commit $highest_tag for branch $branch in repository $repo_namespace" >&2
    else
        transfer_tag="${latest_commit}-${TAG_SUFFIX}"
        echo "Transfer tag  $transfer_tag using latest commit $latest_commit for branch $branch in repository $repo_namespace" >&2
    fi

    if git show-ref --tags | grep -q "$transfer_tag"; then
        echo "Tag $transfer_tag already exists. No new commits to bundle for branch $branch in repository $repo_namespace" >&2
        return 0
    fi

    # Create the transfer tag
    echo "Creating transfer tag $transfer_tag for branch $branch in repository $repo_namespace"
    if ! git tag -a "$transfer_tag" "$latest_commit" -m "Transfer tag $transfer_tag on $branch"; then
        echo "Error: Failed to create transfer tag $transfer_tag for branch $branch" >&2
        return 1
    fi


    # Adjust the bundle creation based on whether it's the first bundle
    if [ "$is_first_bundle" = true ]; then
        # Create a full bundle
        echo "Creating full bundle for $repo_namespace at $bundle_path"
        if ! git bundle create "$bundle_path" --all; then
            echo "Error: Failed to create full bundle for $repo_namespace" >&2
            return 1
        fi
    fi
    # else
    #     # Create an incremental bundle
    #     echo "Creating incremental bundle for $repo_namespace at $bundle_path"
    #     if ! git bundle create "$bundle_path" "$last_commit..$latest_commit"; then
    #         echo "Error: Failed to create incremental bundle for $repo_namespace" >&2
    #         return 1
    #     fi
    # fi

    # # Validate the bundle
    # if ! git bundle verify "$bundle_path"; then
    #     echo "Error: Bundle $bundle_path is invalid. Attempting to recreate..." >&2
    #     rm "$bundle_path"
    #     return 1
    # fi

    # Save the latest commit for future bundles
    echo "$latest_commit" > "$last_commit_file"

    echo "Incremental bundle created for $repo_namespace at $bundle_path"
}


# Function to recursively get all repositories (projects) from a group and its subgroups via GitLab API
get_projects_in_group() {
    local group_id=$1
    local page=1
    local per_page=100
    local project_list

    while :; do
        response=$(curl --silent --header "PRIVATE-TOKEN: $GITLAB_TOKEN" "$GITLAB_URL/api/v4/groups/$group_id/projects?per_page=$per_page&page=$page")

        # Debugging statement (if needed):
        # echo "API Response for Group Projects: $response"

        # Extract project IDs and paths
        project_list=$(echo "$response" | jq -r '.[] | "\(.id);\(.path_with_namespace)"')

        # Break the loop if no more projects are returned
        if [ -z "$project_list" ]; then
            break
        fi

        # Output the project list
        echo "$project_list"

        page=$((page + 1)) # Move to the next page
    done

    # Recursively fetch subgroups
    curl --silent --header "PRIVATE-TOKEN: $GITLAB_TOKEN" "$GITLAB_URL/api/v4/groups/$group_id/subgroups?per_page=100" |
        jq -r '.[].id' |
        while read -r subgroup_id; do
            if should_skip_subgroup "$subgroup_id"; then
                continue
            fi
            get_projects_in_group "$subgroup_id"
        done
}

# Capture the projects in the group in the REPOSITORIES variable
REPOSITORIES=$(get_projects_in_group "$GROUP_ID")

# Check if REPOSITORIES is empty
if [ -z "$REPOSITORIES" ]; then
    echo "Error: No repositories found." >&2
    exit 1
fi

# Loop through all repositories and branches to create incremental bundles and handle tags
echo "$REPOSITORIES" | while IFS=';' read -r repo_id repo_namespace; do
    if [ -z "$repo_id" ]; then
        echo "Error: repo_id is empty for repo_namespace: $repo_namespace" >&2
        continue
    fi

    repo_https_url=$(get_repo_https_url "$repo_id")

    # Debugging: Check if repo_https_url is empty
    if [ -z "$repo_https_url" ]; then
        echo "Error: repo_https_url is empty for repo_id: $repo_id" >&2
        continue
    fi

    # Replace slashes in repo namespace with double underscores
    sanitized_repo_namespace=$(echo "$repo_namespace" | sed 's/\//__/g')

    # Define local repository path
    local_repo_path="$LOCAL_REPO_BASE/$sanitized_repo_namespace"

    # Clone or update the repository
    if [ -d "$local_repo_path/.git" ]; then
        echo "Repository $repo_namespace exists locally. Fetching updates..." >&2
        cd "$local_repo_path" || exit 1

        # Ensure the remote URL is set correctly (without token)
        git remote set-url origin "$repo_https_url"

        if ! git fetch --all --tags; then
            echo "Error: Failed to fetch updates for $repo_namespace" >&2
            continue
        fi
    else
        echo "Cloning repository $repo_namespace..."
        if ! git clone -v "$repo_https_url" "$local_repo_path"; then
            echo "Error: Failed to clone repository $repo_namespace" >&2
            continue
        fi
        cd "$local_repo_path" || exit 1
    fi

    # For each branch in BUNDLE_BRANCHES, ensure it exists locally and process it
    for branch in "${BUNDLE_BRANCHES_ARRAY[@]}"; do
        if git show-ref --verify --quiet "refs/heads/$branch"; then
            echo "Local branch $branch already exists." >&2
        else
            # Attempt to create a local branch tracking the remote branch
            if git show-ref --verify --quiet "refs/remotes/origin/$branch"; then
                echo "Creating local branch $branch tracking origin/$branch..."
                if ! git branch --track "$branch" "origin/$branch"; then
                    echo "Error: Failed to create local branch $branch for repository $repo_namespace" >&2
                    continue
                fi
            else
                echo "Branch $branch does not exist in repository $repo_namespace. Skipping..."
                continue
            fi
        fi

        # Check out the branch only if it exists
        if ! git checkout "$branch"; then
            echo "Error: Failed to checkout branch $branch for repository $repo_namespace" >&2
            continue
        fi

        # Now process the branch only if we successfully checked it out
        create_incremental_bundle_and_tags "$repo_id" "$repo_namespace" "$branch" "$repo_https_url" "$local_repo_path"
        bundle_file="$BUNDLE_DIR/${sanitized_repo_namespace}__${branch}__incremental.bundle"
        if [ -f "$bundle_file" ]; then
            echo "Bundle created for $repo_namespace at $bundle_file" >&2
        else
            echo "No bundle created for $repo_namespace at $bundle_file" >&2
        fi
    done
done

# List all bundles before verification
if ls -l "$BUNDLE_DIR"/*.bundle >/dev/null 2>&1; then
    echo "Bundles found in $BUNDLE_DIR:"
    ls -l "$BUNDLE_DIR"/*.bundle
else
    echo "No bundles found in $BUNDLE_DIR"
fi

# Verify all bundles created
verify_all_bundles

echo "All repositories have been processed, incremental bundles with tags created, and verification completed."


+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
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

# Base GitLab URL (destination GitLab, airgapped environment)
DEST_GITLAB_URL="${DEST_GITLAB_URL:-https://destination_gitlab_url}"

# Directory where the bundles are located
BUNDLE_DIR="${BUNDLE_DIR:-./gitlab_repos_bundles}"

# Directory where local repositories will be stored
LOCAL_REPO_BASE="${LOCAL_REPO_BASE:-./imported_gitlab_repos}"

# Get today's date for branch prefix (YYYYMMDD)
TODAY=$(date +%Y%m%d)
BRANCH_PREFIX="transfer-${TODAY}/"

# Create the directory for local repositories
mkdir -p "$LOCAL_REPO_BASE"

# Declare an associative array to hold ignored files for repository groups
declare -A ignored_files
declare -A ignored_subgroups

# Example: Set ignored files/patterns for individual or multiple repositories
ignored_files["repo1"]="*.log *.tmp"
ignored_files["repo2,repo5,repo8"]="README.md config.json *.bak"
ignored_files["repo3,repo12"]="docs/*"

# Common ignored files for all repositories
common_ignored="*.swp .gitignore"

# Example: Set ignored files/patterns for subgroups by subgroup ID or namespace
ignored_subgroups["123324"]="**/providers.tf"
ignored_subgroups["my-top-level-group/infrastructure-group/terraform-subgroup"]="**/providers.tf"

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
