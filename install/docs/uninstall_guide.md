# ZaaNet Router Uninstall Guide

Short guide for removing ZaaNet and nodogsplash from a router. The **admin laptop** is connected to the router via **Ethernet**. You copy the `install/` folder to the router (if not already there), then run the uninstall script on the router with `sh` (OpenWrt uses ash, not bash).

---

## Prerequisites

- **Access:** SSH as root from the laptop (e.g. `ssh root@192.168.8.1`).
- **Uninstall script on router:** Either already present from a previous install copy, or copy `install/` from the laptop to the router (see below).

---

## What the Uninstall Does

By default the script:

1. **stats_flask** – Stops and disables the service, removes `/etc/init.d/stats_flask` and the pidfile.
2. **/etc/zaanet** – Removes the directory (config, binauth.py, stats.flasky.py, installation.log).
3. **Nodogsplash** – Stops and disables the service, uninstalls the package (`opkg remove nodogsplash`), and deletes `/etc/config/nodogsplash` and the `/etc/nodogsplash` directory.
4. **Wireless** – Restores `/etc/config/wireless` from the backup created during install (if found).
5. **Packages** – Uninstalls ZaaNet-related packages: flask, requests (pip3), then python3-pip and python3 (opkg).

---

## Step 1: Copy install folder to the router (from the laptop, if needed)

If the router does not already have the install folder (e.g. after a reboot, `/tmp` is cleared), from the **laptop** run:

```bash
scp -O -r ~/Documents/zaanet-setup/install root@192.168.8.1:/tmp/
```

Use your actual project path if different.

---

## Step 2: Run the uninstall on the router

1. SSH into the router from the laptop:
   ```bash
   ssh root@192.168.8.1
   ```

2. Run the uninstall script with **sh** (OpenWrt has ash, not bash; running with `sh` avoids "not found"):
   ```bash
   sh /tmp/install/uninstall-zaanet.sh
   ```
   Do not pass a path argument; the script reads backup paths from `/etc/zaanet/installation.log` before removing it.

3. When prompted, confirm with `y` to continue.

4. After it finishes, if wireless was restored, run `wifi reload` so the previous WiFi settings take effect (SSID/password may change).

---

## Options

| Option | Description |
|--------|-------------|
| `--keep-wireless` | Do not restore wireless config; leave current WiFi settings as they are. |
| `--keep-packages` | Do not remove nodogsplash, python3, python3-pip, flask, or requests. Only removes services and files. |
| `-h`, `--help` | Show usage and options. |

**Examples (run on the router after SSH):**

```bash
# Full uninstall (default)
sh /tmp/install/uninstall-zaanet.sh

# Remove ZaaNet but keep current wireless and all packages
sh /tmp/install/uninstall-zaanet.sh --keep-wireless --keep-packages

# Remove everything but do not restore wireless
sh /tmp/install/uninstall-zaanet.sh --keep-wireless
```

---

## Wireless Backup

- The install script backs up wireless config to `/etc/config/wireless.backup.TIMESTAMP`.
- The uninstall reads the backup path from `/etc/zaanet/installation.log` if it exists; otherwise it uses the most recent `wireless.backup.*` in `/etc/config/`.
- If no backup is found, wireless is not changed. You can restore manually if you know the backup path:
  ```bash
  cp /etc/config/wireless.backup.YYYYMMDD-HHMMSS /etc/config/wireless
  wifi reload
  ```

---

## After Uninstall

- Nodogsplash and ZaaNet are fully removed. The router no longer runs a captive portal or ZaaNet services.
- If wireless was restored, WiFi will match the state from before the ZaaNet install (run `wifi reload` if needed).
- Backup files (e.g. `wireless.backup.*`, `nodogsplash.backup.*`) are left on the router; delete them manually if you do not need them.

---

## Troubleshooting

- **"not found" when running the script** – OpenWrt uses ash, not bash. Run with `sh`: `sh /tmp/install/uninstall-zaanet.sh` (do not pass a path argument).
- **"This script must be run as root"** – Use `ssh root@192.168.8.1` and run the script again.
- **Wireless not restored** – Install may not have created a backup (e.g. WiFi step was skipped). Use `--keep-wireless` and configure WiFi manually, or restore from a known backup path.
- **opkg remove fails** – Ensure you have a working connection if opkg needs to resolve dependencies. You can still remove files and services; use `--keep-packages` to skip package removal.
- **Python or nodogsplash still present** – Run without `--keep-packages`, or remove manually: `opkg remove nodogsplash python3-pip python3`.
