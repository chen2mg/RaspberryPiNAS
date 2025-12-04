Build NAS server at home with raspberry Pi

---

# Raspberry Pi Docker NAS Setup

This setup runs **Samba (Windows)**, **MiniDLNA (TV)**, and **NFS (Cameras)** in Docker containers.

---

## Prerequisites

### 1. Mount the Drive

Ensure you have added the drive to `/etc/fstab` and mounted it to `/mnt/nas` on your Pi host system.

Find the new drive name (should be sda or sdb):

```bash
lsblk
```

Format it (⚠️ this erases everything):

```bash
sudo mkfs.ext4 /dev/sda1
```

If `/dev/sda1` doesn’t exist, create a partition first:

```bash
sudo fdisk /dev/sda
```

Find the drive's UUID.

Get it:

```bash
sudo blkid
# looking for something like /dev/sda1: UUID="54efc51a-274c-44d7-a0e2-50da701243b6" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="396c3357-01"
```

Edit fstab:

```bash
sudo vim /etc/fstab
```

Replace the old UUID with the new one:

```
UUID=NEW-UUID-HERE  /mnt/nas  ext4  defaults,auto,users,rw,nofail  0  0
```

### 2. Load Kernel Modules

The NFS container needs kernel support. Run this on your Pi:

```bash
sudo modprobe nfs
sudo modprobe nfsd
```

To make this persistent, add `nfs` and `nfsd` to `/etc/modules`.

---

## How to Start

1. Place `docker-compose.yml` in a folder (e.g., `~/nas`).
2. Run the app:

```bash
make up
```

or

```bash
docker compose up -d
```

**Auto-Restart:**
The `restart: unless-stopped` line in the YAML ensures these containers start automatically after a reboot.

---

## Connecting to Services

### 1. Windows PC (Samba)

* **Address:** `\\RaspberryPiNAS\storage` (or use your Pi's IP)
* **Username:** `pi`
* **Password:** `raspberry` (or what you set in `docker-compose.yml`)

Or set .env file
```
# .env file content
FTP_USERNAME=pi
FTP_PASSWORD=raspberry
```

create samba_config.yml
```
auth:
  - user: pi
    group: pi
    uid: 1000
    gid: 1000
    password: raspberry

global:
  - "force user = pi"
  - "force group = pi"
  # Optional: Fix for some Windows 11 discovery issues
  - "server min protocol = SMB2_10"

share:
  - name: storage
    path: /samba/storage
    browsable: yes
    readonly: no
    guestok: no
    validusers: pi
    writelist: pi
    veto: no
    recycle: yes
```

### 2. Smart TV (DLNA)

* Open your TV's **Input** / **Media** menu
* Look for **“RaspberryPi Media”**

### 3. Amcrest Camera (NFS)

Navigate:
**Camera Web UI → Setup → Storage → Destination → NAS**

* **Server Address:** Your Pi’s IP (e.g., `192.168.1.x`)
* **Remote Path:** `/data`

  * (`fsid=0` may cause paths like `/` or `/data` depending on NFS client version—start with `/data`)

---

## How to Upgrade Storage Drive

If you upgrade to a larger drive later, follow these steps to replace it safely.

---

### 1. Stop the Containers

```bash
cd ~/repo/dockerapps
make down
```

### 2. Unmount Old Drive

```bash
sudo umount /mnt/nas
```

Unplug the old drive and plug in the new one.

---

### 3. Prepare the New Drive

Find the new drive name (should be sda or sdb):

```bash
lsblk
```

Format it (⚠️ this erases everything):

```bash
sudo mkfs.ext4 /dev/sda1
```

If `/dev/sda1` doesn’t exist, create a partition first:

```bash
sudo fdisk /dev/sda
```

---

### 4. Update Auto-Mount (**Critical**)

The new drive has a new UUID.

Get it:

```bash
sudo blkid
# looking for something like /dev/sda1: UUID="54efc51a-274c-44d7-a0e2-50da701243b6" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="396c3357-01"
```

Edit fstab:

```bash
sudo vim /etc/fstab
```

Replace the old UUID with the new one:

```
UUID=NEW-UUID-HERE  /mnt/nas  ext4  defaults,auto,users,rw,nofail  0  0
```

Mount it:

```bash
sudo mount -a
systemctl daemon-reload
sudo chmod -R 777 /mnt/nas
```

Make dirs:

```bash
mkdir -p /mnt/nas/samba
mkdir -p /mnt/nas/media
sudo chmod -R 777 /mnt/nas
```

---

### 5. Restart Services

```bash
docker compose up -d
```

Your containers will now use the new, empty drive at the same path.

---

