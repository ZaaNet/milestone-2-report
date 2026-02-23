# ZaaNet Router Installation Guide

Guide for installing ZaaNet on a GL.iNet GL-XE300 (OpenWrt-based) router. The drive with the project is connected to the **admin laptop**; the laptop is connected to the router via **Ethernet**. You copy the project to the laptop, then copy `install/` and `proxy/` to the router, then run the installer on the router.

---

## Prerequisites

- **Router:** GL.iNet GL-XE300 (or compatible OpenWrt 22.03.x).
- **Admin laptop:** Connected to the router via Ethernet (e.g. 192.168.8.1).
- **ZaaNet project:** On a drive connected to the laptop, or already copied to the laptop (e.g. `~/Documents/zaanet-setup`).
- **Internet:** Recommended for the router (opkg); script can continue without it if packages are already present.

**Required layout:** After copying to the router, the source path (e.g. `/tmp`) must contain both `install/` and `proxy/`. Under `proxy/` the script expects `proxy/pages/` (splash.html, status.html, styles.css, images/zaanet-logo.png) and `proxy/@py-scripts/` (binauth.py, stats.flasky.py), or equivalent flat layout.

---

## Step 1: Copy project to the laptop (if needed)

If the ZaaNet project is on a USB drive, copy the whole project to the laptop (e.g. `~/Documents/zaanet-setup`). Ensure the folder contains the `install/` and `proxy/` directories.

---

## Step 2: Copy install and proxy to the router (from the laptop)

Connect the laptop to the router via Ethernet. From the **laptop** (not the router), run:

```bash
scp -O -r ~/Documents/zaanet-setup/install ~/Documents/zaanet-setup/proxy root@192.168.8.1:/tmp/
```

Use your actual project path instead of `~/Documents/zaanet-setup` if different. After this, the router has `/tmp/install/` and `/tmp/proxy/`.

---

## Step 3: Run the installer on the router

1. SSH into the router from the laptop:
   ```bash
   ssh root@192.168.8.1
   ```

2. (Recommended) Set a root password if the router warns none is set:
   ```bash
   passwd
   ```

3. Run the installer (use **sh** — OpenWrt has ash, not bash; use `/tmp` as the source):
   ```bash
   chmod +x /tmp/install/install-zaanet.sh
   sh /tmp/install/install-zaanet.sh /tmp
   ```

4. Follow the prompts. You will be asked for:
   - **Contract ID** (from ZaaNet platform)
   - **WiFi SSID** (default: ZNET-XXXX)
   - **Admin device whitelist** (optional; recommended so your SSH device keeps access during WiFi reload)

5. When prompted for WiFi configuration, ensure you are connected via **Ethernet or USB**, not WiFi. WiFi will be reconfigured to **2.4 GHz only, open (no password)**; the captive portal handles authentication after users connect.

---

## What the Script Does

- Copies and verifies required files from your source path
- Updates package lists and checks free space
- Installs Python 3, pip, flask, requests (for binauth and stats service)
- Installs nodogsplash (captive portal)
- Generates a **Router ID** (save this)
- Creates `/etc/zaanet/` and deploys config, splash pages, and Python scripts
- Configures nodogsplash (BinAuth only), firewall rules, and optional admin MAC whitelist
- Configures WiFi: 2.4 GHz only, open network; disables 5 GHz
- Enables and starts nodogsplash and the stats_flask service
- Writes an installation log to `/etc/zaanet/installation.log`

---

## After Installation

1. **Save your Router ID** – You need it to register the router on the ZaaNet platform.
2. **Register the router** – Use the Router ID and Contract ID on the ZaaNet platform.
3. **Test the captive portal** – Connect a device to the new WiFi SSID, open a browser, and confirm the ZaaNet splash page appears.
4. **Logs** – See the section *Viewing logs (Nodogsplash, BinAuth, Flask)* below for how to view Nodogsplash, BinAuth, and the stats Flask service logs.

**Useful commands:**

- Service status: `/etc/init.d/nodogsplash status`
- Restart captive portal: `/etc/init.d/nodogsplash restart`
- Installation log: `cat /etc/zaanet/installation.log`
- Connected clients: `ndsctl clients`

---

## Viewing logs (Nodogsplash, BinAuth, Flask)

After the setup is running, use these to inspect logs and debug.

### Nodogsplash (captive portal)

Nodogsplash logs to the system log. On the router:

```bash
# Recent Nodogsplash messages
logread | grep nodogsplash

# Follow live (new lines as they appear)
logread -f | grep nodogsplash
```

To see the full system log then filter locally:

```bash
logread
```

### BinAuth (voucher validation)

BinAuth writes to a file each time it is called (splash login attempts and backend validation):

```bash
# View entire log
cat /tmp/binauth.log

# Follow new entries (e.g. while testing voucher)
tail -f /tmp/binauth.log
```

Note: `/tmp` is cleared on reboot, so this log only exists until the next reboot.

### Stats (Flask) service

The stats service runs in the background and does not write to a log file by default. To see its output:

- Check that it is running:  
  `ps | grep stats.flasky`
- To see live logs, run it in the foreground (stop the service first):  
  ```bash
  /etc/init.d/stats_flask stop
  python3 /etc/zaanet/stats.flasky.py
  ```  
  Press Ctrl+C when done, then start the service again:  
  `/etc/init.d/stats_flask start`

---

## Stability features

- **stats_flask on boot:** The installer sets up a procd-based init script so the stats service starts automatically at boot and is restarted if it crashes.
- **WAN recovery and 4G failover:** If the router loses its WAN IP (no default route), a watchdog runs every 5 minutes: it first runs `ifup wan` (e.g. Starlink on eth1). If there is still no default route after 30s, it tries 4G failover by running `ifup wwan` (or `ifup mobile` / `ifup modem` if that interface exists in UCI). So users can get internet via 4G when the primary WAN is down. Log messages appear as `wan-watchdog` in `logread`. **For 4G failover to work**, the router must have a wwan (or mobile) interface configured (e.g. in the router’s web UI under Network / 4G / Mobile), so that `ifup wwan` can bring up the LTE connection.

---

## Troubleshooting

- **"not found" when running the install script** – OpenWrt uses ash, not bash. Run with **sh**: `sh /tmp/install/install-zaanet.sh /tmp`
- **"Failed to connect to ubus" / nodogsplash inactive:** Run `/etc/zaanet/start-nodogsplash.sh` (this also runs at boot via rc.local). Then check `ps | grep nodogsplash` and `ndsctl clients`.
- **WiFi name shows router hostname instead of your SSID:** Run `uci set system.@system[0].hostname='Zaanet-proxy'; uci commit system; wifi down; sleep 2; wifi up` (use your chosen SSID in place of Zaanet-proxy).
- **No splash page:** Check `/etc/init.d/nodogsplash status` and `logread | grep nodogsplash`. If init fails, use the nds-write-conf.sh fallback above; otherwise restart with `/etc/init.d/nodogsplash restart`.
- **Lost SSH after WiFi change:** Connect via Ethernet or USB; add your MAC to the whitelist:  
  `uci add_list nodogsplash.@nodogsplash[0].trustedmaclist='aa:bb:cc:dd:ee:ff'` then `uci commit nodogsplash` and `/etc/init.d/nodogsplash restart`.
- **Restore backups:** Config backups are listed at the end of the install and in `/etc/zaanet/installation.log`. Example:  
  `cp /etc/config/nodogsplash.backup.TIMESTAMP /etc/config/nodogsplash`

For more detail, see the script output and `INSTALL_ZAANET_AUDIT.md` in this directory.
