# CrowdSecurity Blocklist for DigitalOcean

This repository contains `update_DO_blocklist`, a small script that downloads DigitalOcean's published CIDR list and converts it to a CSV suitable for importing into CrowdSec using `cscli decisions import`.

## Why use this

- Block broad ranges of unwanted traffic originating from DigitalOcean when that provider has no legitimate reason to reach your services.

## How it works

The script downloads DigitalOcean's published CIDR list (the default source is the DigitalOcean geolocation CSV) and converts each CIDR into a CrowdSec decision CSV line with columns `duration,scope,value,reason`. It can then import the resulting CSV into CrowdSec using `cscli decisions import`.

### Important runtime notes

- The script detects whether it's running as root and will prefix `cscli` commands with `sudo` when required. This can be overridden with the `--no-sudo` and `--sudo-command` options.
- Temporary downloads are created in the same directory as the target output using `mktemp` with pattern `_<basename>.tmp.XXXXXX` to avoid race conditions.
- By default the generated CSV is removed after import unless `--keep-file` is used.
- `--dry-run` prints the exact `cscli` commands that would be executed (prefixed with `[DRY-RUN] Would run:`) and prevents those commands from running; note that the current `--dry-run` still performs the network download and writes the CSV file.

## Commands the script runs

When performing import or flush operations the script invokes `cscli` with these commands (the script will prefix with `sudo` automatically if not running as root, unless `--no-sudo` is used):

```bash
cscli decisions delete --scenario "<REASON>"
```

```bash
cscli decisions import -i "<FILE>"
```

If the script is not run as root, it will run these commands as `sudo cscli ...`. You can use `--sudo-command` to specify an alternative to `sudo` (like `doas`).

## Verbose and dry-run behavior

- `--dry-run` (`-n`): The script prints the exact command line it would execute (including `sudo` when applicable) and does not run the `cscli` commands. Example dry-run line:

  ```bash
  [DRY-RUN] Would run: sudo cscli decisions import -i crowdsec_blocklist.csv
  ```

- `--verbose` (`-v`): Adds detailed output describing intermediate steps and enrichment performed, for example:

  ```bash
  Downloading and processing list...
  [VERBOSE] Downloaded to: ./_crowdsec_blocklist.csv.tmp.hYvsZf
  [VERBOSE] Downloaded lines: 1191
  [VERBOSE] Enrichment: adding header 'duration,scope,value,reason' and mapping each CIDR -> 'duration,range,<cidr>,<reason>'
  [VERBOSE] Wrote CSV to: crowdsec_blocklist.csv (decisions: 1190)
  Importing new list to CrowdSec...
  [DRY-RUN] Would run: sudo cscli decisions import -i crowdsec_blocklist.csv
  [DRY-RUN] No changes made
  ```

## Usage

```bash
bash update_DO_blocklist [OPTIONS] {download|append|sync|refresh}
```

### Options

- `-u URL` override the blocklist URL (default in script)
- `-d DURATION` override duration (default `876000h`) — use `h` for hours
- `-r REASON` override reason/scenario name (default `DigitalOcean`)
- `-n`, `--dry-run` show commands that would run, do not execute `cscli` operations
- `-k`, `--keep-file` keep the generated CSV after import
- `-f NAME`, `--filename NAME` set the output CSV filename (default `crowdsec_blocklist.csv`)
- `--no-sudo` do not use `sudo` even if not running as root
- `--sudo-command CMD` use a different command for privilege escalation (e.g. `doas`)
- `-v`, `--verbose` enable verbose output
- `-h`, `--help` show help

### Actions

- `download` — download and format the CSV only (no import)
- `append` — download and import (adds to existing bans)
- `sync` / `refresh` — download, delete decisions matching the configured reason, then import (overwrite method)

### Examples

- Download only:

  ```bash
  bash update_DO_blocklist download
  ```

- Dry-run append (show commands, no cscli calls):

  ```bash
  bash update_DO_blocklist --dry-run append
  ```

- Append and keep file:

  ```bash
  bash update_DO_blocklist -k -f /tmp/do.csv append
  ```

## Systemd timer

This repository includes systemd unit files in the `systemd/` directory to run the script on a daily schedule:

- `systemd/update_do_blocklist.service` — a oneshot service that runs the script
- `systemd/update_do_blocklist.timer` — a timer that triggers the service daily

### Permissions and User

By default, the `update_do_blocklist.service` is configured to run as the `nobody` user and `nogroup` group for better security.
The `ExecStart` command in the service file writes the blocklist to `/var/tmp/do_blocklist.csv`. The `/var/tmp` directory is typically world-writable, so this should work out of the box on most systems.

If you change the output path in the service file, you **must** ensure that the `nobody` user has write permissions to that directory.

For enhanced security, it is recommended to create a dedicated user and group for this service and assign ownership of the script and the output directory to that user.

### Install the systemd units

A convenience script `systemd/systemd_install` is provided to install and enable the systemd timer. The script will use `sudo` if you are not running as root.

```bash
# cd into the systemd directory
cd systemd/
# Make the script executable
chmod +x systemd_install
# Run the installer
./systemd_install
```

You can also run it with the `-v` or `--verbose` flag to see the commands being executed.

```bash
./systemd_install -v
```

### Manual Installation (alternative)

If you prefer to install the units manually:

```bash
sudo cp systemd/update_do_blocklist.service /etc/systemd/system/
sudo cp systemd/update_do_blocklist.timer /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now update_do_blocklist.timer
```

### Verify the timer and view logs

```bash
sudo systemctl status update_do_blocklist.timer
sudo systemctl list-timers --all | grep update_do_blocklist
sudo journalctl -u update_do_blocklist.service -f
```

### Notes

- The service runs the script as a non-root user by default (see `User=` in the service file).
- The example service keeps a CSV at `/var/tmp/do_blocklist.csv` and runs `sync`. Edit the `ExecStart` in `systemd/update_do_blocklist.service` if you want different flags or a different output path.
- The service writes output to the system journal; if you prefer a file, either change the unit to redirect output or use a small wrapper script that handles logging and rotation.

### Test before enabling

Always test manually before enabling the timer. A good dry-run command is:

```bash
bash /home/pdd/src/digital_ocean_blocklist/update_DO_blocklist --dry-run --verbose append
```

To run a one-off real sync (careful — this imports decisions):

```bash
sudo bash /home/pdd/src/digital_ocean_blocklist/update_DO_blocklist -k -f /var/tmp/do_blocklist.csv sync
```

## TODO

- Option to skip network download when doing a dry-run (`--no-download`)
- Optional `--create-dir` to auto-create target directories
- Add support for other providers' lists

If you'd like any of the TODOs implemented, open an issue or request a change.
