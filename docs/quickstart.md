# Quick Start Guide

## Prerequisites

| Requirement | Notes |
|---|---|
| OS | Linux, macOS, Windows |
| Privileges | **root** or `CAP_NET_RAW` — raw packet capture requires elevated access |
| libpcap / npcap | Linux: `libpcap-dev`; macOS: built-in; Windows: [Npcap](https://npcap.com) |
| Go (build only) | 1.23+ |

---

## Installation

### Download pre-built binary (recommended)

Visit [Releases](https://github.com/boy-hack/ksubdomain/releases) and download the binary for your platform.

```bash
# Linux x86_64
curl -L https://github.com/boy-hack/ksubdomain/releases/latest/download/ksubdomain_linux_amd64 \
     -o /usr/local/bin/ksubdomain
chmod +x /usr/local/bin/ksubdomain
```

### Build from source

```bash
git clone https://github.com/boy-hack/ksubdomain.git
cd ksubdomain
# Inject version via ldflags (recommended)
go build -ldflags "-X github.com/boy-hack/ksubdomain/v2/pkg/core/conf.Version=v2.x.y" \
    -o ksubdomain ./cmd/ksubdomain
```

---

## Your first scan

### 1 — Check available network interfaces

```bash
sudo ./ksubdomain device
```

### 2 — Test maximum packet rate

```bash
sudo ./ksubdomain test
```

### 3 — Enumerate subdomains (built-in wordlist)

```bash
sudo ./ksubdomain enum -d example.com
```

Sample output:

```
www.example.com => 93.184.216.34
mail.example.com => 93.184.216.50
api.example.com => 93.184.216.51
```

### 4 — Verify a domain list

```bash
sudo ./ksubdomain verify -f domains.txt -o output.txt
```

---

## Common workflows

### Pipe to httpx for HTTP probing

```bash
sudo ./ksubdomain enum -d example.com --only-domain --silent | httpx -silent
```

### Save as JSONL for streaming processing

```bash
sudo ./ksubdomain enum -d example.com --format jsonl -o result.jsonl
```

### Batch enumeration of multiple root domains

```bash
sudo ./ksubdomain enum --domain-list roots.txt -b 10m --format jsonl -o all.jsonl
```

### Predict + advanced wildcard filter

```bash
sudo ./ksubdomain enum -d example.com --predict --wildcard-filter advanced -o result.txt
```

### Multi-NIC parallel sending

```bash
sudo ./ksubdomain enum -d example.com --interface eth0 --interface eth1 -b 20m
```

---

## Platform notes

### Linux

```bash
# Grant CAP_NET_RAW instead of running as root (optional)
sudo setcap cap_net_raw+ep ./ksubdomain

./ksubdomain enum -d example.com
```

### macOS

macOS uses BPF with smaller default buffers. Keep bandwidth conservative:

```bash
sudo ./ksubdomain enum -d example.com -b 5m
```

### WSL / WSL2

```bash
./ksubdomain enum -d example.com --interface eth0
```

### Windows

1. Install [Npcap](https://npcap.com/) (WinPcap is not supported)
2. Run as Administrator

```bat
.\ksubdomain.exe enum -d example.com
```

---

## Next steps

- [API Reference](./api.md)
- [Best Practices](./best-practices.md)
- [FAQ](./faq.md)
- [SDK README](../sdk/README.md)
