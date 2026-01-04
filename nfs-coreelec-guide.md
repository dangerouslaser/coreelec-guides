# Mounting an NFS Share from Unraid in CoreELEC

This guide walks you through setting up an NFS mount from your Unraid server to CoreELEC for smooth media playback.

---

## Step 1: Enable and Configure NFS on Unraid

1. In the Unraid web UI, go to **Settings → NFS**
2. Set **Enable NFS** to `Yes`
3. Set **Tunable (NFS async)** to `Yes` for better performance
4. Click **Apply**

### Export Your Share

1. Go to **Shares** and click on the share you want to export (e.g., `media`)
2. Scroll down to **NFS Security Settings**
3. Set **Export** to `Yes`
4. Set **Security** to `Public` (easiest) or configure specific host access
5. If using host-based security, add your CoreELEC device's IP (e.g., `192.168.1.100`)
6. Click **Apply**

---

## Step 2: Configure the Mount in CoreELEC

### Option A: Using the GUI (Easiest)

1. Go to **Settings → CoreELEC → Network**
2. Select **Mounts**
3. Choose **Add Network Location**
4. Fill in:
   - **Protocol:** NFS
   - **Server Address:** Your Unraid IP (e.g., `192.168.1.50`)
   - **Remote Path:** `/mnt/user/media` (adjust to your share name)
   - **Mount at Boot:** Yes
5. Save and reboot

### Option B: Using storage/.config/autostart.sh (More Reliable)

SSH into your CoreELEC box and create/edit the autostart script:

```bash
nano /storage/.config/autostart.sh
```

Add the following (adjust IPs and paths):

```bash
#!/bin/bash

# Wait for network to be fully up
sleep 10

# Create mount point
mkdir -p /storage/media

# Mount NFS share
mount -t nfs -o rw,soft,timeo=30,retrans=2,nolock 192.168.1.50:/mnt/user/media /storage/media
```

Make it executable:

```bash
chmod +x /storage/.config/autostart.sh
```

Reboot to test.

### Option C: Using systemd (Most Robust)

Create a systemd mount unit:

```bash
nano /storage/.config/system.d/storage-media.mount
```

Add:

```ini
[Unit]
Description=NFS Mount for Media
After=network-online.target
Wants=network-online.target

[Mount]
What=192.168.1.50:/mnt/user/media
Where=/storage/media
Type=nfs
Options=rw,soft,timeo=30,retrans=2,nolock

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
systemctl enable storage-media.mount
systemctl start storage-media.mount
```

---

## Step 3: Add the Path to Kodi

1. In Kodi, go to **Settings → Media → Library**
2. Select **Videos → Files → Add Videos**
3. Browse to your mount point (e.g., `/storage/media`)
4. Set the content type and scraper
5. Scan the library

---

## Recommended NFS Mount Options Explained

| Option | Purpose |
|--------|---------|
| `rw` | Read-write access |
| `soft` | Return error on timeout instead of hanging |
| `timeo=30` | Timeout in deciseconds (3 seconds) |
| `retrans=2` | Number of retries before soft error |
| `nolock` | Disable NFS locking (improves compatibility) |

For read-only media playback, you can use `ro` instead of `rw`.

---

## Troubleshooting

### "Permission denied" errors

- Ensure your Unraid NFS export is set to `Public` or includes your CoreELEC IP
- Check that the share permissions on Unraid allow read access

### Mount works but doesn't persist after reboot

- Use the systemd method (Option C) for the most reliable boot persistence
- Ensure the `sleep` in autostart.sh is long enough for your network to initialize

### Slow performance or stuttering

- Add `async` to mount options: `-o rw,soft,async,nolock`
- Ensure Unraid's **NFS async** tunable is enabled
- Consider using `nconnect=4` (multiple TCP connections) if your CoreELEC kernel supports it

### Can't see the share at all

Test from CoreELEC SSH:

```bash
# Check if NFS server is reachable
showmount -e 192.168.1.50

# Test manual mount
mount -t nfs 192.168.1.50:/mnt/user/media /storage/media
```

### "mount.nfs: Network is unreachable"

The network isn't ready when the mount is attempted. Increase the `sleep` value in autostart.sh or use the systemd method with proper `After=network-online.target` dependency.

---

## Quick Test Command

To quickly verify everything works before making it permanent:

```bash
mkdir -p /storage/media
mount -t nfs -o rw,soft,nolock 192.168.1.50:/mnt/user/media /storage/media
ls /storage/media
```

If you see your files, you're good to make it permanent using one of the methods above.
