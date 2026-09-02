---
title: ZyraxBuster - A Fast Async Content Discovery Tool Written in Rust
date: 2026-06-10
categories: [Tools, Projects]
tags: [rust, cli, content-discovery, fuzzing, vhost, bug-bounty, pentesting]
image:
  path: "https://github.com/user-attachments/assets/461a8f96-88ba-47e5-8df1-6bf823c6b5e5"
  description: ZyraxBuster — a high-performance, asynchronous directory and VHost content-discovery tool written in Rust.
author: daemonyrax
---

```
███████╗██╗   ██╗██████╗  █████╗ ██╗  ██╗██████╗ ██╗   ██╗███████╗████████╗███████╗██████╗
╚══███╔╝╚██╗ ██╔╝██╔══██╗██╔══██╗╚██╗██╔╝██╔══██╗██║   ██║██╔════╝╚══██╔══╝██╔════╝██╔══██╗
  ███╔╝  ╚████╔╝ ██████╔╝███████║ ╚███╔╝ ██████╔╝██║   ██║███████╗   ██║   █████╗  ██████╔╝
 ███╔╝    ╚██╔╝  ██╔══██╗██╔══██║ ██╔██╗ ██╔══██╗██║   ██║╚════██║   ██║   ██╔══╝  ██╔══██╗
███████╗   ██║   ██║  ██║██║  ██║██╔╝ ██╗██████╔╝╚██████╔╝███████║   ██║   ███████╗██║  ██║
╚══════╝   ╚═╝   ╚═╝  ╚═╝╚═╝  ╚═╝╚═╝  ╚═╝╚═════╝  ╚═════╝ ╚══════╝   ╚═╝   ╚══════╝╚═╝  ╚═╝
```

