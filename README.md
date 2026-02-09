# Wordlists (Raw-friendly)

A small, curated set of web discovery wordlists I use for labs (especially **TryHackMe**).  
The goal is simple: **high-signal lists**, easy to use, and **usable directly via raw URLs** — so you can run tools like `ffuf` without cloning the whole repository. 

## Why this repo exists

- ✅ Curated wordlists (directories, files, auth paths, parameters, backups, etc.)
- ✅ Works great with `ffuf`, `feroxbuster`, `gobuster`, etc.
- ✅ Raw URL usage supported (quick one-liners)
- ✅ Easy to vendor into your own workflow (fish functions, bash aliases, CI, notes)

---

## Raw base URL

You can fetch files directly from GitHub’s raw content endpoint.

Set a base URL once and reuse it:

### POSIX (bash,zsh,...)
```sh
BASE="https://raw.githubusercontent.com/dme86/wordlists/refs/heads/main"
```

### fish
```sh
set -gx BASE "https://raw.githubusercontent.com/dme86/wordlists/refs/heads/main"
```

## Use without cloning

Examples

### Directory discovery with ffuf

#### POSIX (bash/zsh) using process substitution

```sh
ffuf -u "http://IP_ADDR/FUZZ" \
  -w <(curl -fsSL "$BASE/web/dirs/raft-medium-directories.txt") \
  -mc 200,204,301,302,307,401,403 -fc 404 -ac -t 50 -timeout 5
```
#### fish using `psub`

```sh
ffuf -u "http://IP_ADDR/FUZZ" \
  -w (curl -fsSL "$BASE/web/dirs/raft-medium-directories.txt" | psub) \
  -mc 200,204,301,302,307,401,403 -fc 404 -ac -t 50 -timeout 5
````

### File discovery (finding `.php`, backups, logs, etc.)

#### POSIX (bash/zsh)

```sh
ffuf -u "http://IP_ADDR/FUZZ" \
  -w <(curl -fsSL "$BASE/web/files/raft-medium-files.txt") \
  -e .php,.txt,.bak,.old,.zip,.tar,.tar.gz,.log \
  -mc 200,204,301,302,307,401,403 -fc 404 -ac -t 50 -timeout 5
```

#### fish

```sh
ffuf -u "http://IP_ADDR/FUZZ" \
  -w (curl -fsSL "$BASE/web/files/raft-medium-files.txt" | psub) \
  -e .php,.txt,.bak,.old,.zip,.tar,.tar.gz,.log \
  -mc 200,204,301,302,307,401,403 -fc 404 -ac -t 50 -timeout 5
````

## Wrapper: `wlscan` (fish + POSIX)

`wlscan` is a tiny convenience wrapper that:
- selects a wordlist by a short key (`quick`, `raftm`, `filesm`, …)
- downloads the list via this repo’s raw URL (no clone required)
- runs `ffuf` with good default flags
- lets you pass additional `ffuf` arguments at the end

### Wordlist keys

- `quick`  → `web/dirs/quickhits.txt`
- `common` → `web/dirs/common.txt`
- `rafts`  → `web/dirs/raft-small-directories.txt`
- `raftm`  → `web/dirs/raft-medium-directories.txt`
- `filess` → `web/files/raft-small-files.txt`
- `filesm` → `web/files/raft-medium-files.txt`
- `logins` → `web/auth/Logins.fuzz.txt`
- `dbb`    → `web/files/Common-DB-Backups.txt`
- `params` → `web/params/burp-parameter-names.txt`

You can pin a commit by setting `WLSCAN_BASE` to a commit SHA URL.

---

# fish implementation

Create `~/.config/fish/functions/wlscan.fish`:

