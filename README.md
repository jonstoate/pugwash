



# Pugwash

A server for Media Management. Add-ons provide for self hosted cloud, password and photo managment.

[TOC]

## Introduction

This guide has been written to install and configure a Raspberry PI single-board computer the software is installed using docker, so this can be used to install on any computer (or NAS) that runs Linux and docker.

### Hardware

**Server**

- Raspberry Pi 4 - 4Gb RAM - minimum configuration
- Raspberry Pi 4 - 8Gb RAM - recommended configuration for base media server
- Raspberry Pi 5 - 8Gb RAM - recommended configuration for full featured server (16Gb if your budget can stretch that far)
- Raspberry Pi 5 - 16Gb with Raspberry Pi 5 NVMe SSD Kit with M.2 HAT+ - 256GB - for the optimal setup

**Media**

- USB Flash drive - boot drive for Raspberry Pi 4 

- Raspberry Pi 5 NVMe SSD Kit with M.2 HAT+ - 256GB (for Raspberry 5)

- Powered External Hard drive.

  OR

- Powered USB Hub, and external hard drive

:bulb: **Power:** The Raspberry Pi is powered using a usb port, and probably wont supply sufficient power for portable external hard drive. This is why we user either a powered hard drive or a powered usb hub that will supply the power to our hard drive.

### Software

- **Gluetun** as a VPN container
- **OpenVPN** - (deprecated) Easy to use, minimal hassle VPN server, and to keep all we do private
- **qBittorrent** -  Lightweight open-source BitTorrent client.
- **Jellyfin** -open-source media server designed to organize, manage, and share digital media files.
- **Sonarr** - is a PVR that can automatically download and update a TV show library
- **Radarr** - is a PVR that can automatically download and update a library of movies
- **Lidarr** - is a music collection manager.
- **Bazarr** - is a companion application to Sonarr and Radarr that manages and downloads subtitles
- **Flaresolver** - a proxy server to bypass Cloudflare and DDoS-GUARD protection.
- **Prowlarr** or **Jackett** - index management to find torrents to download
- **sftpgo** - file transfer server, to tranfer files from or to the server
- **Immich** - Photo management
- **Nextcloud** - Web-based file management
- **KeePassXC** - password management
- **Nginx** - lightweight web server to provide a summary web page
- **Pi-hole** - ad-block and IOT monitor
- **Watchtower** - automate the updating if Docker containers

Icons for the dashboard are here: https://dashboardicons.com/icons

## Before you start

You will need:

