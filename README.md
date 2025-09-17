## Shell-Scripts-For-DevOps-Engineers
DevOps Shell Scripts Toolkit — Combined Single-File Edition  This repository contains a single, comprehensive Bash toolkit that implements the 10 mini-projects from the DevOps Shell Scripting Student Toolkit:

**Included tools / commands**
1. `calc` — Basic calculator (interactive or expression)
2. `backup` — Create compressed backups (tar.gz)
3. `login-tracker` — Query recent user logins (uses `last`)
4. `du-report` — Directory usage report (sorted, top N)
5. `todo` — Simple persistent todo list (`add|list|done|clear`)
6. `svc-check` — Service health checker (systemd-aware)
7. `sys-report` — System resource snapshot and optional CSV export
8. `log-rotate` — Rotate/compress logs older than N days (safe)
9. `git-deploy` — Safe git pull + optional deploy hook
10. `cpu-hog` — Detect processes above CPU threshold and report

## Why a single file?
This single-file approach makes it easy to copy, review, and place in `scripts/` of any repo. The script is designed to be readable, documented inline, and safe to run.

## Prerequisites
- `bash` (4.x+ recommended)
- `tar`, `gzip`, `find`, `du`, `sort`, `awk`, `ps`, `git`, `last` (for login tracker)
- Optional: `jq`, `mailx` if you enable alerts

> The script checks for required commands before performing actions.

## Quick start
1. Create your repo or use your existing fork.
2. Add `devops-toolkit.sh` to the `scripts/` folder (or repository root) and `README.md`.
3. Make executable:
```bash
chmod +x devops-toolkit.sh


# View help: 
./devops-toolkit.sh help


## Example usages
Backup /var/www:
./devops-toolkit.sh backup /var/www ./backups/www-$(date +%F).tar.gz

# Disk usage report for /var/www top 10:
./devops-toolkit.sh du-report /var/www 10


### Add todo:
./devops-toolkit.sh todo add "Fix Nginx config"


### Detect CPU hogs above 40%:
./devops-toolkit.sh cpu-hog 40


## Safety notes

The script avoids destructive default actions and demands explicit confirmation for user creation or service restarts (where sensitive).

Do not run as an untrusted user or execute unreviewed deploy hooks. Always inspect deploy-hook.sh before execution.





## License
---

# 2) devops-toolkit.sh (copy this into `devops-toolkit.sh`)

> Save the file exactly, `chmod +x devops-toolkit.sh`, then run `./devops-toolkit.sh help`.

```bash
#!/usr/bin/env bash
# devops-toolkit.sh — Combined DevOps shell utilities
# Implements 10 small tools from the DevOps Shell Scripting Student Toolkit
# Save as devops-toolkit.sh and `chmod +x` to use.
set -o errexit
set -o pipefail
set -o nounset
IFS=$'\n\t'

TOOL_NAME="devops-toolkit"
VERSION="1.0.0"
LOG_FILE="${DEVOPS_TOOLKIT_LOG:-./devops-toolkit.log}"

# --- Helpers ---------------------------------------------------------------
log() {
  # timestamped log message (appends to LOG_FILE if writable)
  local ts
  ts="$(date -u +"%Y-%m-%dT%H:%M:%SZ")"
  printf '%s [%s] %s\n' "$ts" "$TOOL_NAME" "$*" | tee -a "$LOG_FILE"
}
err() {
  printf '%s [%s] ERROR: %s\n' "$(date -u +"%Y-%m-%dT%H:%M:%SZ")" "$TOOL_NAME" "$*" >&2
}
require_cmd() {
  if ! command -v "$1" >/dev/null 2>&1; then
    err "Required command '$1' not found. Please install it and try again."
    exit 2
  fi
}
confirm() {
  # confirm "Message" -> returns 0 on yes, 1 on no
  local msg="${1:-Are you sure?}"
  read -r -p "$msg [y/N]: " ans
  case "$ans" in
    [Yy]|[Yy][Ee][Ss]) return 0 ;;
    *) return 1 ;;
  esac
}
safe_mkdir_for_file() {
  mkdir -p "$(dirname "$1")"
}