```bash
function wlscan --description "ffuf wrapper using wordlists from raw GitHub (keys)"
    # Base raw URL (override per-shell if you want to pin a commit)
    set -q WLSCAN_BASE; or set -l WLSCAN_BASE "https://raw.githubusercontent.com/dme86/wordlists/refs/heads/main"

    set -l mode $argv[1]
    set -l key  $argv[2]
    set -l target $argv[3]

    if test -z "$mode" -o -z "$key" -o -z "$target"
        echo "Usage:"
        echo "  wlscan dir   <key> <base_url> [extra ffuf args...]"
        echo "  wlscan file  <key> <base_url> [extra ffuf args...]"
        echo "  wlscan param <key> <url>      [extra ffuf args...]"
        echo ""
        echo "Examples:"
        echo "  wlscan dir raftm  http://10.10.10.10"
        echo "  wlscan dir quick  http://10.10.10.10 -fs 166"
        echo "  wlscan file filesm http://10.10.10.10 -e .php,.txt,.bak,.zip"
        echo "  wlscan param params http://10.10.10.10/index.php"
        return 1
    end

    # Map keys to raw URLs
    set -l wl_url ""
    switch $key
        case quick
            set wl_url "$WLSCAN_BASE/web/dirs/quickhits.txt"
        case common
            set wl_url "$WLSCAN_BASE/web/dirs/common.txt"
        case rafts
            set wl_url "$WLSCAN_BASE/web/dirs/raft-small-directories.txt"
        case raftm
            set wl_url "$WLSCAN_BASE/web/dirs/raft-medium-directories.txt"
        case filess
            set wl_url "$WLSCAN_BASE/web/files/raft-small-files.txt"
        case filesm
            set wl_url "$WLSCAN_BASE/web/files/raft-medium-files.txt"
        case logins
            set wl_url "$WLSCAN_BASE/web/auth/Logins.fuzz.txt"
        case dbb
            set wl_url "$WLSCAN_BASE/web/files/Common-DB-Backups.txt"
        case params
            set wl_url "$WLSCAN_BASE/web/params/burp-parameter-names.txt"
        case '*'
            echo "Unknown key: $key" 1>&2
            echo "Keys: quick common rafts raftm filess filesm logins dbb params" 1>&2
            return 1
    end

    # Fetch wordlist to a temp file (fish-friendly)
    set -l wl (curl -fsSL "$wl_url" | psub)
    or begin
        echo "Failed to download wordlist: $wl_url" 1>&2
        return 1
    end

    # Defaults (tweak as you like)
    set -l defaul
````

Optional: pin a specific commit for reproducibility:
```sh
set -Ux WLSCAN_BASE "https://raw.githubusercontent.com/dme86/wordlists/<COMMIT_SHA>"
````

# POSIX (bash/zsh) implementation

Create a script called `wlscan` somewhere in your `PATH` (e.g. `~/bin/wlscan`) and `chmod +x` it:

```bash
#!/usr/bin/env sh
set -eu

BASE="${WLSCAN_BASE:-https://raw.githubusercontent.com/dme86/wordlists/refs/heads/main}"

mode="${1:-}"
key="${2:-}"
target="${3:-}"

if [ -z "$mode" ] || [ -z "$key" ] || [ -z "$target" ]; then
  cat <<'USAGE'
Usage:
  wlscan dir   <key> <base_url> [extra ffuf args...]
  wlscan file  <key> <base_url> [extra ffuf args...]
  wlscan param <key> <url>      [extra ffuf args...]

Examples:
  wlscan dir raftm  http://10.10.10.10
  wlscan dir quick  http://10.10.10.10 -fs 166
  wlscan file filesm http://10.10.10.10 -e .php,.txt,.bak,.zip
  wlscan param params http://10.10.10.10/index.php
USAGE
  exit 1
fi

case "$key" in
  quick)  wl="$BASE/web/dirs/quickhits.txt" ;;
  common) wl="$BASE/web/dirs/common.txt" ;;
  rafts)  wl="$BASE/web/dirs/raft-small-directories.txt" ;;
  raftm)  wl="$BASE/web/dirs/raft-medium-directories.txt" ;;
  filess) wl="$BASE/web/files/raft-small-files.txt" ;;
  filesm) wl="$BASE/web/files/raft-medium-files.txt" ;;
  logins) wl="$BASE/web/auth/Logins.fuzz.txt" ;;
  dbb)    wl="$BASE/web/files/Common-DB-Backups.txt" ;;
  params) wl="$BASE/web/params/burp-parameter-names.txt" ;;
  *) echo "Unknown key: $key" >&2; exit 1 ;;
esac

# defaults
defaults="-t 50 -timeout 5 -ac -fc 404 -mc 200,204,301,302,307,401,403"

shift 3

case "$mode" in
  dir)
    ffuf -u "$target/FUZZ" -w <(curl -fsSL "$wl") $defaults "$@"
    ;;
  file)
    ffuf -u "$target/FUZZ" -w <(curl -fsSL "$wl") $defaults "$@"
    ;;
  param)
    ffuf -u "$target?FUZZ=test" -w <(curl -fsSL "$wl") $defaults "$@"
    ;;
  *)
    echo "Unknown mode: $mode (dir|file|param)" >&2
    exit 1
    ;;
esac
````

Pin to a commit if you want deterministic results:
```sh
export WLSCAN_BASE="https://raw.githubusercontent.com/dme86/wordlists/<COMMIT_SHA>"
````
