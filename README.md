tende_bundle (MODE=full ./tende_bundle.sh to run)
coder@rudy:~/transfer-scripts$ MODE=full ./tende_bundle.sh
+ LOG_FILE=error.log
+ :
+ exec
Your branch is up to date with 'origin/main'.
[MODE: full] Processing gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__spacewire_uvc on branch main
Your branch is up to date with 'origin/main'.
Already up to date.
[BUNDLE] Creating full bundle for gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__spacewire_uvc on main
[WARNING] Branch staging not found in gm-tma/tenants/gmz/omd/swa/swa_firmware/fw-uvcs/spacewire_uvc
Fetching origin
Your branch and 'origin/main' have diverged,
and have 3 and 10 different commits each, respectively.
  (use "git pull" to merge the remote branch into yours)
[MODE: full] Processing gm-tma__infrastructure__coder-templates__questa-vnc on branch main
Your branch and 'origin/main' have diverged,
and have 3 and 10 different commits each, respectively.
  (use "git pull" to merge the remote branch into yours)

----------------------------------------------------------------------------------------------------------------------------------------

#!/usr/bin/env bash

GITLAB_TOKEN="d9z9AcayqfEsGe5sv9RG"

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


-----------------------------------------------------------------------------------------------------------------------------
-----------------------------------------------------------------------------------------------------------------------------
#!/usr/bin/env bash

GITLAB_TOKEN="d9z9AcayqfEsGe5sv9RG"

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

--------------------------------------------------------------------------------------------------------------------------------------------
--------------------------------------------------------------------------------------------------------------------------------------------











