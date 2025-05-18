# Servarr Stack

Make sure to review everything here and if you have any issues please submit it as an issue. Also, we are more than open to any suggests or edits. Also, checkout the [Servarr Docker Setup](https://wiki.servarr.com/docker-guide) for more details on installing the stack.

## Navigation

  - [Companion Video](#companion-video)
    * [Updates Since Video Publish](#updates-since-video-publish)
  - [Media Server](#media-server)
    * [Jellyfin](https://github.com/TechHutTV/homelab/tree/main/media/jellyfin)
    * [Plex](https://github.com/TechHutTV/homelab/tree/main/media/jellyfin)
  - [Data Directory](#data-directory)
    * [Folder Mapping](#folder-mapping)
    * [Network Share](#network-share)
  - [User Permissions](#user-permissions)
  - [Docker Compose and .env](#docker-compose-and-env)
  - [Gluetun VPN](#gluetun-vpn)
    * [Setup and Configuration](#setup-and-configuration)
    * [Testing Gluetun Connectivity](#testing-gluetun-connectivity)
    * [Passing Through Containers](#passing-through-containers)
    * [External Container to Gluetun](#external-container-to-gluetun)
    * [Gluetun Proxmox LXC Setup](#gluetun-proxmox-fix)
    * [Reduce Gluetun Ram Usage](#reduce-gluetun-ram-usage)
  - [Download Clients](#download-clients)
    * [NZBGet](#nzbget)
      + [NZBGet Login Credentials](#nzbget-login-credentials)
      + [Download Directories Mapping](#nzbget-download-directories)
      + [Fix "directory does not appear" error in Sonarr/Radarr](#fix-directory-does-not-appear-to-exist-inside-the-container-error)
    * [qBittorrent](#qbittorrent)
      + [qBittorrent Login Credentials](#qbittorrent-login-credentials)
      + [Download Directories Mapping](#qbittorrent-download-directories)
      + [qBittorrent Stalls with VPN Timeout](#qbittorrent-stalls-with-vpn-timeout)
  - [arr Apps](#arr-apps)

## Companion Video
```
# Updated video coming soon
[![alt text)](image url)](video link)
```
### Updates Since Video Publish
* Added [ytdl-sub](https://ytdl-sub.readthedocs.io/en/latest/) to the compose.yaml. Remove if unwanted.

## Media Server
Media Servers have their own guides! Check the link below and it will take you to the folder for the guides.

- [Jellyfin](https://github.com/TechHutTV/homelab/tree/main/media/jellyfin)
- [Plex](https://github.com/TechHutTV/homelab/tree/main/media/plex)

## Data Directory
### Folder Mapping
It's good practise to give all containers the same access to the same root directory or share. This is why all containers in the compose file have the bind volume mount ```/data:/data```. It makes everything easier, plus passing in two volumes such as the commonly suggested /tv, /movies, and /downloads makes them look like two different file systems, even if they are a single file system outside the container. See my current setup below. 
```
data
├── books
├── downloads
│   ├── qbittorrent
│   │   ├── completed
│   │   ├── incomplete
│   │   └── torrents
│   └── nzbget
│       ├── completed
│       ├── intermediate
│       ├── nzb
│       ├── queue
│       └── tmp
├── movies
├── music
├── shows
└── youtube
```
Here is a easy command to create the download directory scheme. Run within the /data directory.
```
mkdir -p downloads/qbittorrent/{completed,incomplete,torrents} && mkdir -p downloads/nzbget/{completed,intermediate,nzb,queue,tmp}
```

### Network Share
I generally install Docker on the same LXC that I have my media server on as well as all my data. This; however, is [not recommened by Proxmox](https://pve.proxmox.com/pve-docs/pve-admin-guide.html#chapter_pct). Going forward you should create a seperate VM for all your docker containers and mount the data directory we created in the storage guide with the share. You can also use this method if you're using a serpate share on another machine running something like Unraid or TrueNAS.

Within the VM install `cifs-utils` 
```
sudo apt install cifs-utils
```
Now, edit the fstab file and add the following lines editing them to match your information.
```
sudo nano /etc/fstab
//10.0.0.100/data /data cifs uid=1000,gid=1000,username=user,password=password,iocharset=utf8 0 0
```
Storing the user creditentials within this file it's the best idea. Check out [this question](https://unix.stackexchange.com/questions/178187/how-to-edit-etc-fstab-properly-for-network-drive) on Stack Exchange to learn more.

Now reload the configuration and mount the shares with the following commands.
```
sudo systemctl daemon-reload
sudo mount -a
```

## User Permissions
Using bind mounts (path/to/config:/config) may lead to permissions conflicts between the host operating system and the container. To avoid this problem, you we specify the user ID (PUID) and group ID (PGID) to use within some of the containers. This will give your user permission to read and write configuration files, etc.

In the compose file I use `PUID=1000` and `PGID=1000`, as those are generally the default user ID's in most Linux systems, but depending on your setup you may need to chage this.

```
id your_user
uid=1000(brandon) gid=1000(brandon) groups=1000(brandon),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),101(lxd),988(docker),993(render)
```
If you are using a network share mounted though ```/etc/fstab``` match the permissions there. Learn more above.

If you run into errors, after creating all the folders you can assign the permissions using chmod. For example,
```
sudo chown -R 1000:1000 /data
```
Also, I like to store all my Docker configurations in a root /docker directory on my Linux system. These can go whereever you prefer whether that be your home directory or somewhere else. Do note, many Docker apps may have issues if you're trying to store you Docker configurations in a SMB network share.
```
mkdir /docker
sudo chown -R 1000:1000 /docker
```
## Docker Compose and .env
Navigate to the directory you want to spin up the servarr stack in. I run mine from `/docker/servarr` but you can run it from anywhere you'd like such as `/home/user/docker/servarr`. Then download the compose.yaml and .env files from this repo.
```
wget https://github.com/TechHutTV/homelab/raw/refs/heads/main/media/compose.yaml && wget https://github.com/TechHutTV/homelab/raw/refs/heads/main/media/.env
```
Most of our editing is going to be done in the .env file. Here you change your UID and GID, timezone, and add all your VPN keys and info. You can also make edits to the compose.yaml file such as the mount point locations, for example, if you are using something other than `/data:/data` or even changing the docker network IP addresses for your services.

## Gluetun VPN

### Setup and Configuration
I like to set this out with [AirVPN](https://airvpn.org/?referred_by=673908) (referral link). I'm not affliated with them in anyway other than the referral link. I've tried a few other providers and they're my preference. If you already have a VPN checkout the [providers](https://github.com/qdm12/gluetun-wiki/tree/main/setup/providers) page on their wiki.

On AirVPN navigate to the **Client Area** from here select the Config Generator. Now in the options select **Linux** then toggle the **WireGuard** option. Select **New device** and then scroll down to **By single server** and select a server that is best for you. For example, _Titawin (Vancouver)_ was my selection becuase, at the time, it had the fewest users with good speeds. Scoll all the way to the bottom and select **Generate**. This will download a conf file with all of your information.

Back AirVPN navigate to the **Client Area** from here select Manage under Ports. If you already have a port open click on **Test open** otherwise click the plus button under **Add a new port** then click **Test open** for that port. Here you will find the specific servers that you can use your port on. If there is a Connection refused warning next the server you generated your configuration for change the port until the warning goes away. For example, in my case the _'Titawin (Vancouver)_ sever that I selected with my port is good to use.

> [!CAUTION]
> do NOT forward on your router the same ports you use on your listening services while connected to the VPN.

Now, in the same directory as your docker compose.yaml file create a `.env` file. Paste in the varibles below and then add all the information from your downloaded .conf file.

```
nano .env
```
```
# General UID/GIU and Timezone
TZ=America/Los_Angeles
PUID=1000
PGID=1000

# Input your VPN provider and type here
VPN_SERVICE_PROVIDER=airvpn
VPN_TYPE=wireguard

# Mandatory, airvpn forwarded port
FIREWALL_VPN_INPUT_PORTS=port # mandatory, airvpn forwarded port

# Copy all these varibles from your generated configuration file
WIREGUARD_PUBLIC_KEY=key
WIREGUARD_PRIVATE_KEY=key
WIREGUARD_PRESHARED_KEY=key
WIREGUARD_ADDRESSES=ipv4

# Optional location varbiles, comma seperated list,no spaces after commas, make sure it matches the config you created
SERVER_COUNTRIES=country
SERVER_CITIES=city 

# Heath check duration
HEALTH_VPN_DURATION_INITIAL=120s
```

### Testing Gluetun Connectivity 
Once your containers are up and running, you can test your connection is correct and secured. This assumes you keeo the gluetun container name. Learn more at the [gluetun wiki](https://github.com/qdm12/gluetun-wiki/blob/main/setup/test-your-setup.md). __Note:__ If you run into issues try restarting the stack with `docker compose restart`.
```
docker run --rm --network=container:gluetun alpine:3.18 sh -c "apk add wget && wget -qO- https://ipinfo.io"
```
If you'd like to test Gluetun Connectivity from a container using the service jump into the Exec console and run the wget command below. Tested with nzbget, deluge, and prowlarr. Ensure you open the ports through the the gluetun container.
```
docker exec -it conatiner_name bash
wget -qO- https://ipinfo.io
```
### Passing Through Containers 
When containers are in the same docker compose they all you need to add is a ```network_mode: service:container_name``` and open the ports through the the gluetun container. See example with a different torrent client below.
```
services:
  gluetun:
    image: qmcgaw/gluetun
    container_name: gluetun
    ...
    ports:
      - 8888:8112 # deluge web interface
      - 58846:58846 # deluge RPC
  deluge:
    image: linuxserver/deluge:latest
    container_name: deluge
    ...
    network_mode: service:gluetun
```
### External Container to Gluetun
Add the following when launching the container, provided Gluetun is already running on the same machine. 
```
--network=container:gluetun
``` 
If the container is in another docker compose.yaml, assuming Gluetun is already running add the following network mode. Ensure you open the ports through the the gluetun container.
```
network_mode: "container:gluetun"
```

### Gluetun Proxmox LXC Setup

"cannot Unix Open TUN device file: operation not permitted and cannot create TUN device file node: operation not permitted" May happen if you're running this on LXC containers.

Find your container number, for example mine is 101

Edit `/etc/pve/lxc/101.conf` and add:
```
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net dev/net none bind,create=dir
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
```
Make sure you pass through the tun device (/dev/net/tun:/dev/net/tun) as shown in my compose file.

### Reduce Gluetun Ram Usage
As mentioned in this [issue](https://github.com/TechHutTV/homelab/issues/12) there is [feature request](https://github.com/qdm12/gluetun/issues/765#issuecomment-1019367595) on the Glutun Github page to help reduce ram usage. Gluetun bundles a recursive caching DNS resolver called unbound for handling domain name requests securely. Over time the cache size, which rests in RAM, can balloon to gigabytes.

You can do this by adding the following to your docker compose file under the glutun enviromental varibles.
```
BLOCK_MALICIOUS=off
```
This may not be an issue as [DNS over HTTPS in Go to replace Unbound](https://github.com/qdm12/gluetun/issues/137) is impletmented, but it's worth the mention.

## Download Clients

### NZBGet

#### NZBGet Login Credentials 
The default credentials for NZBGet are a username of `nzbget` and a password of `tegbzn6789`. It's strongly recommended to change these default credentials for security reasons. This can be done under Settings > SECURITY, then change the ControlUsername and ControlPassword.

#### NZBGet Download Directories
If following the /data:/data directory scheme and used the command to setup the download directories open the qBitttorent Web UI and do under Settings > PATHS and change the paths.

_MainDir:_ `/data/downloads/nzbget`

_DestDir:_ `${MainDir}/completed`

_InterDir:_ `${MainDir}/intermediate`

And keep everything else as is.

#### Fix directory does not appear to exist inside the container error
This error may appear within Sonarr and Radarr. Once NZBGet is setup go to settings and under **INCOMING NZBS** change the **AppendCategoryDir** to **No**. This will prevent some potential mapping issues and save on unnessesary directories.

### qBittorrent

#### qBittorrent Login Credentials
When you first launch qBittorrent it will be givin a random password. To find this password you can view the logs to see what the password is.
```
docker container logs qbittorrent 
```
Now, go to your settings and setup a new username and password under WebUI > Authentication.

#### Qbittorrent Download Directories
If following the /data:/data directory scheme and used the command to setup the download directories open the qBitttorent Web UI and do under Settings > Downloads and change the paths.

_Default Save Path:_ `/data/downloads/qbittorrent/completed`

_Keep incomplete torrents in:_ `/data/downloads/qbittorrent/incomplete`

_Copy .torrent files to:_ `/data/downloads/qbittorrent/torrents`

#### qBittorrent Stalls with VPN Timeout
qBittorrent stalls out if there is a timeout or any type of interuption on the VPN. This is good becuase it drops connection, but we need it to fire back up when the connection is restored without manually restarting the container.

__Solution #1:__ Within the WebUI of qbittorrent head over to advanced options and select ```tun0``` as the networking interface. See image below for example.

![Set Network Interface to tun0](https://github.com/TechHutTV/homelab/blob/main/media/images/qbittorrent_tun0.jpeg)

Next, I added ```HEALTH_VPN_DURATION_INITIAL=120s``` to my glutun enviroment varibles as [per this issue](https://github.com/qdm12/gluetun/issues/1832). I updated my compose.yaml above with this varible so you may already have this enabled. You can learn more about this on their [wiki](https://github.com/qdm12/gluetun-wiki/blob/main/faq/healthcheck.md). If you continue to have issues continue to next solution.

__Solution #2:__ Another solution, that can be used in conjection with __Solution #1__ is using the [deunhealth](https://github.com/qdm12/deunhealth/tree/main) container to automatically restart qbittorrent when it give an unheathly status. We've added this to our compose.yaml for this stack.
```
  deunhealth:
    image: qmcgaw/deunhealth
    container_name: deunhealth
    network_mode: "none"
    environment:
      - LOG_LEVEL=info
      - HEALTH_SERVER_ADDRESS=127.0.0.1:9999
      - TZ=America/Los_Angeles
    restart: always
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
```

Next we need to add a health check and label to our qbittorrent container. We add ```deunhealth.restart.on.unhealthy=true``` as a label and a simple ping health check as shown below.

```
  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    restart: unless-stopped
    labels:
      deunhealth.restart.on.unhealthy=true # Label added for deunhealth monitoring
    ...
```
Relevent Resources: [DBTech video on deunhealth](https://www.youtube.com/watch?v=Oeo-mrtwRgE), [gluetun/issues/2442](https://github.com/qdm12/gluetun/issues/2442) and [gluetun/issues/1277](https://github.com/qdm12/gluetun/issues/1277#issuecomment-1352009151)

## arr Apps

When connecting your *arr applications be sure to use the new configured IP addresses in the servarrnetwork. We will soon update this section with more text documentation.