**Fast async content discovery tool written in Rust**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Rust](https://img.shields.io/badge/Rust-2021-blue.svg)](https://www.rust-lang.org/)
[![Release](https://img.shields.io/github/v/release/ans-inayat/zyraxbuster)](https://github.com/ans-inayat/zyraxbuster/releases)

> A high-performance, asynchronous directory/content brute-forcer built on `tokio` + `reqwest`.

## Features

- **Async concurrent scanning** — configurable worker pool (`-t`)
- **Auto wordlist detection** — automatically finds wordlists from `/usr/share/wordlists/` and seclists
- **Wordlist listing** — browse all available wordlists with `--list-wordlists`
- **Extension appending** — `-x php,html,bak,txt`
- **Status code filtering** — whitelist (`-s`) or blacklist (`-b`, default excludes 404)
- **Content-length filtering** — `--min-length`, `--exclude-length` to cut soft-404 noise
- **Random User-Agent** — rotate User-Agent per request (`--random-agent`)
- **Rate limiting** — delay between requests (`--delay`)
- **Custom headers** — `-H "Authorization: Bearer ..."` (repeatable)
- **Redirect following** — toggle with `-r`
- **TLS bypass** — skip cert verification for self-signed certs (`-k`)
- **Retry on failure** — configurable retries (`--retries`)
- **JSON output** — export results as JSON (`--json -o results.json`)
- **Colorized output** — live results with progress bar
- **Short wordlist names** — use `-w common.txt` and it resolves automatically

## Installation

### From Source (Recommended)

Requires **Rust 2021+** and **Cargo**.

```bash
git clone https://github.com/ans-inayat/zyraxbuster.git
cd zyraxbuster
cargo build --release
# Binary at target/release/zyraxbuster
```

### Download Release

Download the latest binary from [Releases](https://github.com/ans-inayat/zyraxbuster/releases).

```bash
chmod +x zyraxbuster-linux-x86_64
./zyraxbuster-linux-x86_64 --help
```

### Install with Cargo

```bash
cargo install --path .
```

## New Features

- Flags
  - `--vhost`
  - `--auto-filter`

Usage examples:

**Basic VHost fuzzing**

```bash
zyraxbuster -u http://10.129.13.57 -w subdomains.txt --vhost -H "Host: paperwork.htb"
```

**With auto-filter (excludes baseline-matching responses)**

```bash
zyraxbuster -u http://10.129.13.57 -w subdomains.txt --vhost --auto-filter
```

**Full example with JSON output**

```bash
zyraxbuster -u http://target.com -w subdomains.txt --vhost --auto-filter -t 50 --random-agent -o results.json --json
```

**How it works**

- Directory mode (default): `GET http://target.com/WORD`
- VHost mode (`--vhost`): `GET http://target.com/` with `Host: WORD.target.com`
- Auto-filter (`--auto-filter`): makes a baseline request first, then filters out responses matching the baseline (same status + same size)

## Usage

```
zyraxbuster [OPTIONS]
```

### Basic Directory Scan

```bash
zyraxbuster -u https://target.com -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
```

### Auto-Detect Wordlist

```bash
# No -w needed — auto-detects common.txt from seclists
zyraxbuster -u https://target.com
```

### List Available Wordlists

```bash
zyraxbuster --list-wordlists
```

### Scan with Extensions

```bash
zyraxbuster -u https://target.com -w common.txt -x php,html,bak,zip
```

### Filter by Status Codes

```bash
# Only show 200, 301, 302
zyraxbuster -u https://target.com -w common.txt -s 200,301,302

# Hide 403 and 500 (in addition to default 404)
zyraxbuster -u https://target.com -w common.txt -b 404,403,500
```

### Filter Soft-404s

```bash
# Exclude pages with exact size of 1024 bytes
zyraxbuster -u https://target.com -w common.txt --exclude-length 1024

# Only show results larger than 500 bytes
zyraxbuster -u https://target.com -w common.txt --min-length 500
```

### Authenticated Scan

```bash
zyraxbuster -u https://target.com -w common.txt \
  -H "Cookie: session=abc123" \
  -H "Authorization: Bearer token123" \
  -r -o results.txt
```

### Stealth Scan (Random UA + Rate Limit)

```bash
zyraxbuster -u https://target.com -w common.txt \
  --random-agent --delay 100 -t 10
```

### JSON Output

```bash
zyraxbuster -u https://target.com -w common.txt -o results.json --json
```

### Full Example

```bash
zyraxbuster \
  -u https://payment.tallymarkscloud.com:4431/admin/ \
  -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt \
  -x php,html,bak \
  -t 50 \
  --random-agent \
  --delay 50 \
  -o results.txt
```

## CLI Options

| Flag | Description | Default |
|------|-------------|---------|
| `-u, --url` | Target base URL | *(required)* |
| `-w, --wordlist` | Path to wordlist (auto-detects if omitted) | auto |
| `-t, --threads` | Number of concurrent workers | `40` |
| `-x, --extensions` | Comma-separated extensions to append | — |
| `-s, --status-codes` | Whitelist of status codes to show | — |
| `-b, --blacklist-codes` | Blacklist of status codes to hide | `404` |
| `--timeout` | Request timeout in seconds | `10` |
| `-r, --follow-redirects` | Follow HTTP redirects | `false` |
| `--user-agent` | Custom User-Agent string | `zyraxbuster/0.1` |
| `--random-agent` | Rotate User-Agent per request | `false` |
| `--add-slash` | Add trailing slash to each word | `false` |
| `-o, --output` | Output file path | — |
| `--json` | Output in JSON format | `false` |
| `--min-length` | Minimum content-length to show | — |
| `--exclude-length` | Exclude exact content-length | — |
| `-H, --header` | Custom header (repeatable) | — |
| `-k, --insecure` | Skip TLS verification | `false` |
| `--retries` | Retry count on network error | `1` |
| `--delay` | Delay between requests (ms) | `0` |
| `--list-wordlists` | List available wordlists and exit | `false` |

## Wordlist Auto-Detection

ZyraxBuster automatically searches for wordlists in these locations, in order:

1. `/usr/share/wordlists/seclists/Discovery/Web-Content/common.txt`
2. `/usr/share/seclists/Discovery/Web-Content/common.txt`
3. `/usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-directories.txt`
4. `/usr/share/seclists/Discovery/Web-Content/raft-large-directories.txt`
5. `/usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-words.txt`
6. `/usr/share/seclists/Discovery/Web-Content/raft-large-words.txt`
7. `/usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt`
8. `/usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt`
9. `/usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt`
10. `/usr/share/wordlists/dirbuster/directory-list-2.3-small.txt`
11. `/usr/share/wordlists/common.txt`
12. `/usr/share/wordlists/rockyou.txt`

You can also use **short names** with `-w`:

```bash
zyraxbuster -u https://target.com -w common.txt
zyraxbuster -u https://target.com -w raft-large-directories.txt
```

## Project Structure

```
zyraxbuster/
├── src/
│   ├── main.rs      — entry point, banner, wiring
│   ├── cli.rs       — clap argument definitions
│   ├── wordlist.rs  — wordlist loading, auto-detection, candidate generation
│   └── scanner.rs   — async HTTP scanning engine
├── Cargo.toml
├── LICENSE
├── CHANGELOG.md
├── CONTRIBUTING.md
├── SECURITY.md
└── .github/
    ├── ISSUE_TEMPLATE/
    ├── PULL_REQUEST_TEMPLATE.md
    └── workflows/
        └── release.yml
```

## Building

```bash
# Debug build
cargo build

# Release build (optimized)
cargo build --release

# Run directly
cargo run -- -u https://target.com -w wordlist.txt
```

### Cross-Compilation

```bash
# Linux x86_64
cargo build --release --target x86_64-unknown-linux-gnu

# Linux ARM64
cargo build --release --target aarch64-unknown-linux-gnu

# macOS
cargo build --release --target x86_64-apple-darwin
cargo build --release --target aarch64-apple-darwin

# Windows
cargo build --release --target x86_64-pc-windows-msvc
```

## Performance

ZyraxBuster is designed for speed:

- **Async I/O** — non-blocking requests via `tokio` + `reqwest`
- **Configurable concurrency** — tune `-t` based on target tolerance
- **Rust performance** — zero-cost abstractions, no GC pauses
- **Optimized release** — LTO, single codegen unit, panic=abort

Typical performance on a single machine:

| Threads | Requests/sec |
|---------|-------------|
| 10 | ~500 |
| 50 | ~2,000 |
| 100 | ~4,000 |

*Actual performance depends on network latency and target response time.*

## Roadmap

- [ ] VHost / subdomain fuzzing mode
- [ ] Recursive directory scanning
- [ ] FUZZ-keyword templating (like ffuf)
- [ ] Response body diffing / similarity filtering
- [ ] Proxy support (`--proxy`)
- [ ] Multiple wordlist support
- [ ] Rate limit auto-detection
- [ ] colored table output mode

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

## Security

See [SECURITY.md](SECURITY.md) for reporting vulnerabilities.

## License

This project is licensed under the **MIT License** — see [LICENSE](LICENSE) for details.

## Disclaimer

**This tool is for authorized security testing only.** Unauthorized scanning of systems you don't own or have permission to test is illegal in most jurisdictions. Always obtain proper authorization before scanning.

## Author

**ans-inayat** — [GitHub](https://github.com/ans-inayat)
