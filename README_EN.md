# KSubdomain: Ultra-Fast Stateless Subdomain Enumeration Tool

[![Release](https://img.shields.io/github/release/boy-hack/ksubdomain.svg)](https://github.com/boy-hack/ksubdomain/releases) [![Go Report Card](https://goreportcard.com/badge/github.com/boy-hack/ksubdomain)](https://goreportcard.com/report/github.com/boy-hack/ksubdomain) [![License](https://img.shields.io/github/license/boy-hack/ksubdomain)](https://github.com/boy-hack/ksubdomain/blob/main/LICENSE)

[中文文档](./readme.md) | **English**

**KSubdomain is a stateless subdomain enumeration tool delivering unprecedented scanning speed with extremely low memory consumption.** By directly operating network adapters via raw sockets — bypassing the OS kernel stack — paired with a reliable state table retransmission mechanism, it ensures result completeness. Supports Windows, Linux, and macOS.

## 🚀 Core Advantages

- **Lightning-Fast:** Stateless scanning operating directly on network adapters — **7× faster** than massdns, **10× faster** than dnsx
- **Low Resource Usage:** Object pools + global memory pools keep memory footprint minimal even on massive domain lists
- **Dynamic Timeout:** RTT sliding window (TCP RFC 6298 EWMA, α=0.125) auto-adjusts timeouts — no manual tuning needed
- **Multi-NIC Parallel Sending:** Repeat `--interface` for multiple adapters; goroutines share a single `domainChan` for natural load balancing
- **Streaming SDK:** `EnumStream`/`VerifyStream` callback APIs for real-time result processing
- **Cross-Platform:** Windows, Linux, macOS

## ⚡ Performance

4-core CPU, 5M bandwidth, 100k wordlist:

| Tool | Method | Time | Found |
|------|--------|------|-------|
| **KSubdomain** | pcap raw socket | **~30 sec** | 1397 |
| massdns | pcap/socket | ~3 min 29 sec | 1396 |
| dnsx | socket | ~5 min 26 sec | 1396 |

## 📦 Installation

### Pre-built Binary

Download the latest release from [Releases](https://github.com/boy-hack/ksubdomain/releases).

**Dependencies:**
- **Windows:** Install [Npcap](https://npcap.com/)
- **Linux:** Statically compiled — usually no extra steps; install `libpcap-dev` if needed
- **macOS:** libpcap built-in, no installation needed

### Build from Source

```bash
git clone https://github.com/boy-hack/ksubdomain.git
cd ksubdomain
go build -ldflags "-X github.com/boy-hack/ksubdomain/v2/pkg/core/conf.Version=v2.x.y" \
    -o ksubdomain ./cmd/ksubdomain
```

Or via `go install`:

```bash
go install github.com/boy-hack/ksubdomain/v2/cmd/ksubdomain@latest
```

> **Note:** Requires root / `CAP_NET_RAW` privilege for raw packet capture.

## 📖 Usage

```
Usage:
  ksubdomain [global options] <command> [command options]

Commands:
  enum, e      Enumerate subdomains via dictionary brute-force
  verify, v    Verify a domain list for DNS resolution
  test         Test maximum packet rate of local network adapter
  device       List available network interfaces
  help, h      Show help

Global Options:
  --help, -h       Show help
  --version, -v    Print version
```

### Verify Mode

Check whether a list of domains resolves in DNS.

```
Options:
  --domain, -d          Target domain(s) (repeatable)
  --filename, -f        File containing domain list
  --stdin               Read domains from stdin
  --bandwidth, -b       Bandwidth limit, e.g. 5m, 10m (default: 3m)
  --resolvers, -r       DNS resolver(s), uses built-in defaults
  --output, -o          Output file path
  --format              Output format: txt (default), json, csv, jsonl
  --silent, -s          Silent mode: suppress banner/logs, output domains only
  --only-domain, --od   Output domains only (no IPs/records)
  --retry               Retry count; -1 = infinite (default: 3)
  --timeout             DNS query timeout in seconds (default: 6)
  --interface, -e       Network interface (repeatable for multi-NIC)
  --wildcard-filter     Wildcard filter: none (default), basic, advanced
  --predict             Enable subdomain prediction
  --quiet, -q           Suppress screen output (file only)
  --color, -c           Colorized output
```

```bash
# Verify single/multiple domains
./ksubdomain v -d www.example.com -d mail.example.com

# Read from file, save results
./ksubdomain v -f domains.txt -o output.txt

# Pipe to httpx (silent + domain-only)
cat domains.txt | ./ksubdomain v --stdin -b 10m -s --od | httpx -silent

# Advanced wildcard filter, JSONL output
./ksubdomain v -f domains.txt --wildcard-filter advanced --format jsonl -o output.jsonl

# Multi-NIC parallel sending
./ksubdomain v -f domains.txt --interface eth0 --interface eth1
```

### Enum Mode

Brute-force subdomains using a dictionary and optional prediction.

```
Additional Options (beyond verify):
  --filename, -f        Dictionary file (uses built-in wordlist by default)
  --domain-list, --ds   File containing root domains for batch enumeration
  --use-ns-records      Query domain NS records and add them as resolvers
```

```bash
# Enumerate with built-in wordlist
./ksubdomain e -d example.com

# Custom dictionary
./ksubdomain e -d example.com -f subdomains.txt -o result.txt

# Prediction + advanced wildcard filter
./ksubdomain e -d example.com --predict --wildcard-filter advanced

# Batch enumeration from file
./ksubdomain e --domain-list roots.txt -b 10m --format jsonl -o result.jsonl

# Pipe to httpx
./ksubdomain e -d example.com -s --od | httpx -silent
```

### Test / Device Commands

```bash
# Test max packet rate of current network adapter
./ksubdomain test

# List available network interfaces
./ksubdomain device
```

## ✨ Features

### Dynamic Timeout Adaptation
No need to hand-tune `--timeout`. The engine measures real RTT samples and computes an adaptive timeout using EWMA (α=0.125, β=0.25 per RFC 6298), bounded between 1 s and 10 s.

### Multi-NIC Parallel Sending
Repeat `--interface` to distribute sending across multiple adapters. Each adapter runs an independent `sendCycleForIface` goroutine competing on the shared `domainChan`.

```bash
./ksubdomain e -d example.com --interface eth0 --interface eth1 -b 20m
```

### Prediction Mode
With `--predict`, the tool uses discovered subdomains to predict related patterns, increasing coverage.

### Wildcard Filtering
- `none` — no filtering (default)
- `basic` — remove obvious wildcard IPs
- `advanced` — multi-dimensional wildcard detection

### Output Formats

| Format | Description | Best For |
|--------|-------------|----------|
| `txt` | `domain => record`, real-time | Human review |
| `json` | Full JSON, written on completion | Programmatic parsing |
| `csv` | CSV table, written on completion | Data analysis |
| `jsonl` | One JSON per line, real-time | Streaming / tool chaining |

## 🔗 Integration Examples

```bash
# Pipe to httpx
./ksubdomain e -d example.com -s --od | httpx -silent

# Pipe to nuclei
./ksubdomain e -d example.com -s --od | nuclei -l /dev/stdin

# JSONL streaming: extract A records only
./ksubdomain e -d example.com --format jsonl | jq -r 'select(.type=="A") | .domain'

# Batch + deduplicate
./ksubdomain e --domain-list roots.txt -s --od | sort -u > all_subs.txt
```

## 🌐 Platform Notes

**macOS** — BPF buffer is smaller; keep bandwidth conservative:
```bash
sudo ./ksubdomain e -d example.com -b 5m
```

**WSL/WSL2:**
```bash
./ksubdomain e -d example.com --interface eth0
```

**Windows** — Must install Npcap first; run as Administrator:
```bash
.\ksubdomain.exe enum -d example.com
```

## 🧩 Go SDK

```bash
go get github.com/boy-hack/ksubdomain/v2/sdk
```

```go
import "github.com/boy-hack/ksubdomain/v2/sdk"

scanner := sdk.NewScanner(&sdk.Config{
    Bandwidth:      "5m",
    Retry:          3,
    WildcardFilter: "advanced",
})

// Blocking: collect all results
results, err := scanner.Enum("example.com")

// Streaming: real-time callback
err = scanner.EnumStream(ctx, "example.com", func(r sdk.Result) {
    fmt.Printf("%s => %v\n", r.Domain, r.Records)
})
```

See [SDK README](./sdk/README.md) and [API Reference](./docs/api.md).

## 🧪 Testing

```bash
# Unit tests (no root needed)
go test ./pkg/...

# Regression tests (root + network required)
go build -o ksubdomain ./cmd/ksubdomain
sudo go test ./test/regression/... -tags regression -v -timeout 120s
```

## 📚 Documentation

- [Quick Start Guide](./docs/quickstart.md)
- [API Reference](./docs/api.md)
- [Best Practices](./docs/best-practices.md)
- [FAQ](./docs/faq.md)
- [SDK README](./sdk/README.md)

## 💡 References

- Original KSubdomain: [knownsec/ksubdomain](https://github.com/knownsec/ksubdomain)
- Stateless scanning theory: [paper.seebug.org/1052](https://paper.seebug.org/1052/)
- KSubdomain introduction: [paper.seebug.org/1325](https://paper.seebug.org/1325/)

## 📜 License

MIT License — see [LICENSE](LICENSE)