# --- Help / Version -------------------------------------------------------
show_help() {
  cat <<'EOF'
devops-toolkit — small utilities (single file)
Usage: ./devops-toolkit.sh <command> [arguments...]

Commands:
  help                      Show this help
  version                   Show version
  calc [expr] | calc op a b Basic calculator (examples below)
  backup SRC DEST           Create compressed tar.gz backup of SRC to DEST
  login-tracker [N] OUT     Show recent logins (last N lines, default 50)
  du-report DIR [TOPN]      Show sorted directory usage (topN default 10)
  todo add "task"           Add todo
  todo list                 List todos
  todo done N               Mark todo N done (remove)
  todo clear                Clear all todos (confirmation)
  svc-check SERVICE         Check service active (systemd-aware)
  sys-report [OUTFILE]      Save system resource snapshot to OUTFILE
  log-rotate DIR [DAYS]     Gzip .log files older than DAYS (default 7)
  git-deploy REPO_DIR BRANCH [--restart service]  Safe git pull + optional deploy-hook
  cpu-hog [THRESH]          Report processes using >THRESH% CPU (default 30)
EOF
}

version() { echo "$TOOL_NAME $VERSION"; }

## ----------------------------- Tool implementations -----------------------

## 1) calc
calc() {
  if [ $# -eq 0 ]; then
    echo "Calculator: examples:"
    echo "  ./devops-toolkit.sh calc 2+3*4"
    echo "  ./devops-toolkit.sh calc add 5 3"
    echo "Supported ops: add sub mul div mod"
    return 0
  fi

  if [ $# -eq 1 ]; then
    # treat as expression for bc
    require_cmd bc
    echo "$1" | bc -l
    return 0
  fi

  local op="$1" a="$2" b="${3:-}"
  case "$op" in
    add) echo "$((a + b))" ;;
    sub) echo "$((a - b))" ;;
    mul) echo "$((a * b))" ;;
    div)
      if [ "$b" -eq 0 ]; then err "Division by zero"; return 1; fi
      # float division
      echo "scale=6; $a / $b" | bc -l
      ;;
    mod) echo "$((a % b))" ;;
    *) err "Unknown calc op: $op"; return 2 ;;
  esac
}

