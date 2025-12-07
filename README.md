# 🎬 Plex Media Server Setup Guide for Proxmox (LXC Container)

This guide provides a detailed, step-by-step process for setting up a high-performance Plex Media Server in a Proxmox LXC container, including critical steps for GPU hardware acceleration (Quick Sync/NVENC).

## 🚀 Quick Start (Basic Setup - No GPU Passthrough)

If you only need a basic Plex server without GPU hardware acceleration:

1.  Create the container with the recommended **Container Specifications**.
2.  Skip **STEP 0** (GPU passthrough configuration).
3.  Follow **STEPS 1–6, 9, 11–13**.
4.  Access via `http://YOUR_IP:32400/web`.

***
👉 ***PRO TIP**: For a secure, domain-based setup using **HTTPS** (plex.yourdomain.com), follow the [NGINX Proxy Manager Setup Guide for Proxmox (LXC Container)](https://github.com/zombiehunternr1/NGINX-Proxy-Manager-Setup-Guide) first, and then proceed with this guide.*
***

---

## ⚙️ Container Specifications

| Component | Recommendation | Notes |
| :--- | :--- | :--- |
| **General** | Hostname | Call it however you see fit (e.g. Plex-Server). |
| **General** | ✓ Enable Unpriviliged | Recommended. |
| **General** | ✓ Enable Nesting | Required. |
| **Template** | Debian 12 | Recommended. |
| **Disk** | 4 GB | Recommended for OS (media storage will be separate). |
| **CPU** | 4 cores | Start with 4, adjust to 6 if performance is slow. |
| **Memory** | 4 GB (4096MB) | Recommended. |
| **Network** | `vmbr0` | Use a static IP or DHCP reservation. (e.g. 192.168.20.x/24) |
| **DNS** | 8.8.8.8 or 1.1.1.1 | Recommended. |
| **FUSE** | ✓ Enable FUSE | Required. |

❗ Enabling FUSE (Filesystem in Userspace)
The FUSE feature can be set after the container has been created:
 1. Click on the container in the Proxmox GUI.
 2. Navigate to the **Options** panel.
 3. Select **"Features"** and click on **"Edit"** at the top.
 4. Inside the popup window, check the **FUSE** box.

### Storage Strategy

* **Default (Simple):** Single mounted storage for media, metadata, and transcode.
* **Recommended for Large Libraries:** Metadata on SSD, media on HDD.

---

## 🛠️ STEP 0 - Initial Container Setup (Proxmox Host)

**Perform all actions in this step from the Proxmox host console.**

### 1. Configure GRUB for IOMMU (GPU Passthrough)

Edit your GRUB configuration: `nano /etc/default/grub`

**Find and REPLACE this line:**
`GRUB_CMDLINE_LINUX_DEFAULT="quiet"`

**With ONE of these options (Choose based on your CPU):**

| CPU Type | Configuration Line | Notes |
| :--- | :--- | :--- |
| **Intel CPU** (Most Common) | `GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt"` | Use for Intel iGPU or dedicated GPU. |
| **NVIDIA GPU** | `GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt video=efifb:off"` | Use if you're using an RTX-series card |
| **AMD CPU** | `#GRUB_CMDLINE_LINUX_DEFAULT="quiet amd_iommu=on iommu=pt"` | Use for AMD iGPU and AMD GPU. |

Apply changes:

Ctrl + o → Enter → Ctrl + x

Update the grub:
```bash
update-grub
```

### 2. Add Kernel Modules

Edit the modules file: 
````bash
nano /etc/modules-load.d/vfio-pci.conf
````
Add these lines to the end:
````bash
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
kvmgt
````
Apply changes:

Ctrl + o → Enter → Ctrl + x

Update the initramfs:
````bash
update-initramfs -u -k all
````
After this is done reboot the system so it can start up with the right configuration:
```bash
reboot
```

### 3. Install NVIDIA Drivers on Proxmox Host (NVIDIA Only)
This is **ONLY** needed if you plan to use an NVIDIA dedicated GPU. **SKIP** if using only Intel or AMD iGPU.

Add the ```non-free``` repository: ```nano /etc/apt/sources.list```

Append ```non-free``` to your Debian repository lines.

```bash
apt update
apt install -y nvidia-driver nvidia-smi
```
```bash
# Blacklist nouveau driver
echo "blacklist nouveau" >> /etc/modprobe.d/blacklist.conf
echo "options nouveau modeset=0" >> /etc/modprobe.d/blacklist.conf
```

### 4. Configure LXC Container for GPU Passthrough
Edit the container configuration file: 
```bash
nano /etc/pve/lxc/YOUR_CONTAINER_ID.conf
```

**IMPORTANT**: Replace ```YOUR_CONTAINER_ID``` with your actual container ID (e.g., 100).

Add **ONE** of the following configurations to the end of the file:

🟢 OPTION A: INTEL / AMD iGPU ONLY

- **Best for:** Intel CPUs with Quick Sync (6th gen+) or AMD iGPUs.
- **Use if:** You only have integrated graphics or want the simplest setup.

````bash
lxc.cgroup2.devices.allow: c 226:* rwm
lxc.cgroup2.devices.allow: c 29:* rwm
lxc.mount.entry: /dev/dri dev/dri none bind,optional,create=dir
lxc.mount.entry: /dev/fb0 dev/fb0 none bind,optional,create=file
````
🟡 OPTION B: NVIDIA GPU ONLY

- **Best for:** Dedicated NVIDIA GPU (GTX 1050+, RTX, Quadro).
- **Note:** Requires **Step 3** (NVIDIA drivers on Proxmox host).

````bash
lxc.cgroup2.devices.allow: c 195:* rwm
lxc.cgroup2.devices.allow: c 226:* rwm
lxc.mount.entry: /dev/nvidia0 dev/nvidia0 none bind,optional,create=file
lxc.mount.entry: /dev/nvidiactl dev/nvidiactl none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-uvm dev/nvidia-uvm none bind,optional,create=file
lxc.mount.entry: /dev/dri dev/dri none bind,optional,create=dir
````
🔴 OPTION C: BOTH INTEL iGPU AND NVIDIA GPU

- **Best for:** Intel CPU + NVIDIA GPU combo systems.
- **Note:** Requires **Step 3** (NVIDIA drivers on Proxmox host).
````bash
# Intel iGPU
lxc.cgroup2.devices.allow: c 226:* rwm 
lxc.cgroup2.devices.allow: c 29:* rwm 

# NVIDIA GPU
lxc.cgroup2.devices.allow: c 195:* rwm 

# Mount points (both GPUs)
lxc.mount.entry: /dev/dri dev/dri none bind,optional,create=dir
lxc.mount.entry: /dev/fb0 dev/fb0 none bind,optional,create=file
lxc.mount.entry: /dev/nvidia0 dev/nvidia0 none bind,optional,create=file
lxc.mount.entry: /dev/nvidiactl dev/nvidiactl none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-uvm dev/nvidia-uvm none bind,optional,create=file
````
Save and the configuration:

Ctrl + o → Enter → Ctrl + x

Restart the container if it's already running, otherwise you can **skip** this step
````bash
pct stop YOUR_CONTAINER_ID
pct start YOUR_CONTAINER_ID
````

## 🖥️ STEP 1: Update and Configure/Check Network Settings for the Container ##
Perform these actions from inside the Plex container console.

Update system and install essential packages
```bash
apt update && apt upgrade -y
```
Install OpenSSH and GNUPG
```bash
apt install ufw sudo curl nano openssh-server wget gnupg -y
```
Enable SSH
```bash
systemctl enable ssh --now
```
## ⚙️ STEP 2 - Set Static IP ##

Check your interface and current IP
```bash
ip a
```
Edit network configuration
```bash
nano /etc/network/interfaces
````
Replace the content with the following:
```bash
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
    address YOUR_IP_ADDRESS/24  # /24 = 255.255.255.0
    gateway YOUR_GATEWAY
    dns-nameservers 8.8.8.8 1.1.1.1
```
Ctrl + o → Enter → Ctrl + x

Restart networking
```bash
systemctl restart networking
```
## 🔒 STEP 3 - Secure the Container with UFW

This step applies firewall rules to the LXC container to restrict access.

### Port Logic & Assumptions

| Component | Example Network | Notes |
| :--- | :--- | :--- |
| **Trusted Network** (Your PC/LAN) | `192.168.10.x` | Adjust this to your local network segment (e.g., `192.168.1.0/24`). |
| **Server Network** (Plex Container Subnet) | `192.168.20.x` | Adjust this to the network where your Plex LXC lives. |
| **YOUR PLEX PORT** | `32400` or `32402` | Use **32400** for your main server. Use **32402**, **32403**, etc., for **additional** Plex servers to match your `socat` configuration (Step 7). |

> ❗ **Important Note:** Use the correct port for your server. If this is your **first/main** Plex server, use port **32400**. If this is an **additional** server on your network, use the corresponding `socat` proxy port (e.g., `32402`).

**SSH:** Allow only from your trusted network (where your PC is)
```bash
ufw allow from 192.168.10.0/24 to any port 22 comment 'SSH from Trusted Network' 
```
**Plex:** Allow from your trusted network (to access from your PC)
```bash
ufw allow from 192.168.10.0/24 to any port YOUR_PLEX_PORT comment 'Plex from Trusted Network' 
```
**Plex:** Allow from server network (where Plex container runs)
```bash
ufw allow from 192.168.20.0/24 to any port YOUR_PLEX_PORT comment 'Plex from Server Network' 
```
Enable UFW and check if the firewall rules are implemented correctly
```bash
ufw enable
ufw status
```
### Removing Rules (In case of a mistake)

If you need to undo a rule, use `ufw delete` followed by the exact rule you want to remove.

```bash
# Example: Removes the rule allowing port 22 from the trusted network
ufw delete allow from 192.168.10.0/24 to any port 22
```

## 👤 STEP 4 - Create a Non-Root User ##
Create admin user for managing media files
````bash 
adduser plexadmin
usermod -aG sudo plexadmin
````
You get asked to fill in the following fields:
 - Full Name
 - Room Number
 - Work Phone
 - Home Phone
 - Other

You don't need to fill these in and just press **Enter** to continue.

When confirming if the information type '**Y**' and hit **Enter**.

## 💿 STEP 5 - Install Plex Media Server and Check GPU Access ##

Download latest Plex Media Server

Visit: [https://www.plex.tv/media-server-downloads/](https://www.plex.tv/media-server-downloads/).

In this guide we're working with **Intel**.

Copy the link for the **Ubuntu (16.04+) / Debian (8+) - Intel/AMD 64-bit for Linux** package, and paste it into the command below.

```bash
wget PASTE_LINK_HERE
```
Unpack the Plex media server package
```bash
dpkg -i plexmediaserver_*.deb
```
Enable en start the Plex media server
```bash
systemctl enable plexmediaserver
systemctl start plexmediaserver
```
Optional: Clean up Installation File

The ```.deb``` file is no longer needed after the installation is complete and can be safely removed.
```bash
rm plexmediaserver_*.deb
```

Check if the GPU is visible in the container:
```bash
echo "=== Checking GPU Access ==="
ls /dev/dri 2>/dev/null && echo "✓ Intel/AMD iGPU is accessible" || echo "✗ Intel/AMD iGPU NOT accessible"
ls /dev/nvidia* 2>/dev/null && echo "✓ NVIDIA GPU is accessible" || echo "✗ NVIDIA GPU NOT accessible"
echo ""
```

## 🚀 STEP 5.1 - Install GPU Drivers (CHOOSE ONE OPTION) ##
Choose the driver installation option that matches your hardware configuration from STEP 0.

OPTION 1: Intel CPU with Quick Sync (iGPU)
````bash
echo "=== Installing Intel Quick Sync Drivers ==="
apt install -y intel-media-va-driver vainfo intel-gpu-tools
usermod -aG video plex
usermod -aG render plex
sudo vainfo 2>/dev/null | head -5
echo "✓ Intel Quick Sync drivers installed"
````
OPTION 2: NVIDIA GPU Only
````bash
echo "=== Installing NVIDIA GPU Drivers ==="
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -fsSL [https://nvidia.github.io/libnvidia-container/gpgkey](https://nvidia.github.io/libnvidia-container/gpgkey) | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L [https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.list](https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.list) | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

apt update
apt install -y nvidia-container-toolkit nvidia-driver nvidia-smi
sudo groupadd nvidia 2>/dev/null || true
sudo usermod -aG nvidia plex
sudo usermod -aG video plex
nvidia-smi --query-gpu=name,driver_version --format=csv
echo "✓ NVIDIA drivers installed"

# Optional: Remove 3-stream limit (Highly Recommended)
echo ""
echo "OPTIONAL: Remove NVIDIA's 3-stream limit:"
echo "wget [https://raw.githubusercontent.com/keylase/nvidia-patch/master/patch.sh](https://raw.githubusercontent.com/keylase/nvidia-patch/master/patch.sh)"
echo "chmod +x patch.sh"
echo "sudo ./patch.sh"
````
OPTION 3: AMD CPU with iGPU
````bash
echo "=== Installing AMD GPU Drivers ==="
apt install -y mesa-va-drivers vainfo libva-utils
apt install -y rocm-opencl-runtime
usermod -aG video plex
usermod -aG render plex
usermod -aG kvm plex
sudo vainfo 2>/dev/null | head -5
echo "✓ AMD GPU drivers installed"
````
OPTION 4: Intel CPU + NVIDIA GPU COMBO
````bash
echo "=== Installing BOTH Intel & NVIDIA GPU Drivers ==="
apt install -y intel-media-va-driver-non-free vainfo intel-gpu-tools
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -fsSL [https://nvidia.github.io/libnvidia-container/gpgkey](https://nvidia.github.io/libnvidia-container/gpgkey) | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L [https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.list](https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.list) | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

apt update
apt install -y nvidia-container-toolkit nvidia-driver nvidia-smi
sudo usermod -aG video plex
sudo usermod -aG render plex
sudo groupadd nvidia 2>/dev/null || true
sudo usermod -aG nvidia plex
echo "✓ BOTH Intel and NVIDIA drivers installed"
````
Restart Plex after driver installation:
````bash
systemctl restart plexmediaserver
````
## 🗄️ STEP 6 - Prepare Dedicated Storage Volume ##
1. On Proxmox Host

Open the **Shell** on your Proxmox host and create the dedicated directory. 

The `chown` command is critical for granting the LXC container ownership of the volume.
````bash
mkdir -p /srv/plexdata
chown 100000:100000 /srv/plexdata
````
2. In Proxmox Web Interface
Go to your Plex **container → Resources → Add → Mount Point**.
- Storage: local-lvm
- Disk Size: [Your preferred size, e.g., 4TB]
- Path: /mnt/plexdata
- Backup: uncheck

3. Back in Plex Container Console
````bash
systemctl stop plexmediaserver
````
Choose Storage Configuration:

OPTION A: SINGLE DRIVE SETUP
````bash
mkdir -p /mnt/plexdata
mkdir -p /mnt/plexdata/movies
mkdir -p /mnt/plexdata/tvshows
mkdir -p /mnt/plexdata/transcode
mkdir -p /mnt/plexdata/metadata
chown -R plex:plex /mnt/plexdata
chmod -R 775 /mnt/plexdata
````
OPTION B: SEPARATE METADATA LOCATION (When running Proxmox on a SSD and your media storage on an HDD)
````bash
mkdir -p /opt/plex/metadata
mkdir -p /opt/plex/transcode
mkdir -p /mnt/plexdata/movies
mkdir -p /mnt/plexdata/tvshows
chown -R plex:plex /opt/plex
chown -R plex:plex /mnt/plexdata
chmod -R 775 /opt/plex
chmod -R 775 /mnt/plexdata
````
** ⚠️ IMPORTANT WARNING:** Ignore 'Permission denied' 

You may see the following denied messages regarding the system-created */mnt/plexdata/lost+found directory*. 

These are normal and expected because only the root user can modify this directory. 

You can safely ignore these messages and continue:
```bash
chown: cannot read directory '/mnt/plexdata/lost+found': Permission denied
chmod: changing permissions of '/mnt/plexdata/lost+found': Operation not permitted
chmod: cannot read directory '/mnt/plexdata/lost+found': Permission denied
```

Continue with User and Symlink:

Add plexadmin to plex group for file management
```bash
usermod -aG plex plexadmin
```
Create convenience symlink for plexadmin
```bash
sudo -u plexadmin ln -s /mnt/plexdata /home/plexadmin/plexdata
```
Restart Plex
```bash
systemctl start plexmediaserver
```
Create RAM Disk for Transcoding (Best Performance):
````bash
echo "Creating RAM disk for optimal transcoding performance..."
mkdir -p /tmp/plex-transcode
chown plex:plex /tmp/plex-transcode
chmod 775 /tmp/plex-transcode
echo "tmpfs /tmp/plex-transcode tmpfs defaults,size=2G 0 0" | tee -a /etc/fstab 
mount /tmp/plex-transcode
df -h /tmp/plex-transcode
echo "✓ RAM disk created at /tmp/plex-transcode (2GB)"

systemctl restart plexmediaserver
````
## 🔌 STEP 7 - Setup Socat Proxy for Port Translation (OPTIONAL) ##
**ONLY NEEDED IF:**
- Plex is already running on port 32400 elsewhere
- You need a different external port (e.g., 32402).

Install socat
```bash
apt install socat -y
```
Create socat proxy service to map external port 32402 to internal Plex on 32400
```bash
cat > /etc/systemd/system/plex-proxy.service << 'EOF'
[Unit]
Description=Plex Port Proxy (32402 to 32400)
After=plexmediaserver.service
Requires=plexmediaserver.service

[Service]
Type=simple
ExecStart=/usr/bin/socat TCP-LISTEN:32402,bind=0.0.0.0,fork,reuseaddr TCP:127.0.0.1:32400
Restart=always
User=root

[Install]
WantedBy=multi-user.target
EOF
```
Enable and start the proxy
```bash
systemctl daemon-reload
systemctl enable plex-proxy
systemctl start plex-proxy
```
Verify both services are running
```bash
systemctl status plexmediaserver
systemctl status plex-proxy
```
Verify if Plex is running on port 32400 and Socat is running on port 32402.

*Expected Output:* You should see two lines, confirming both ports are in the ```LISTEN``` state.
- Plex should be listening on ```127.0.0.1:32400``` (or sometimes ```0.0.0.0:32400```).
- Socat should be listening on ```0.0.0.0:32402``` (which confirms it's ready to receive traffic on all IPs).
```bash
ss -tlnp | grep -E '32400|32402'
```
A correct output will look something like this:
```bash
tcp LISTEN 0 5 127.0.0.1:32400 0.0.0.0:* users:(("Plex Media Server",pid=1234,fd=54))
tcp LISTEN 0 5 0.0.0.0:32402 0.0.0.0:* users:(("socat",pid=5678,fd=3))
```

## 🌐 STEP 8 - Configure Reverse Proxy (OPTIONAL) ##
Choose the section that applies to your setup (Nginx or Nginx Proxy Manager).

*ONLY DO **STEP 8A** IF YOU'RE NOT HOSTING YOUR OWN NGINX REVERSE PROXY SERVER.*

**STEP 8A - Install and Configure NGINX Reverse Proxy (Standalone)**

Install nginx
```bash
apt install nginx -y
```
Install certbot
```bash
apt install certbot python3-certbot-nginx -y
```

# Create nginx configuration
```bash
nano /etc/nginx/sites-available/plex
```
Add the following configuration (Replace ```YOUR_DUCKDNS_DOMAIN.duckdns.org```):
```bash
# Plex Media Server reverse proxy
server {
    listen 80;
    server_name plex.YOUR_DUCKDNS_DOMAIN.duckdns.org; 
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name plex.YOUR_DUCKDNS_DOMAIN.duckdns.org;
    
    # SSL configuration (Certbot will manage)
    ssl_certificate /etc/letsencrypt/live/YOUR_DUCKDNS_DOMAIN.duckdns.org/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/YOUR_DUCKDNS_DOMAIN.duckdns.org/privkey.pem;
    ssl_trusted_certificate /etc/letsencrypt/live/YOUR_DUCKDNS_DOMAIN.duckdns.org/chain.pem;
    
    # SSL settings (reuse your existing config)
    include /etc/nginx/snippets/ssl-params.conf;
    
    # Plex proxy configuration
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "Upgrade";

    # Plex direct access
    location / {
        proxy_pass [http://127.0.0.1:32400](http://127.0.0.1:32400); # CHANGE PORT IF NEEDED
        proxy_buffering off;
    }
    
    # Plex websockets for player
    location /:/websockets/ {
        proxy_pass [http://127.0.0.1:32400](http://127.0.0.1:32400); # CHANGE PORT IF NEEDED
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
    }
    
    # Plex media files
    location /web/ {
        proxy_pass [http://127.0.0.1:32400](http://127.0.0.1:32400); # CHANGE PORT IF NEEDED
    }
}
```
Ctrl + o → Enter → Ctrl + x

**Enable the site**
```bash
ln -s /etc/nginx/sites-available/plex /etc/nginx/sites-enabled/
```

**Restart NGINX for the changes to take effect**
```bash
systemctl restart nginx
```

**STEP 8B - SSL CERTIFICATE VIA NGINX PROXY MANAGER**

Go to your NGINX Proxy Manager container interface and add a new Proxy Host:

| Settings | Value | Notes |
| :--- | :--- | :--- |
| **Domain Names**  | `YOUR_DOMAIN_NAME` | |
| **Scheme** | `HTTP` | |
| **Forward Hostname / IP** | `YOUR_PLEX_IP_ADDRESS (192.168.20.7)` | |
| **Forward Port** | `32400` | Use 32402 if using socat proxy! Otherwise use 32400. |
| **Block Common Exploits** | `Enabled` | |
| **Websockets Support** | `Enabled` | |
| **(Tab SSL) SSL Certificate** | `Request a new SSL Certificate` | |
| **Force SSL** | `Enabled` | |
| **HSTS Enabled** | `Enabled` | |
| **HTTP/2 Support** | `Enabled` | |

## 🔑 STEP 9 - Configure SSH for FileZilla Access ##
Edit SSH configuration: ```nano /etc/ssh/sshd_config```

Make these changes:
````bash
PermitRootLogin no      # Uncomment and change to no
PubkeyAuthentication yes # Uncomment
PasswordAuthentication yes # Uncomment
AllowUsers plexadmin    # Add this line at the end
````
Ctrl + o → Enter → Ctrl + x

Restart SSH for the changes to take effect:
````bash
systemctl restart ssh
````
## 📡 STEP 10 - Configure Port Forwarding on Your Router ##

If you are using the NGINX Proxy Manager, forward ports 80 and 443 to its IP address.

If accessing Plex directly from outside, forward port **32400** to **YOUR_MAIN_PLEX_IP**.

## 💻 STEP 11 - Plex Web Interface Setup ##
Access Plex via: ```http://YOUR_IP_ADDRESS:32400/web``` (or ````:32402/web```` if using socat proxy).

**Initial Setup**

1.  **Sign in** to your Plex account.
2.  **Give the server a name** (e.g., *Proxmox-Plex*).
3.  **Add Libraries using these paths:**

| Library type | Media Folder Path |
| :--- | :--- |
| **Movies** | `/mnt/plexdata/movies` |
| **TV Shows** | `/mnt/plexdata/tvshows` |

**Remote Access Settings**
For your **FIRST/MAIN** Plex Server **ONLY**:
Go to Settings → Remote Access
- Check **"Manually specify public port"**.
- Set Public port to: ```32400```.
- Apply changes.

**Configure Transcoder Settings**
Go to Settings → Transcoder

| Setting | Value |
| :--- | :--- |
| **Transcoder quality** | "Prefer higher speed encoding" |
| **Transcoder temporary directory** | `/tmp/plex-transcode` |
| **Use hardware acceleration when available** | ✓ Checked |
| **Use hardware-accelerated video encoding** | ✓ Checked |
| **Hardware transcoding device** | `auto` |

**For ADDITIONAL Plex Servers ONLY (If running multiple)**:
1.  Go to **Settings → Remote Access**
    * Make sure **Remote Access** is **Disabled**.
2.  Go to **Settings → Network**
    * In **"Custom server access URLs"** fill in your **domain-name**.
3.  **Reboot the container** to make sure the server runs with the desired changes:

    **From Proxmox host:**
    ```bash
    pct restart YOUR_CONTAINER_ID
    ```

    **Or from inside the container console:**
    ```bash
    reboot
    ```

## ✨ NEXT STEPS: Adding Media and Final Checks ##
After completing the web interface setup (Step 11), you are ready to use your Plex server!
1. **Media Upload:** Proceed directly to **STEP 12** to upload your media files using the ```plexadmin``` user and SFTP.
2. **Naming Convention:** Ensure all your movies and TV shows follow the **Plex Naming Convention** (e.g., ```Movie Name (Year).ext``` or ```TV Show Name/Season 01/Episode Name - s01e01.ext```) for proper identification and metadata fetching.
3. **Hardware Check:** Double-check your **Transcoder settings** (under Settings → Transcoder) to ensure both "Use hardware acceleration when available" and "Use hardware-accelerated video encoding" are checked to utilize the GPU passthrough you configured in **STEP 0**.
4. **Verification:** Once media is uploaded, proceed to **STEP 13** to run the final checks, particularly the GPU acceleration test.

## 📤 STEP 12 - Upload Media via FileZilla ##
Use the **plexadmin** user and **SFTP** protocol on port **22**

| Folder | Content |
| :--- | :--- |
| ```/mnt/plexdata/movies/``` | Upload movie files here
| ```/mnt/plexdata/tvshows/``` | Upload TV show files here

## ✅ STEP 13 - Final Verification ##
Test your setup:
- Direct access: ```http://YOUR_PLEX_IP:32400/web```
- Domain access: ```https://YOUR_DOMAIN```
- Remote access: Test from mobile data
- GPU acceleration test: Play a 4K video, check Plex Dashboard for ```(HW)``` indicator. (Optional if you have a 4k video file)
- Verify server appears in ```app.plex.tv````. (Note: If you don't see it directly click on the **"More >"** button and see if the server is in your list
- Run GPU verification: ```sudo vainfo``` (Intel/AMD) or ```nvidia-smi``` (NVIDIA).

## 💡 Troubleshooting Common Issues ##
Hardware Acceleration Not Working
````bash
# 1. Check GPU device access:
ls /dev/dri/ 2>/dev/null && echo "✓ Intel/AMD GPU accessible" || echo "✗ GPU NOT accessible"
ls /dev/nvidia* 2>/dev/null && echo "✓ NVIDIA GPU accessible" || echo "✗ NVIDIA GPU NOT accessible"

# 2. Check driver installation:
which vainfo 2>/dev/null && echo "✓ VAAPI tools installed" || echo "✗ VAAPI tools missing"

# 3. Check plex user permissions:
groups plex | grep -E "video|render|nvidia" && echo "✓ Plex has GPU groups" || echo "✗ Plex missing GPU groups"
````
Port Conflicts / Access Issues
````bash
# Check services are listening:
ss -tlnp | grep -E '32400|32402'

# Check firewall rules:
ufw status numbered | grep -E "32400|32402|22"
````
RAM Disk / Transcoding Issues
````bash
# 1. Check mount:
df -h /tmp/plex-transcode 2>/dev/null && echo "✓ RAM disk mounted" || echo "✗ RAM disk not mounted"

# 2. Check fstab entry:
grep -q "plex-transcode" /etc/fstab && echo "✓ RAM disk in fstab" || echo "✗ RAM disk not in fstab"
````
Quick Fixes Summary
- Reboot container: ```reboot``` 
- Restart Plex: ```systemctl restart plexmediaserver```
- Restart socat: ```systemctl restart plex-proxy```
- Check groups: ```usermod -aG video plex && usermod -aG render plex```

## 📈 Hardware Selection & Performance Guide ##

**Quick Selection Guide**
| Your Hardware | Choose Option | Performance Expectations
| :--- | :--- | :--- |
| **Intel CPU (6th gen+) with iGPU** | OPTION 1 | 1080p: 15-20 streams, 4K: 5-8 streams
| **NVIDIA GPU (GTX 1050+, RTX)** | OPTION 2 | 1080p: Unlimited*, 4K: 8-12 streams*
| **AMD CPU with iGPU (Ryzen G-series)** | OPTION 3 | 1080p: 10-15 streams, 4K: 3-5 streams
| **Intel + NVIDIA Combo** | OPTION 4 | Intel: Efficient transcoding. NVIDIA: Demanding tasks/special codecs

*Requires patch to remove the 3-stream limit.

**Intel Quick Sync Capabilities**
| Generation | GPU Series | Capabilities
| :--- | :--- | :--- |
| Pascal | GTX 10xx | H.264/HEVC 8-bit, Unlimited 1080p*
| Turing | RTX 20xx | H.264/HEVC 10-bit, Better 4K performance
| Ampere | RTX 30xx | AV1 encode, Excellent 4K performance
| Ada Lovelace | RTX 40xx | AV1 encode, Dual encoders, Best performance

*With patch to remove 3-stream limit.
