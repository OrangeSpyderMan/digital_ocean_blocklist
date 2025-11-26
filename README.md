# CrowdSecurity Blocklist for Digital Ocean

Simple script to download the list of CIDR that [Digital Ocean](https://www.digitalocean.com/) publishes ([here](https://digitalocean.com/geo/google.csv)) and convert them to a CSV list to re-import using the `cscli decisions import` command in order to block all traffic from Digital Ocean.

NOTE : My experience is the list is not quite complete. It contains a massive range of CIDR, and blocked DO traffic on my servers has dropped considerably, but I do still see the occasional alert for an IP that does map back to DO. Not sure how often its updated - I can't believe it's really in their interest to do it too often as my internet searching whilst looking for a pre-existing blocklist suggests that almost everyone who's looking for an exhaustive list of DO CIDRs is doing it for the same reason I am :D

## Why bother?

- There is no valid use case for Digital Ocean traffic to access services I publish
- it is an absolute cesspit of all sorts of malicious traffic.
- They seem entirely uninterested in making any sort of sustainable change to prevent/reduce the number of bad actors using their service(s)

I had crowdsec protecting individual services with specific scenarios, but the amount of IPs that were from DO meant that I'd just save myself the bother and block them all.

## How does it work

This repo contains a small script, `update_DO_blocklist`, that downloads DigitalOcean's published CIDR list and converts it to a CSV suitable for importing into CrowdSec via `cscli decisions import`.

### What the script does (current version)

- Downloads the CSV from the configured `URL` (default: https://digitalocean.com/geo/google.csv).
- Converts each CIDR into a CrowdSec decision CSV line with columns `duration,scope,value,reason`.
- Imports decisions into CrowdSec using `cscli decisions import` (unless run in dry-run).

### Important runtime notes

- The script detects whether it's running as root and uses `sudo` when required.
- Temporary downloads are created in the same directory as the target output using `mktemp` with pattern `_<basename>.tmp.XXXXXX` to avoid races.
- By default the generated CSV is removed after import unless `--keep-file` is used.
- `--dry-run` prints the exact `cscli` commands that would be executed (prefixed with `[DRY-RUN] Would run:`) and prevents those commands from running; note that the current `--dry-run` still performs the network download and writes the CSV file.

### Usage

```
bash update_DO_blocklist [OPTIONS] {download|append|sync|refresh}
```

#### Options

- `-u URL` override the blocklist URL (default in script)
- `-d DURATION` override duration (default `876000h`) — use `h` for hours
- `-r REASON` override reason/scenario name (default `DigitalOcean`)
- `-n`, `--dry-run` show commands that would run, do not execute `cscli` operations
- `-k`, `--keep-file` keep the generated CSV after import
- `-f NAME`, `--filename NAME` set the output CSV filename (default `crowdsec_blocklist.csv`)
- `-h`, `--help` show help

#### Actions

- `download` — download and format the CSV only (no import)
- `append` — download and import (adds to existing bans)
- `sync` / `refresh` — download, delete decisions matching the configured reason, then import (overwrite method)

#### Examples

- Download only:
  `bash update_DO_blocklist download`
- Dry-run append (show commands, no cscli calls):
  `bash update_DO_blocklist --dry-run append`
- Append and keep file:
  `bash update_DO_blocklist -k -f /tmp/do.csv append`



## TODO

- Option to skip network download when doing a dry-run (`--no-download`)
- Optional `--create-dir` to auto-create target directories
- Add support for other providers' lists

If you'd like any of the TODOs implemented, open an issue or request a change.