coder@rudy:~/transfer-scripts$ ./tende-airgap.sh 
[INFO] Starting GitLab Airgap Bundle Import Process
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__coder-support__coder-blob-storage__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__coder-support__coder-blob-storage__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__coder-templates__coder-templates-security-policy-project__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__coder-templates__coder-templates-security-policy-project__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__coder-templates__coder-templates-security-policy-project-security-policy-project__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__coder-templates__coder-templates-security-policy-project-security-policy-project__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__coder-templates__kubernetes-custom-security-policy-project__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__coder-templates__kubernetes-custom-security-policy-project__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__containers__code-marketplace__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__containers__code-marketplace__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__pipelines__azure-vm-pipeline__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__pipelines__azure-vm-pipeline__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__pipelines__vscode-extensions-pipeline__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__pipelines__vscode-extensions-pipeline__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__project-templates__coder-project-template__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__project-templates__coder-project-template__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__renovate-runner__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__renovate-runner__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__scripts__mcr-inspector__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__scripts__mcr-inspector__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__scripts__runner-check__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__scripts__runner-check__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__scripts__windows-scripts__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__scripts__windows-scripts__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__terraform__terraform-modules__azure-custom__aks_log_analytics__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__terraform__terraform-modules__azure-custom__aks_log_analytics__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__terraform__terraform-modules__azure-custom__aks__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__terraform__terraform-modules__azure-custom__aks__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__terraform__terraform-modules__azure-custom__avd__avd_core__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__terraform__terraform-modules__azure-custom__avd__avd_core__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__terraform__terraform-modules__azure-custom__avd__avd_session_hosts__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__terraform__terraform-modules__azure-custom__avd__avd_session_hosts__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__terraform__terraform-modules__azure-custom__key_vault__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__terraform__terraform-modules__azure-custom__key_vault__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__terraform__terraform-modules__azure-custom__log_analytics__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__terraform__terraform-modules__azure-custom__log_analytics__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__terraform__terraform-modules__azure-custom__node_pool__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__terraform__terraform-modules__azure-custom__node_pool__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__terraform__terraform-modules__azure-custom__postgres__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__terraform__terraform-modules__azure-custom__postgres__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__terraform__terraform-modules__azure-custom__storage_account__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__terraform__terraform-modules__azure-custom__storage_account__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__terraform__terraform-modules__azure-custom__virtual_network__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__terraform__terraform-modules__azure-custom__virtual_network__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__terraform__terraform-modules__azure-custom__vm__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__terraform__terraform-modules__azure-custom__vm__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__ana__requests__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__ana__requests__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__de__requests__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__de__requests__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__ied__requests__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__ied__requests__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__mns__requests__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__mns__requests__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__requests__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__requests__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-automation__findings_app__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-automation__findings_app__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-automation__runner_sandbox__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-automation__runner_sandbox__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-automation__svuml__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-automation__svuml__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-automation__uvm_error_binning__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-automation__uvm_error_binning__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-automation__uvm_error_parse__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-automation__uvm_error_parse__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-gmn-gws__gws_dashking__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-gmn-gws__gws_dashking__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-gmn-gws__ivv_gem_modem_uml__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-gmn-gws__ivv_gem_modem_uml__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-gmn-gws__rxtx_fpga-5NdtR__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-gmn-gws__rxtx_fpga-5NdtR__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-gmn-gws__rxtx_tb.dsc__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-gmn-gws__rxtx_tb.dsc__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-gmn-gws__rxtx_tb-nueQI__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-gmn-gws__rxtx_tb-nueQI__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__cic__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__cic__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__gem_ublaze_microproc__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__gem_ublaze_microproc__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ad5666__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ad5666__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ad7655__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ad7655__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ad7691__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ad7691__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ad7923__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ad7923__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ad7949__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ad7949__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ad7980__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ad7980__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ad9253__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ad9253__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ad9786__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ad9786__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_adf411x__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_adf411x__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ads8326__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ads8326__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_at28lv010_uvc-F2l5n__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_at28lv010_uvc-F2l5n__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_at28lv_eeprom__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_at28lv_eeprom__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_avalon__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_avalon__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_cad_l3h__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_cad_l3h__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ds1626__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ds1626__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_gige_temac_example__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_gige_temac_example__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_hmc1019a_uvc-nYXwz__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_hmc1019a_uvc-nYXwz__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_hmc1112_uvc__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_hmc1112_uvc__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_hmc1112_uvc-oDOlt__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_hmc1112_uvc-oDOlt__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ina226_uvc-5S1gp__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ina226_uvc-5S1gp__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_lm83_uvc-ILqby__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_lm83_uvc-ILqby__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_max106__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_max106__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_max1237_uvc-AlZbF__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_max1237_uvc-AlZbF__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_max531__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_max531__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_pci__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_pci__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_pnor_flash__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_pnor_flash__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_quick_eth_uvc-QUeQ3__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_quick_eth_uvc-QUeQ3__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_sdlc__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_sdlc__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_serpnor__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_serpnor__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_spi__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_spi__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_srio_bundle__codec__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_srio_bundle__codec__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_srio_bundle__sfpd__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_srio_bundle__sfpd__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_srio_bundle__srio__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_srio_bundle__srio__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_srio_bundle__srio_over_sfpdp__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_srio_bundle__srio_over_sfpdp__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_stft__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_stft__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_tlv2556__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_tlv2556__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_tlv5638__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_tlv5638__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_tmp422__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_tmp422__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_tmp461__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_tmp461__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_tms320c67xx__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_tms320c67xx__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_uart__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_uart__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ublazefsl__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ublazefsl__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ublaze_uvc__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ublaze_uvc__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_wishbone__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_wishbone__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__qddrx__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__qddrx__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__qeth_uvc__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__qeth_uvc__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__qpcie__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__qpcie__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__spacewire_uvc__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__spacewire_uvc__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-vendors__diamond__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-vendors__diamond__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-vendors__ise__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-vendors__ise__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-vendors__ivv_uvm-KycfX__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-vendors__ivv_uvm-KycfX__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-vendors__libero__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-vendors__libero__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-vendors__qformal_xilinx-SJyvL__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-vendors__qformal_xilinx-SJyvL__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-vendors__quartus__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-vendors__quartus__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-vendors__vivado-l7h0x__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-vendors__vivado-l7h0x__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-vendors__vsi_api-jDWEi__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-vendors__vsi_api-jDWEi__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__questa_job_mgr__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__questa_job_mgr__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__tma__tma-issue-project__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__tma__tma-issue-project__main.bundle. Skipping.