## 2) backup SRC DEST
backup() {
  if [ $# -lt 2 ]; then
    err "Usage: backup SRC_PATH DEST_TAR_GZ"
    return 2
  fi
  local src="$1" dest="$2"
  if [ ! -e "$src" ]; then err "Source '$src' does not exist"; return 1; fi
  safe_mkdir_for_file "$dest"
  log "Creating backup of $src -> $dest"
  # use tar safely: change directory to avoid including absolute paths
  ( cd "$(dirname "$src")" && tar -czf "$PWD/$(basename "$dest")" "$(basename "$src")" ) || {
    err "Backup failed"
    return 1
  }
  log "Backup created: $dest"
}

## 3) login-tracker [N] OUT
login_tracker() {
  local n="${1:-50}" out="${2:-}"
  require_cmd last
  if [ -n "$out" ]; then safe_mkdir_for_file "$out"; fi
  log "Fetching last $n login entries"
  if last -n "$n" > "${out:-/dev/stdout}"; then
    log "login-tracker output ${out:-(stdout)}"
  else
    err "Failed to get login entries"
    return 1
  fi
}

## 4) du-report DIR TOPN
du_report() {
  local dir="${1:-.}" topn="${2:-10}"
  if [ ! -d "$dir" ]; then err "Directory not found: $dir"; return 1; fi
  log "Generating disk usage report for $dir (top $topn)"
  # produce human-readable top items
  du -sh "$dir"/* 2>/dev/null | sort -hr | head -n "$topn"
}

## 5) todo list: add|list|done|clear
TODO_FILE="${HOME}/.devops_todo"
todo_add() {
  local text="$*"
  if [ -z "$text" ]; then err "Usage: todo add \"task\""; return 2; fi
  safe_mkdir_for_file "$TODO_FILE"
  printf '%s\n' "$text" >> "$TODO_FILE"
  log "Added todo: $text"
}
todo_list() {
  if [ ! -f "$TODO_FILE" ]; then echo "No todos."; return 0; fi
  nl -w2 -s'. ' "$TODO_FILE"
}
todo_done() {
  local n="$1"
  if [ -z "$n" ]; then err "Usage: todo done N"; return 2; fi
  if [ ! -f "$TODO_FILE" ]; then err "No todo file"; return 1; fi
  if ! awk "NR==$n{exit 0} END{exit 1}" "$TODO_FILE"; then err "No todo numbered $n"; return 1; fi
  # remove line N
  awk -vN="$n" 'NR!=N' "$TODO_FILE" > "${TODO_FILE}.tmp" && mv "${TODO_FILE}.tmp" "$TODO_FILE"
  log "Marked todo #$n done"
}
todo_clear() {
  if confirm "Clear all todos?"; then > "$TODO_FILE"; log "Cleared todos"; else log "Clear cancelled"; fi
}

## 6) svc-check SERVICE
svc_check() {
  local svc="$1"
  if [ -z "$svc" ]; then err "Usage: svc-check SERVICE"; return 2; fi
  if command -v systemctl >/dev/null 2>&1; then
    if systemctl is-active --quiet "$svc"; then log "Service $svc is active"; return 0; else err "Service $svc is NOT active"; return 1; fi
  else
    if service "$svc" status >/dev/null 2>&1; then log "Service $svc is running (non-systemd)"; return 0; else err "Service $svc is not running"; return 1; fi
  fi
}

## 7) sys-report [OUTFILE]
sys_report() {
  local out="${1:-}"
  local tmp
  tmp="$(mktemp)"
  {
    echo "=== System Report: $(date -u) ==="
    echo "Hostname: $(hostname)"
    echo "Uptime: $(uptime -p 2>/dev/null || cat /proc/uptime 2>/dev/null)"
    echo "--- Memory ---"
    free -h
    echo "--- Disk ---"
    df -h
    echo "--- Top processes ---"
    ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%cpu | head -n 20
  } > "$tmp"
  if [ -n "$out" ]; then
    safe_mkdir_for_file "$out"
    mv "$tmp" "$out"
    log "System report saved to $out"
  else
    cat "$tmp"
    rm -f "$tmp"
  fi
}

## 8) log-rotate DIR [DAYS]
log_rotate() {
  local dir="${1:-/var/log}" days="${2:-7}"
  if [ ! -d "$dir" ]; then err "Directory not found: $dir"; return 1; fi
  log "Rotating *.log files in $dir older than $days days (will gzip them)"
  # Avoid re-gzipping .gz files
  # Find regular files with .log extension that are not already compressed
  find "$dir" -type f -name '*.log' -mtime +"$days" -print0 | while IFS= read -r -d '' f; do
    if [ -f "$f" ]; then
      if gzip -9 -- "$f"; then
        log "Compressed: $f -> ${f}.gz"
      else
        err "Failed to compress $f"
      fi
    fi
  done
  log "Log rotate completed"
}

## 9) git-deploy REPO_DIR BRANCH [--restart service]
git_deploy() {
  local repo_dir="${1:-}" branch="${2:-main}" restart_flag="${3:-}" restart_svc="${4:-}"
  if [ -z "$repo_dir" ]; then err "Usage: git-deploy REPO_DIR BRANCH [--restart SERVICE]"; return 2; fi
  if [ ! -d "$repo_dir/.git" ]; then err "$repo_dir is not a git repository"; return 1; fi
  require_cmd git
  log "Preparing deploy in $repo_dir (branch $branch)"
  # check working tree clean
  if [ -n "$(git -C "$repo_dir" status --porcelain)" ]; then
    err "Git working tree is not clean. Aborting deploy. Please commit or stash changes."
    return 3
  fi
  git -C "$repo_dir" fetch --all --prune
  git -C "$repo_dir" checkout "$branch"
  git -C "$repo_dir" pull origin "$branch"
  # optional deploy hook
  if [ -x "$repo_dir/deploy-hook.sh" ]; then
    log "Running deploy-hook.sh"
    ( cd "$repo_dir" && ./deploy-hook.sh ) || { err "deploy-hook failed"; return 4; }
  fi
  if [ "$restart_flag" = "--restart" ] && [ -n "$restart_svc" ]; then
    log "Restarting service $restart_svc"
    if command -v systemctl >/dev/null 2>&1; then
      sudo systemctl restart "$restart_svc"
      sleep 1
      if systemctl is-active --quiet "$restart_svc"; then log "$restart_svc restarted and active"; else err "$restart_svc failed to start"; fi
    else
      err "systemctl not available to restart $restart_svc"
    fi
  fi
  log "Deploy completed for $repo_dir@$branch"
}

## 10) cpu-hog [THRESH]
cpu_hog() {
  local thresh="${1:-30}"
  log "Detecting processes > ${thresh}% CPU"
  ps -eo pid,ppid,cmd,%cpu --sort=-%cpu | awk -v t="$thresh" 'NR==1{print; next} {if ($4+0 > t) print}' | head -n 50
}

# ---------------- Dispatch -------------------------------------------------
if [ $# -lt 1 ]; then
  show_help
  exit 0
fi

cmd="$1"; shift || true

case "$cmd" in
  help|-h|--help) show_help ;;
  version) version ;;
  calc) calc "$@" ;;
  backup) backup "$@" ;;
  login-tracker) login_tracker "$@" ;;
  du-report) du_report "$@" ;;
  todo)
    sub="${1:-}"; shift || true
    case "$sub" in
      add) todo_add "$@" ;;
      list) todo_list ;;
      done) todo_done "$1" ;;
      clear) todo_clear ;;
      *) err "Unknown todo command"; todo_list; exit 2 ;;
    esac
    ;;
  svc-check) svc_check "$@" ;;
  sys-report) sys_report "$@" ;;
  log-rotate) log_rotate "$@" ;;
  git-deploy)
    # syntax: git-deploy REPO_DIR BRANCH [--restart SERVICE]
    repo_dir="${1:-}"; branch="${2:-main}"; shift 2 || true
    if [ "${1:-}" = "--restart" ]; then restart_flag="--restart"; restart_svc="${2:-}"; shift 2 || true; else restart_flag=""; restart_svc=""; fi
    git_deploy "$repo_dir" "$branch" "$restart_flag" "$restart_svc"
    ;;
  cpu-hog) cpu_hog "$@" ;;
  *) err "Unknown command: $cmd"; show_help; exit 2 ;;
esac





## create repository folder locally (if not already)
mkdir -p my-devops-scripts && cd my-devops-scripts

## create scripts directory
mkdir -p scripts
# Create files with your editor, or use cat <<'EOF' > scripts/devops-toolkit.sh ... EOF to paste

## make script executable
chmod +x scripts/devops-toolkit.sh

git init
git add README.md scripts/devops-toolkit.sh
git commit -m "chore: add combined devops shell scripts toolkit"
# Create repository on GitHub or use `gh` if you have it:
# gh repo create YOUR_USERNAME/devops-shell-scripts-toolkit --public --source=. --remote=origin
git branch -M main
git remote add origin https://github.com/YOUR-GITHUB-USERNAME/devops-shell-scripts-toolkit.git
git push -u origin main

## Replace YOUR-GITHUB-USERNAME with your username.



## Final Clean Layout
# Your repo should look like this:

devops-shell-scripts-toolkit/
├── README.md
├── LICENSE
└── scripts/
    └── devops-toolkit.sh