- A **VPN Service** - I use NordVPN, which has a interdependently audited no-log policy. If you use something different then i recommend checking here [gluetun](https://github.com/qdm12/gluetun)  for comparability
- You will want to know the DNS server addresses of the VPN provider.
-  

A note on passwords:

This server is sitting on a private network and the applications (in most cases) do not need a high level of security from users on the network.  I use the same password across all the applications, except where noted.



## Server setup

Install the Operating system

Raspberry PI OS Lite - no desktop, as we are going to run headless

Boot the server - The starting point is a new clean install of the operating system - and an initial boot.

ssh to the server (hint, check your router if you don't know the ip address)

This will connect to the server without saving the key, as we are going to change the ip address to a static one.

```
ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null user@host
```



### Change boot device

For our server we will want to run the operating system from either a **usb** device or **NVMe SSD**. Either of these offer better performance and reliability over the SD card.

[change boot device](RaspberryPi.md#Change boot device)



### Set Static IP Address

The next set is to set a static IP address - so our server is always known by the same address on the network.

[Static IP address](RaspberryPi.md#Static IP Address)



### Install external hard drive

We are now installing an external hard drive. - which we will mount at /srv/data



Make sure you know what your system architecture is.

```
dpkg --print-architecture
```

> ```
> jon@pugwash:~ $ dpkg --print-architecture
> arm64
> ```

### User and group

Create a user - and a group to which all will belong

```bash
sudo adduser media
```

You will be prompted for a password and other details - after entering a password just press enter to accept the defaults.

```bash
sudo adduser media
New password: 
Retype new password: 
passwd: password updated successfully
Changing the user information for media2
Enter the new value, or press ENTER for the default
	Full Name []: 
	Room Number []: 
	Work Phone []: 
	Home Phone []: 
	Other []: 
Is the information correct? [Y/n]
```

:bulb: **Passwords:** I am using `qazwsxedc` as the password for all of our application that are accessible from the internal network only. I use this for the media user on install, and will change it later

Make a note of the uid and gid for the user we have just created

```bash
id media
```

We will want the ids later when we are setting up the applications

```
uid=1001(media) gid=1001(media) groups=1001(media),100(users)
```

:warning: **Important:** We will need the `uid` and `gid` when we are installing the applications .

And then add my user the media group

```
sudo adduser $USER media
```

#### A note on users

There are two key users in our setup.

1. **me** - the user we created when we installed our operating system
2. **media** - the user we just created and will "own" all the files we download and will run the applications we install.



### Directories and files

Make the various directories we are going to use.

There are where the files we download are stored. We are saving our files under **/srv** and we have mounted our data drive as /srv/data. This directory structure supports using docker for installing our apps, and makes sense even if we are installing the apps directly.

The layout is described here; https://wiki.servarr.com/docker-guide#consistent-and-well-planned-paths

| Directory       | Description                                        |
| --------------- | -------------------------------------------------- |
| /srv/data       | mount point for our hard disk                      |
| ├── Torrent     | Base folder for all torrent data                   |
| │  ├── Download | The folder for all that we download                |
| └── MediaCentre | The base folder for all our media data             |
| ├── Audio       | Audio files                                        |
| ├── Book        | Book that we download (and are not in our library) |
| ├── Gallery     | Our photo gallery                                  |
| ├── Library     | Our calibre library                                |
| ├── Movie       | The movies that we watch                           |
| └── TV          | TV shows that we watch                             |

:memo: **Note:** We have used singular names for everything - just for consistency.

:bulb: Downloads don’t even have to be sorted into subfolders either, since movies, music and tv will rarely conflict.

This will make all the directories we want and set the permissions on the directories. The *"chown"* sets the ownership and the *"chmod g+s"* mean that any file or directory created in a directory will get the directory's group ownership (media).

```
sudo mkdir -p /srv/data/MediaCentre && \
sudo mkdir /srv/data/MediaCentre/Audio && \
sudo mkdir /srv/data/MediaCentre/Book && \
sudo mkdir /srv/data/MediaCentre/Gallery && \
sudo mkdir /srv/data/MediaCentre/Library && \
sudo mkdir /srv/data/MediaCentre/Movie && \
sudo mkdir /srv/data/MediaCentre/Music && \
sudo mkdir /srv/data/MediaCentre/TV && \
sudo chmod -R g+ws /srv/data/MediaCentre/ && \
sudo chown -R media:media /srv/data/MediaCentre && \
sudo mkdir /srv/data/Torrent && \
sudo mkdir /srv/data/Torrent/Download && \
sudo mkdir /srv/data/Torrent/Complete && \
sudo chown -R media:media /srv/data/Torrent && \
sudo chmod -R g+ws /srv/data/Torrent &&
```





## Docker

Docker is open-source software for deploying and running of containerized applications. This means that the heavy lifting has been done for us- and all we need to do is installed the pre-packaged container.

```bash
# Add Docker's official GPG key:
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/debian
Suites: $(. /etc/os-release && echo "$VERSION_CODENAME")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
```

Now install the docker packages

```bash
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

The install process will have created a docker group - so add our user to that group

```bash
sudo usermod -aG docker $USER
```

Log out and then back in again to establish the group setting.

The `id` command will now show docker as one of our groups

Now test the install with `docker run hello-world`

```bash
docker run hello-world
```

You should see and number of messages - and confirmation that all is well

```console
jon@pugwash:~ $ docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
58dee6a49ef1: Pull complete 
c3bdf82c34d1: Download complete 
Digest: sha256:f9078146db2e05e794366b1bfe584a14ea6317f44027d10ef7dad65279026885
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.
...
```

We are now ready to install our applications

## Docker images

Where practical we install images from **linuxserver.io** -  https://www.linuxserver.io/. These images are well built. well maintained and documented.  where an image is not available from here we use the best option for that application.

Make a directory for our docker files in `/opt`

```bash
cd /opt
sudo mkdir docker
sudo chown $USER:$USER docker
```

Within `/opt/docker` we will have individual directories for each application.



## Docker Apps

### Gluetun

 A **VPN**, or virtual private network, is used to protect our on-line privacy and ensure that our activity cannot be monitored.

[Gluetun](https://github.com/qdm12/gluetun) is a VPN client in a thin Docker container for multiple VPN providers,  written in Go, and using OpenVPN or Wireguard, DNS over TLS, with a few  proxy servers built-in.



## VPN

 A **VPN**, or virtual private network, is used to protect our on-line privacy and ensure that our activity cannot be monitored.

### Installation

We are using OpenVPN to provide the foundation for our VPN service

```bash
sudo apt install openvpn
```

Login to NordVPN and follow the manual setup procedure. If you are using a different VNP provider then follow their setup process.

You will need;

- user token and password
- an OpenVPN configuration file (copy the link from *Download TCP* )

Create a file in /etc/openvpn/client called login.conf. This will have two lines userid and password. We then change the permissions on the file, so it can't be messed with.

```bash
cd /etc/openvpn/client
sudo vi login.conf
sudo chmod 600 login.conf
sudo wget https://downloads.nordcdn.com/configs/files/ovpn_tcp/servers/?????.nordvpn.com.tcp.ovpn
```

To work with the OpenVPN startup process the file need to renamed something**.conf** 

```
sudo mv ?????.nordvpn.com.tcp.ovpn nordvpn.conf
sudo vi nordvpn.conf
```

and change this line

```
auth-user-pass
```

to this

```
auth-user-pass /etc/openvpn/client/login.conf
```

Do not change anything else

There is a OpenVPN startup file in the `/usr/lib/systemd/system` folder - `/usr/lib/systemd/system/openvpn-client@.service`

We don;t need to change it we just enable and start the service it by running..

```bash
sudo systemctl enable openvpn-client@nordvpn.service
sudo systemctl start openvpn-client@nordvpn.service
```

:memo: **Note:** The text after the @ sign is the filename from earlier - without the .conf



We can check the status with

```bash
sudo systemctl status openvpn-client@nordvpn.service
```

you should see something like this

```
● openvpn-client@nordvpn.service - OpenVPN tunnel for nordvpn
     Loaded: loaded (/usr/lib/systemd/system/openvpn-client@.service; enabled; preset: enabled)
     Active: active (running) since Sun 2026-02-15 13:31:40 AWST; 2min 45s ago
 Invocation: 30110243981b48dea9c20b655b62083e
       Docs: man:openvpn(8)
             https://openvpn.net/community-resources/reference-manual-for-openvpn-2-6/
             https://community.openvpn.net/openvpn/wiki/HOWTO
   Main PID: 3451 (openvpn)
     Status: "Initialization Sequence Completed"
      Tasks: 1 (limit: 3919)
        CPU: 83ms
     CGroup: /system.slice/system-openvpn\x2dclient.slice/openvpn-client@nordvpn.service
             └─3451 /usr/sbin/openvpn --suppress-timestamps --nobind --config nordvpn.conf

Feb 15 13:31:42 pugwash openvpn[3451]: TUN/TAP device tun0 opened
Feb 15 13:31:42 pugwash openvpn[3451]: net_iface_mtu_set: mtu 1500 for tun0
Feb 15 13:31:42 pugwash openvpn[3451]: net_iface_up: set tun0 up
Feb 15 13:31:42 pugwash openvpn[3451]: net_addr_v4_add: 10.100.0.2/20 dev tun0
Feb 15 13:31:42 pugwash openvpn[3451]: net_route_v4_add: 194.107.161.16/32 via 192.168.0.1 dev [NULL] table 0 metri>
Feb 15 13:31:42 pugwash openvpn[3451]: net_route_v4_add: 0.0.0.0/1 via 10.100.0.1 dev [NULL] table 0 metric -1
Feb 15 13:31:42 pugwash openvpn[3451]: net_route_v4_add: 128.0.0.0/1 via 10.100.0.1 dev [NULL] table 0 metric -1
Feb 15 13:31:42 pugwash openvpn[3451]: Initialization Sequence Completed
Feb 15 13:31:42 pugwash openvpn[3451]: Data Channel: cipher 'AES-256-GCM', peer-id: 0, compression: 'stub'
Feb 15 13:31:42 pugwash openvpn[3451]: Timers: ping 60, ping-restart 180
```

Now if we run `ip a`we should see an entry that look like this

```
ip a
....
4: tun0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UNKNOWN group default qlen 500
    link/none 
    inet 10.100.0.2/20 scope global tun0
       valid_lft forever preferred_lft forever
    inet6 fe80::3bd9:30eb:1cf:9202/64 scope link stable-privacy proto kernel_ll 
       valid_lft forever preferred_lft forever
```





### qBittorrent

We are installing  a command-line version of qBittorrent;  **qbittorrent-nox** - which is ideal for running a headless environment.

```
sudo apt install qbittorrent-nox
```

Let's start qBittorrent to make sure all is well

```
qbittorrent-nox
```

We see will this:

```
*** Legal Notice ***
qBittorrent is a file sharing program. When you run a torrent, its data will be made available to others by means of upload. Any content you share is your sole responsibility.

No further notices will be issued.

Press 'y' key to accept and continue...
y
WebUI will be started shortly after internal preparations. Please wait...

******** Information ********
To control qBittorrent, access the WebUI at: http://localhost:8080

The Web UI administrator username is: admin
The Web UI administrator password has not been changed from the default: adminadmin
This is a security risk, please change your password in program preferences.
```

Press **Ctrl+C** to quit the program 

#### qBittorrent service

We will create a service that will run and manage the qBittorrent client.

Firstly we need a user

```shell
sudo adduser qbittorrent --ingroup media --shell /usr/sbin/nologin
```



Now create the service file

```shell
sudo vi /etc/systemd/system/qbittorrent.service
```

And add this...

```
[Unit]
Description=qBittorrent
After=network.target

[Service]
WorkingDirectory=/home/media
Type=forking
User=media
Group=media
UMask=002
ExecStart=/usr/bin/qbittorrent-nox -d --webui-port=8080
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

What all this means

The qBittorrent software will be run under the user `qbittorrent` and the group `media` that we created earlier.

We apply a “`UMask`” to the user so that all files created by this user will hold the following Linux permission, “`rw-rw-r--`“.  This permission means that the owner and group can read and write all files created by this user.

We start the qBittorrent executable `/usr/bin/qbittorrent-nox` using it full path and with:

- `-d`, which tells the program we want to run it as a “daemon”.
- `--webui-port`” gives the port we will connect to with our browser for the web interface

Now start the service with

```shell
sudo systemctl start qbittorrent
```

Now start the service with

```shell
sudo systemctl status qbittorrent
```

You should see the service is:

```
     Active: active (running) since Sat 2025-10-11 10:06:18 AWST; 8s ago
```

as well as some other information.

Now enable the service with

```shell
sudo systemctl enable qbittorrent
```

Which means the service will start whenever the Pi starts.

#### qBittorrent Web Interface

This is how we manage the application

Open a browser and enter

```http
http://<the pi ip address>:8080
```



Downloads

/mnt/data/Torrent/Download

/mnt/data/Torrent/Download/temp



Advanced

​                        Network interface:                    tun0



#### Web API

https://github.com/qbittorrent/qBittorrent/wiki/WebUI-API-(qBittorrent-4.1)



## TODO

https://selfhosting.sh/apps/qbittorrent/



### FlareSolverr

FlareSolverr is a proxy server to bypass Cloudflare and DDoS-GUARD protection.

FlareSolverr starts a proxy server, and it waits for user requests in an idle state using few resources. When some request arrives, it uses [Selenium](https://www.selenium.dev) with the [undetected-chromedriver](https://github.com/ultrafunkamsterdam/undetected-chromedriver) to create a web browser (Chrome). It opens the URL with user parameters and waits until the Cloudflare challenge is solved (or timeout). The HTML code and the cookies are sent back to the user, and those cookies can be used to bypass Cloudflare using other HTTP clients.

```bash
mkdir /opt/docker/flaresolverr && \
cd /opt/docker/flaresolverr && \
vi flaresolverr.yaml
```

and paste

```yaml
---
services:
  flaresolverr:
    # DockerHub mirror flaresolverr/flaresolverr:latest
    image: ghcr.io/flaresolverr/flaresolverr:latest
    container_name: flaresolverr
    environment:
      - LOG_LEVEL=${LOG_LEVEL:-info}
      - LOG_HTML=${LOG_HTML:-false}
      - CAPTCHA_SOLVER=${CAPTCHA_SOLVER:-none}
      - TZ=Australia/West    
    ports:
      - "${PORT:-8191}:8191"
    restart: unless-stopped
```



Now start the application with:

```bash
docker compose -f flaresolverr.yaml up -d
```

We will now see messages as docker downloads the image

Once this is finished you can run `docker container ls --latest`  - to lost the latest container which will display something like this

```bash
docker container ls --latest
CONTAINER ID   IMAGE                                      COMMAND                  CREATED              STATUS              PORTS                                                   NAMES
dea2975ad88e   ghcr.io/flaresolverr/flaresolverr:latest   "/usr/bin/dumb-init …"   About a minute ago   Up About a minute   0.0.0.0:8191->8191/tcp, [::]:8191->8191/tcp, 8192/tcp   flaresolverr

```

#### Application Setup¶

Access the webui at http://192.168.0.250:9696, for more information check out [Prowlarr](https://github.com/Prowlarr/Prowlarr).

Setup info can be found here. https://wikijs.servarr.com/prowlarr/quick-start-guide







### Prowlarr

[Prowlarr](https://github.com/Prowlarr/Prowlarr) is a indexer manager/proxy built on the popular arr .net/reactjs base stack to  integrate with your various PVR apps. Prowlarr supports both Torrent  Trackers and Usenet Indexers. 

:warning: **NOTE:** Prowlarr is giving me issues with Cloudfare and not connecting to many sites - so going to Jacket instead

```bash
mkdir /opt/docker/prowlarr && \
cd /opt/docker/prowlarr && \
vi prowlarr.yaml
```

and paste

```yaml
---
services:
  prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    environment:
      - PUID=1001
      - PGID=1001
      - TZ=Australia/West
    volumes:
      - /opt/docker/prowlarr/data:/config
    ports:
      - 9696:9696
    restart: unless-stopped

```



Now start the application with:

```bash
 docker compose -f prowlarr.yaml up -d
```

We will now see messages as docker downloads the image

Once this is finished you can run `docker container ls --latest`  - to lost the latest container which will display something like this

```bash
docker container ls --latest
CONTAINER ID   IMAGE                                 COMMAND   CREATED          STATUS          PORTS                                         NAMES
b8f41edaa713   lscr.io/linuxserver/prowlarr:latest   "/init"   47 seconds ago   Up 43 seconds   0.0.0.0:9696->9696/tcp, [::]:9696->9696/tcp   prowlarr

```

#### Application Setup¶

Access the webui at http://192.168.0.250:9696, for more information check out [Prowlarr](https://github.com/Prowlarr/Prowlarr).

Setup info can be found here. https://wikijs.servarr.com/prowlarr/quick-start-guide



### Jackett

[Jackett](https://github.com/Jackett/Jackett) works as a  proxy server: it translates queries from apps (Sonarr, SickRage,  CouchPotato, Mylar, etc) into tracker-site-specific http queries, parses the html response, then sends results back to the requesting software.  This allows for getting recent uploads (like RSS) and performing  searches. Jackett is a single repository of maintained indexer scraping  & translation logic - removing the burden from other apps.

tead

```bash
mkdir /opt/docker/jackett && \
cd /opt/docker/jackett && \
vi jackett.yaml
```

and paste

```yaml
---
services:
  jackett:
    image: lscr.io/linuxserver/jackett:latest
    container_name: jackett
    environment:
      - PUID=1001
      - PGID=1001
      - TZ=Australia/West
      - AUTO_UPDATE=true #optional
    volumes:
      - /opt/docker/jackett/data:/config
      - /mnt/data/Torrent/torrent:/downloads
    ports:
      - 9117:9117
    restart: unless-stopped

```



Now start the application with:

```bash
 docker compose -f jackett.yaml up -d
```

We will now see messages as docker downloads the image

Once this is finished you can run `docker container ls --latest`  - to lost the latest container which will display something like this

```bash
docker container ls --latest
CONTAINER ID   IMAGE                                COMMAND   CREATED          STATUS          PORTS                                         NAMES
dae77c9c1694   lscr.io/linuxserver/jackett:latest   "/init"   25 seconds ago   Up 20 seconds   0.0.0.0:9117->9117/tcp, [::]:9117->9117/tcp   jackett

```

#### Application Setup¶

The web interface is at  http://192.168.0.250:9117 , configure various trackers and connections to other apps there. More info at [Jackett](https://github.com/Jackett/Jackett).



Add indexers



### 

### Sonarr

[Sonarr](https://sonarr.tv/) (formerly NZBdrone) is a PVR for usenet and bittorrent users. It can monitor multiple RSS feeds for new  episodes of your favorite shows and will grab, sort and rename them. It  can also be configured to automatically upgrade the quality of files  already downloaded when a better quality format becomes available.

https://hub.docker.com/r/linuxserver/sonarr/



```bash
mkdir /opt/docker/sonarr && cd /opt/docker/sonarr && vi sonarr.yaml
```

and paste

```
---
services:
  sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    environment:
      - PUID=1001
      - PGID=1001
      - TZ=Australia/West
    volumes:
      - /opt/docker/sonarr/config:/config
      - /mnt/data:/mnt/data
    ports:
      - 8989:8989
    restart: unless-stopped
```



```bash
docker compose -f sonarr.yaml up -d
```

We will now see messages as docker downloads the image

Once this is finished you can run `docker container ls --latest`  - to lost the latest container which will display something like this

```bash
docker container ls --latest
CONTAINER ID   IMAGE                               COMMAND   CREATED          STATUS          PORTS                                         NAMES
ea02ed177102   lscr.io/linuxserver/sonarr:latest   "/init"   15 seconds ago   Up 11 seconds   0.0.0.0:8989->8989/tcp, [::]:8989->8989/tcp   sonarr

```

#### Application Setup¶

The web interface is at  http://192.168.0.250:8989. More information is here:



```
username: admin
password: qazwsxedc
```



Import existing



Add indexers





qBittorrent save path

sonarr-tv /mnt/data/Torrent/TV





### Radarr

Radarr - Radarr is a movie collection manager

https://hub.docker.com/r/linuxserver/radarr



```bash
mkdir /opt/docker/radarr && cd /opt/docker/radarr && vi radarr.yaml
```



and paste

```properties
---
services:
  radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    environment:
      - PUID=1001
      - PGID=1001
      - TZ=Australia/West
    volumes:
      - /opt/container/radarr/config:/config
      - /mnt/data:/mnt/data
    ports:
      - 7878:7878
    restart: unless-stopped
```

Now run;

```bash
docker compose -f radarr.yaml up -d
```

We will now see messages as docker downloads the image

Once this is finished you can run `docker container ls --latest`  - to view the latest container which will display something like this

```bash
docker container ls --latest
CONTAINER ID   IMAGE                               COMMAND   CREATED          STATUS          PORTS                                         NAMES
3accdc3966a1   lscr.io/linuxserver/radarr:latest   "/init"   44 seconds ago   Up 32 seconds   0.0.0.0:7878->7878/tcp, [::]:7878->7878/tcp   radarr
```

#### 

#### Application setup

Access the webui at http://192.168.0.250:7878, for more information check out [Radarr](https://github.com/Radarr/Radarr).

username: admin

password: qazwsxedc





**qBittorrent save path**

radarr /mnt/data/Torrent/Movie

### 

### Lidarr

https://hub.docker.com/r/linuxserver/lidarr



```bash
mkdir /opt/docker/lidarr && cd /opt/docker/lidarr && vi lidarr.yaml
```

and paste

```properties
---
services:
  lidarr:
    image: lscr.io/linuxserver/lidarr:latest
    container_name: lidarr
    environment:
      - PUID=1001
      - PGID=1001
      - TZ=Australia/West
    volumes:
      - /opt/docker/lidarr/data:/config
      - /mnt/data:/mnt/data
    ports:
      - 8686:8686
    restart: unless-stopped    
```

Now we  run `docker compose`, which will pull down the image and start up the container

```bash
docker compose -f /opt/docker/lidarr/lidarr.yaml up -d
```

Once this is finished you can run `docker container ls --latest`  - to view the latest container which will display something like this

```bash
docker container ls --latest

CONTAINER ID   IMAGE                               COMMAND   CREATED       STATUS       PORTS                                         NAMES
1206c3cbe298   lscr.io/linuxserver/lidarr:latest   "/init"   2 hours ago   Up 2 hours   0.0.0.0:8686->8686/tcp, [::]:8686->8686/tcp   lidarr
```

Now open a browser and enter;

```http
http://pugwash:8686
```

username: admin

password: qazwsxedc



### 

### 

### Bazarr

https://hub.docker.com/r/linuxserver/bazarr





### Jellyfin

Jellyfin is a Free Software Media System that puts you in control of managing and streaming your media. It is an alternative to the proprietary Emby and Plex, to provide media from a dedicated server to end-user devices via multiple apps.

https://hub.docker.com/r/linuxserver/jellyfin



```bash
mkdir /opt/docker/jellyfin && cd /opt/docker/jellyfin && vi jellyfin.yaml
```

and paste

```
---
services:
  jellyfin:
    image: lscr.io/linuxserver/jellyfin:latest
    container_name: jellyfin
    environment:
      - PUID=1001
      - PGID=1001
      - TZ=Australia/West
      - JELLYFIN_PublishedServerUrl=http://192.168.0.250 #optional
    volumes:
      - /path/to/jellyfin/library:/config
      - /mnt/data/MediaCentre/TV:/data/tvshows
      - /mnt/data/MediaCentre/Movie:/data/movies
      - /mnt/data/MediaCentre/Music:/data/music
    ports:
      - 8096:8096
      - 8920:8920 #optional
      - 7359:7359/udp #optional
      - 1900:1900/udp #optional
    restart: unless-stopped

```



```bash
docker compose -f jellyfin.yaml up -d
```

We will now see messages as docker downloads the image

Once this is finished you can run `docker container ls --latest`  - to lost the latest container which will display something like this

```bash
docker container ls --latest
CONTAINER ID   IMAGE                               COMMAND   CREATED          STATUS          PORTS                                         NAMES
ea02ed177102   lscr.io/linuxserver/sonarr:latest   "/init"   15 seconds ago   Up 11 seconds   0.0.0.0:8989->8989/tcp, [::]:8989->8989/tcp   sonarr

```

#### Application Setup¶

The web interface is at  http://192.168.0.250:8096. More information is here:



Add indexers



username: admin

password: qazwsxedc



### sftpgo

sftpgo is a Secure SFTP & FTP Server for Cloud & Local Storage

https://sftpgo.com/

```bash
mkdir /opt/docker/sftpgo && cd /opt/docker/sftpgo && vi sftpgo.yaml
```

and paste

```yaml
---
services:
  sftpgo:
    image: drakkan/sftpgo:latest
    container_name: sftpgo
    environment:
      - PUID=1002
      - PGID=1002
      - TZ=Australia/West
    volumes:
      - /mnt/data:/srv/sftpgo
      - /opt/docker/sftpgo/data:/var/lib/sftpd
    ports:
      - 8000:8080
      - 2022:2022
    restart: unless-stopped

```

```bash
docker compose -f sftpgo.yaml up -d
```

#### Application Setup¶

http://192.168.0.250:8000/web/admin/setup

Create the admin user

admin

qazwsxedc

Create a user



Create a virtual folder



### Watchtower

Watchtower is an open-source tool that automates the process of updating Docker containers. It operates by polling the Docker registry to check for updates to the images from which containers were initially instantiated.

Suppose Watchtower detects that an image has been updated. In that case, it gracefully shuts down the container running the outdated image, pulls the new image from the Docker registry, and starts a new container with the same configurations as the previous one. This ensures that our containers are always running the latest version of the base image without manual intervention.

Furthermore, Watchtower can monitor all containers on a host or only those explicitly specified, providing flexibility in how we manage updates across different environments.



compose.yaml

```yaml
---
services:
    watchtower:
        image: nickfedor/watchtower:latest
        container_name: watchtower
        environment:
            - TZ=Australia/West
            - WATCHTOWER_CLEANUP=true
            - WATCHTOWER_SCHEDULE=0 0 4 * * *
        volumes:
            - /var/run/docker.sock:/var/run/docker.sock
        ports:
            - 8060:8080
        restart: unless-stopped
```

**Configuration**

Defines when and how often Watchtower checks for new images using a 6-field [Cron expression](https://pkg.go.dev/github.com/robfig/cron@v1.2.0?tab=doc#hdr-CRON_Expression_Format).

Example: - WATCHTOWER_SCHEDULE=0 0 4 * * * runs daily at 4:00 am







## Install Apps



And for the apps we install them into /opt and keep their "user" data here also

```
sudo mkdir -p /opt/MediaCentre/userdata
```

The app will be installed directly under /opt.  We will create those directories as we install the app.

Now install some prerequisite packages that are required by all

```
sudo apt update
sudo apt install curl sqlite3
```



## Mono

Mono is a free and open-source software framework that aims to run software made for the.NET Framework on Linux (and other OSes).

The install needs mono-complete as the more basic mono-runtime isn’t enough.

```
sudo apt install apt-transport-https dirmngr gnupg ca-certificates
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 3FA7E0328081BFF6A14DA29AA6A19B38D3D831EF
echo "deb https://download.mono-project.com/repo/debian stable-trixie main" | sudo tee /etc/apt/sources.list.d/mono-official-stable.list
sudo apt update
sudo apt install mono-complete.
```



## MediaInfo

**MediaInfo** is a convenient unified display of the most relevant technical and tag data for video and audio files.

Go here and check the below commands have the latest version and update as necessary - https://mediaarea.net/en/Repos#deb

```
wget https://mediaarea.net/repo/deb/repo-mediaarea_1.0-25_all.deb
sudo dpkg -i repo-mediaarea_1.0-25_all.deb
sudo apt update
sudo apt install mediainfo
```



## bittorrent client

i like deluge - but it has been giving me issues on my Pi so for now we are using qBittorrent

### qBittorrent

We are installing  a command-line version of qBittorrent;  **qbittorrent-nox** - which is ideal for running a headless environment.

```
sudo apt install qbittorrent-nox
```

Let's start qBittorrent to make sure all is well

```
qbittorrent-nox
```

We see will this:

```
*** Legal Notice ***
qBittorrent is a file sharing program. When you run a torrent, its data will be made available to others by means of upload. Any content you share is your sole responsibility.

No further notices will be issued.

Press 'y' key to accept and continue...
y
WebUI will be started shortly after internal preparations. Please wait...

******** Information ********
To control qBittorrent, access the WebUI at: http://localhost:8080

The Web UI administrator username is: admin
The Web UI administrator password has not been changed from the default: adminadmin
This is a security risk, please change your password in program preferences.
```

Press **Ctrl+C** to quit the program 

#### qBittorrent service

We will create a service that will run and manage the qBittorrent client.

Firstly we need a user

```shell
sudo adduser qbittorrent --ingroup media --shell /usr/sbin/nologin
```



Now create the service file

```shell
sudo vi /etc/systemd/system/qbittorrent.service
```

And add this...

```
[Unit]
Description=qBittorrent
After=network.target

[Service]
Type=forking
User=media
Group=media
UMask=002
ExecStart=/usr/bin/qbittorrent-nox -d --webui-port=8080
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

What all this means

The qBittorrent software will be run under the user `qbittorrent` and the group `media` that we created earlier.

We apply a “`UMask`” to the user so that all files created by this user will hold the following Linux permission, “`rw-rw-r--`“.  This permission means that the owner and group can read and write all files created by this user.

We start the qBittorrent executable `/usr/bin/qbittorrent-nox` using it full path and with:

- `-d`, which tells the program we want to run it as a “daemon”.
- `--webui-port`” gives the port we will connect to with our browser for the web interface

Now start the service with

```shell
sudo systemctl start qbittorrent
```

Now start the service with

```shell
sudo systemctl status qbittorrent
```

You should see the service is:

```
     Active: active (running) since Sat 2025-10-11 10:06:18 AWST; 8s ago
```

as well as some other information.

Now enable the service with

```shell
sudo systemctl enable qbittorrent
```

Which means the service will start whenever the Pi starts.

#### qBittorrent Web Interface

This is how we manage the application

Open a browser and enter

```http
http://<the pi ip address>:8080
```





### deluge





### transmission



## Provider

Need either Jackett or Prowlarr - I have been having Cloudflare issues with Prowlarr 

### Jackett

https://varhowto.com/install-jackett-ubuntu-20-04/#Install_monodevel_package

Firstly we need a user

```shell
sudo adduser jackett --home /opt/MediaCentre/Userdata/Jackett --ingroup media --comment "Jackett User" --disabled-password
```

We are going to run the rest of this section as the jackett user - 

```
sudo su jackett
cd /opt/MediaCentre/Userdata/Jackett
```

Go to github here; 

https://github.com/Jackett/Jackett/releases

Copy the link (right click) for  `Jackett.Binaries.LinuxARM64.tar.gz` and paste it into a wget command 

```shell
wget https://github.com/Jackett/Jackett/releases/download/v0.24.101/Jackett.Binaries.LinuxARM64.tar.gz
```

Now uncompress the file we just downloaded

```
cd ..
tar xvzf Jackett/Jackett.Binaries.LinuxARM64.tar.gz
```

Note: we have changed directories so the files are extracted to /opt/MediaCentre/Userdata/Jackett rather than a Jackett directory within that directory.

Now change back into the Jackett directory and start jacket by entering`./jackett`

```shell
cd Jackett
./jackett
```

We will see

```
10-11 10:53:36 Info Starting Jackett v0.24.101
...
Now listening on: http://[::]:9117
```

Open a browser and enter

```http
http://<the pi ip address>:9117
```

To bring up the Jackett web interface

All okay?

Ctrl+d (or exit) to return to our usual login id

```shell
jackett@pugwash:~ $ exit
exit
jon@pugwash:~
```

#####  Jackett service

The developers have done a nice job of creating a service script for Jackett, so we can run this to create it.

```shell
sudo /opt/MediaCentre/Userdata/Jackett/install_service_systemd.sh
```

There is one change we need to make - as the script thinks we have a jackett group

```
sudo vi /etc/systemd/system/jackett.service
```

and change the Group line from jackett to media

```
[Unit]
Description=Jackett Daemon
After=network.target

[Service]
SyslogIdentifier=jackett
Restart=always
RestartSec=5
Type=simple
User=jackett
Group=media
WorkingDirectory=/opt/MediaCentre/Userdata/Jackett
Environment="DOTNET_EnableDiagnostics=0"
ExecStart=/bin/sh "/opt/MediaCentre/Userdata/Jackett/jackett_launcher.sh"
TimeoutStopSec=30

[Install]
WantedBy=multi-user.target
```

Then

```
sudo systemctl daemon-reload
sudo systemctl start jackett
sudo systemctl status jackett
```

All we should be up and running



Back in our browser and enter

```http
http://<the pi ip address>:9117
```

To bring up the Jackett web interface

Add some indexers...

I usually add these but YMMV

- 1337x 
- EZTV 
- kickasstorrents.ws 
- LimeTorrents 
- The Pirate Bay 







### Prowlarr

```
sudo adduser prowlarr --home /opt/Prowlaar -g media --no-user-group --shell /usr/sbin/nologin
```



Download them

```
sudo wget --content-disposition 'http://prowlarr.servarr.com/v1/update/master/updatefile?os=linux&runtime=netcore&arch=arm64'
```



```
sudo mkdir /var/lib/prowlarr && sudo chown prowlarr:media /var/lib/prowlarr

```





```
sudo vi /etc/systemd/system/prowlarr.service
```

with the following content

```
[Unit]
Description=Prowlarr Daemon
After=syslog.target network.target

[Service]
User=prowlarr
Group=media
Type=simple

ExecStart=/opt/Prowlarr/Prowlarr -nobrowser -data=/var/lib/prowlarr/
TimeoutStopSec=20
KillMode=process
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

- Reload systemd:

```shell
sudo systemctl -q daemon-reload
```

- Enable the Prowlarr service:

```shell
sudo systemctl enable prowlarr
sudo systemctl start prowlarr
sudo systemctl status prowlarr
```

- (Optional) Remove the tarball:

```shell
rm Prowlarr*.linux*.tar.gz
```

Typically to access the Prowlarr web GUI browse to `http://{Your server IP Address}:9696`

If Prowlarr did not appear to start, then check the status of the service:

```shell
sudo journalctl --since today -u prowlarr
```

------

### 





## Sonarr



https://sonarr.tv/#downloads-linux-debian

```bash
wget -qO- https://raw.githubusercontent.com/Sonarr/Sonarr/develop/distribution/debian/install.sh | sudo bash
```





## Jellyfin

**Jellyfin** is a Free Software Media System is an alternative to the  proprietary Emby and Plex, to provide media from a dedicated server to  end-user devices via multiple apps.

Instructions from Jellyfin here: https://jellyfin.org/docs/general/installation/linux



If you feel trustworthy run this:

```sh
curl https://repo.jellyfin.org/install-debuntu.sh | sudo bash
```



Or, if you want to know what is going on...

```sh
diff <( curl -s https://repo.jellyfin.org/install-debuntu.sh -o install-debuntu.sh; sha256sum install-debuntu.sh ) <( curl -s https://repo.jellyfin.org/install-debuntu.sh.sha256sum )
```

An empty output means everything is correct. Then you can inspect the  script to see what it does: 

```sh
less install-debuntu.sh
```

 and execute it  with:

```sh
sudo bash install-debuntu.sh
```









## qBittorrent

:warning: **NOTE:** qBittoerrent isnt installing how i like it with docker so.....

Saving this for now - coz i might tray again later

The [Qbittorrent](https://www.qbittorrent.org/) project aims  to provide an open-source software alternative to µTorrent. qBittorrent  is based on the Qt toolkit and libtorrent-rasterbar library.

https://docs.linuxserver.io/images/docker-qbittorrent/

```bash
# create the directory for our app
mkdir qbittorrent
# go there
cd qbittorrent
# create out docker compose file
vi qbittorrent.yaml
```

Paste the following into the file

```yaml
---
services:
  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    environment:
      - PUID=1001
      - PGID=1001
      - TZ=Australia/West
      - WEBUI_PORT=8080
      - TORRENTING_PORT=6881
    volumes:
      - /opt/container/qbittorrent/appdata:/config
      - /mnt/data/Torrent/Download:/downloads
    ports:
      - 8080:8080
      - 6881:6881
      - 6881:6881/udp
    restart: unless-stopped
```

:memo: **Note:** Make sure the PGUID, PGID and directories are correct.

Now start the application with:

```bash
 docker compose -f qbittorrent.yaml up -d
```

We will now see messages as docker downloads the image

Once this is finished you can run `docker ps` which will display something like this

```
docker ps
CONTAINER ID   IMAGE                                    COMMAND   CREATED              STATUS              PORTS                                                                                                                                   NAMES
e55d008f9ed3   lscr.io/linuxserver/qbittorrent:latest   "/init"   About a minute ago   Up About a minute   0.0.0.0:6881->6881/tcp, [::]:6881->6881/tcp, 0.0.0.0:8080->8080/tcp, 0.0.0.0:6881->6881/udp, [::]:8080->8080/tcp, [::]:6881->6881/udp   qbittorrent
```



### Application setup

The web UI is at `<your-ip>:8080` and a temporary password for the `admin` user will be printed to the container log on startup.

You must then change username/password in the web UI section of settings.  If you do not change the password a new one will be generated every time the container starts.

Enter this docker command

```bash
docker logs qbittorrent
```

Look for a line like this and note the password

```
The WebUI administrator username is: admin
The WebUI administrator password was not set. A temporary password is provided for this session: vQMgnGqC6
```

now connect to the web client with 

http://192.168.0.250:8080





After you login you should see this

![qbittorrent-home](./image/qbittorrent-home.png)

Click on Tools -> Options (or the options icon) 

![qbittorrent-options](./image/qbittorrent-options.png)

Click on the **WebUI** tab

Scroll down to **Authentication** and enter the password we noted earlier

Now scroll to the bottom of the page and click save







## Stuff

https://gist.github.com/rxaviers/7360908



```
> :warning: **Warning:** Do not push the big red button.

> :memo: **Note:** Sunrises are beautiful.

> :bulb: **Tip:** Remember to appreciate the little things in life.
```

:warning: **Warning:** Do not push the big red button.

:memo: **Note:** Sunrises are beautiful.

:bulb: **Tip:** Remember to appreciate the little things in life.





Installing the torrenting stuff .

This is wriiten from a point of view of a Raspberry Pi running Raspberry Pi OS - which is based on Debian. At the time of this the install is based on **Debian 13 “Trixie”**. Some of the sources may need to be changed if you are running a newer (or older) version of the OS - or a different OS.

I user vi as my editor of choice. It is the best editor ever made - but if you are a weakling then you might want to substitute nano for vi.

Apps

OpenVPN - keep all that we do private

qBittorrent - 

Jellyfin - Local media streamer

Immich - Photo management

Nextcloud - Web-based file management



KeePassXC or Vaultwarden or Passbolt



??? - backup software

Other "docker" apps to consider

Syncthing - peer-to-peer file syncronisation



Home Assistant - home automation

Uptime Kuma - monitor services and apps

Portainer - managing containers

Pi-hole - ad-block and IOT monitor





## Loading books into Calibre

cd tmp

ssh pugwash "ls -lt '/mnt/data/Torrent/Download' | head"

rsync -vr 'pugwash:/mnt/data/Torrent/Download/The Farseer Trilogy by Robin Hobb' .





