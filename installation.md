# Home Server Setup Guide
Windows + VirtualBox + Ubuntu + Docker + Nextcloud + Jellyfin

This document describes the complete setup of a personal home server environment.

Host Machine
Windows 11

Virtualization
Oracle VirtualBox

Guest OS
Ubuntu Server

Services
Docker
Nextcloud
MariaDB
Redis
Portainer
Jellyfin

Storage
Windows D:\ drive shared to Ubuntu VM via CIFS


------------------------------------------------------------
SYSTEM ARCHITECTURE
------------------------------------------------------------

Windows 11 Host
│
├─ Oracle VirtualBox
│   └─ Ubuntu Server VM
│        └─ Docker
│             ├─ Nextcloud
│             ├─ MariaDB
│             ├─ Redis
│             ├─ Portainer
│             └─ Jellyfin
│
└─ Windows Storage (D:)
      ├─ Movies
      ├─ Audio
      ├─ Documents
      ├─ Personal_Photos
      ├─ NextcloudData
      └─ NextcloudBackups


------------------------------------------------------------
USERNAMES AND PASSWORDS
------------------------------------------------------------

Ubuntu Server
username: my_homeserver
password: created during installation

Windows Share
username: nasuser
password: Cisco@430

Nextcloud
username: kalyan
password: Vadhulasa@430

Database
user: nextcloud
password: nextcloudpassword


------------------------------------------------------------
PHASE 1 — INSTALL VIRTUALBOX
------------------------------------------------------------

Download VirtualBox

https://www.virtualbox.org/wiki/Downloads

Install
VirtualBox
Extension Pack

Windows may show warning

"This program might not have installed correctly"

Select

This program installed correctly


------------------------------------------------------------
PHASE 2 — CREATE VIRTUAL MACHINE
------------------------------------------------------------

Open VirtualBox
Click NEW

Name
HomeServer

Type
Linux

Version
Ubuntu (64-bit)

RAM
8192 MB

CPU
4 cores

Disk
120 GB

Disk Type
VDI

Allocation
Dynamically allocated


------------------------------------------------------------
PHASE 3 — NETWORK CONFIGURATION
------------------------------------------------------------

VirtualBox → Settings → Network

Adapter 1

Attached to
Bridged Adapter

Select actual adapter
Example

Intel WiFi Adapter


------------------------------------------------------------
PHASE 4 — INSTALL UBUNTU SERVER
------------------------------------------------------------

Download ISO

https://ubuntu.com/download/server

Attach ISO to VM
Start VM


------------------------------------------------------------
UBUNTU INSTALL SETTINGS
------------------------------------------------------------

Language
English

Storage
Use entire disk
Enable LVM

Proxy
Leave blank

Ubuntu Pro
Skip

SSH
Install OpenSSH server

Snaps
Skip all


------------------------------------------------------------
PHASE 5 — FIND SERVER IP
------------------------------------------------------------

ip a

Example output

192.168.0.189


------------------------------------------------------------
PHASE 6 — SSH ACCESS
------------------------------------------------------------

ssh my_homeserver@192.168.0.189


------------------------------------------------------------
PHASE 7 — FIX APT MIRROR ISSUE
------------------------------------------------------------

sudo nano /etc/apt/sources.list

Use mirror

archive.ubuntu.com

Then

sudo apt update


------------------------------------------------------------
PHASE 8 — INSTALL DOCKER
------------------------------------------------------------

curl -fsSL https://get.docker.com | sudo sh

Verify

docker ps

Add user to docker group

sudo usermod -aG docker $USER

Logout and login again


------------------------------------------------------------
INSTALL DOCKER COMPOSE
------------------------------------------------------------

sudo apt install docker-compose-plugin


------------------------------------------------------------
PHASE 9 — WINDOWS STORAGE SETUP
------------------------------------------------------------

Create folders on Windows

D:\

Movies
Audio
Documents
Personal_Photos
NextcloudData
NextcloudBackups


------------------------------------------------------------
SHARE WINDOWS DRIVE
------------------------------------------------------------

Right click D drive
Properties
Sharing

Create user

nasuser

Grant full access


------------------------------------------------------------
PHASE 10 — MOUNT WINDOWS DRIVE IN UBUNTU
------------------------------------------------------------

Install CIFS tools

sudo apt install cifs-utils

Create mount directory

mkdir ~/storage

Mount share

sudo mount -t cifs //192.168.0.59/DDrive ~/storage \
-o username=nasuser,password=Cisco@430,uid=1000,gid=1000

Verify mount

ls ~/storage


------------------------------------------------------------
MAKE MOUNT PERMANENT
------------------------------------------------------------

sudo nano /etc/fstab

Add line

//192.168.0.59/DDrive /home/my_homeserver/storage cifs username=nasuser,password=Cisco@430,uid=1000,gid=1000 0 0

Apply mount

sudo mount -a


------------------------------------------------------------
PHASE 11 — DEPLOY NEXTCLOUD
------------------------------------------------------------

mkdir -p ~/stacks/nextcloud
cd ~/stacks/nextcloud

nano docker-compose.yml


services:

 db:
  image: mariadb:11
  restart: always
  environment:
   MYSQL_ROOT_PASSWORD: rootpassword
   MYSQL_DATABASE: nextcloud
   MYSQL_USER: nextcloud
   MYSQL_PASSWORD: nextcloudpassword
  volumes:
   - db:/var/lib/mysql

 redis:
  image: redis:7
  restart: always

 app:
  image: nextcloud:apache
  restart: always
  ports:
   - 8081:80
  volumes:
   - /home/my_homeserver/storage/NextcloudData:/var/www/html/data
  environment:
   MYSQL_HOST: db
   MYSQL_DATABASE: nextcloud
   MYSQL_USER: nextcloud
   MYSQL_PASSWORD: nextcloudpassword

volumes:
 db:


Start containers

docker compose up -d


------------------------------------------------------------
ACCESS NEXTCLOUD
------------------------------------------------------------

http://192.168.0.189:8081

Create account

username: kalyan
password: Vadhulasa@430


------------------------------------------------------------
ERROR FIX — DATA DIRECTORY NOT WRITABLE
------------------------------------------------------------

sudo chown -R 33:33 ~/storage/NextcloudData
sudo chmod -R 770 ~/storage/NextcloudData


------------------------------------------------------------
ERROR FIX — DATA DIRECTORY READABLE BY OTHERS
------------------------------------------------------------

sudo chmod 770 ~/storage/NextcloudData


------------------------------------------------------------
RESET NEXTCLOUD PASSWORD
------------------------------------------------------------

docker exec -it nextcloud-app php occ user:list

docker exec -it nextcloud-app php occ user:resetpassword kalyan


------------------------------------------------------------
INSTALL PORTAINER
------------------------------------------------------------

docker volume create portainer_data

docker run -d \
-p 9000:9000 \
--name portainer \
--restart=always \
-v /var/run/docker.sock:/var/run/docker.sock \
-v portainer_data:/data \
portainer/portainer-ce:latest

Access

http://192.168.0.189:9000


------------------------------------------------------------
INSTALL JELLYFIN
------------------------------------------------------------

mkdir -p ~/stacks/jellyfin
cd ~/stacks/jellyfin

nano docker-compose.yml


services:
  jellyfin:
    image: jellyfin/jellyfin:latest
    container_name: jellyfin
    restart: unless-stopped
    ports:
      - "8096:8096"
    volumes:
      - /home/my_homeserver/docker/jellyfin/config:/config
      - /home/my_homeserver/docker/jellyfin/cache:/cache
      - /home/my_homeserver/storage/Movies:/media/movies
      - /home/my_homeserver/storage/Audio:/media/audio
      - /home/my_homeserver/storage/Personal_Photos:/media/photos


Start container

docker compose up -d


------------------------------------------------------------
ACCESS JELLYFIN
------------------------------------------------------------

http://192.168.0.189:8096

Create account

username: kalyan
password: Vadhulasa@430


------------------------------------------------------------
JELLYFIN LIBRARIES
------------------------------------------------------------

Movies

/media/movies

Music

/media/audio

Photos

/media/photos


------------------------------------------------------------
MOVIE FOLDER FORMAT
------------------------------------------------------------

D:\Movies
   ├── Interstellar (2014)
   │     └── Interstellar (2014).mkv

   ├── Oppenheimer (2023)
   │     └── Oppenheimer (2023).mkv


------------------------------------------------------------
SONY TV JELLYFIN SETUP
------------------------------------------------------------

Open Google Play Store on Sony TV

Install

Jellyfin for Android TV

Open app

Enter server

http://192.168.0.189:8096

Login

username: kalyan
password: Vadhulasa@430


------------------------------------------------------------
AUTO START VM
------------------------------------------------------------

Win + R

shell:startup

Create shortcut

"C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" startvm "HomeServer" --type headless


------------------------------------------------------------
GRACEFUL VM SHUTDOWN
------------------------------------------------------------

Create script

C:\Scripts\shutdown_vm.bat

"C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" controlvm "HomeServer" acpipowerbutton
timeout /t 20

Create Task Scheduler job

Trigger
Event ID 1074

Action
Run shutdown_vm.bat


------------------------------------------------------------
SERVER STARTUP FLOW
------------------------------------------------------------

Windows boot
↓
Startup shortcut
↓
VirtualBox VM starts
↓
Ubuntu boots
↓
Docker containers start
↓
Nextcloud + Jellyfin available


------------------------------------------------------------
SERVER SHUTDOWN FLOW
------------------------------------------------------------

Windows shutdown
↓
Task Scheduler triggers script
↓
VirtualBox sends ACPI signal
↓
Ubuntu shuts down
↓
Containers stop


------------------------------------------------------------
ACCESS URLS
------------------------------------------------------------

Nextcloud

http://192.168.0.189:8081

Portainer

http://192.168.0.189:9000

Jellyfin

http://192.168.0.189:8096


------------------------------------------------------------
------------------------------------------------------------
PHASE 4 — MEDIA AUTOMATION STACK
------------------------------------------------------------

The goal of Phase 4 is to automate movie downloads and make them appear automatically inside Jellyfin.

Stack Components

qBittorrent
Download client

Prowlarr
Indexer manager

Radarr
Movie automation manager

Overseerr
Movie request interface

Jellyfin
Media server (already installed)


------------------------------------------------------------
MEDIA AUTOMATION ARCHITECTURE
------------------------------------------------------------

User
 ↓
Overseerr (movie request UI)
 ↓
Radarr
 ↓
Prowlarr (search indexers)
 ↓
qBittorrent (downloads movie)
 ↓
Windows Storage D:\Downloads
 ↓
Radarr moves file to D:\Movies
 ↓
Jellyfin scans library
 ↓
Movie appears in Jellyfin automatically


------------------------------------------------------------
WINDOWS STORAGE STRUCTURE
------------------------------------------------------------

D:\

Movies
Audio
Documents
Personal_Photos
NextcloudData
NextcloudBackups
Downloads


------------------------------------------------------------
LINUX STORAGE MAPPING
------------------------------------------------------------

Windows

D:\Movies
D:\Downloads
D:\Audio
D:\Personal_Photos

Ubuntu

/home/my_homeserver/storage/Movies
/home/my_homeserver/storage/Downloads
/home/my_homeserver/storage/Audio
/home/my_homeserver/storage/Personal_Photos


------------------------------------------------------------
PHASE 4.1 — INSTALL QBITTORRENT
------------------------------------------------------------

Create stack folder

mkdir -p ~/stacks/qbittorrent
cd ~/stacks/qbittorrent

Create compose file

nano docker-compose.yml


services:
  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Asia/Kolkata
      - WEBUI_PORT=8080
    ports:
      - "8080:8080"
      - "6881:6881"
      - "6881:6881/udp"
    volumes:
      - /home/my_homeserver/docker/qbittorrent/config:/config
      - /home/my_homeserver/storage/Downloads:/downloads


Start container

docker compose up -d


Verify container

docker ps


------------------------------------------------------------
ACCESS QBITTORRENT
------------------------------------------------------------

URL

http://192.168.0.189:8080


Default login

Username
admin

Password
adminadmin

If password is different check logs

docker logs qbittorrent


------------------------------------------------------------
CHANGE QBITTORRENT PASSWORD
------------------------------------------------------------

Open

Settings → Web UI

Set new credentials


Username
kalyan

Password
Vadhulasa@430


------------------------------------------------------------
SET DOWNLOAD LOCATION
------------------------------------------------------------

Open

Settings → Downloads

Default Save Path

/downloads


This maps to

D:\Downloads


------------------------------------------------------------
CREATE CATEGORY FOR RADARR
------------------------------------------------------------

Inside qBittorrent

Categories → Add

Name

radarr


------------------------------------------------------------
DOWNLOAD FLOW
------------------------------------------------------------

Radarr sends job to qBittorrent
 ↓
qBittorrent downloads to

/downloads

 ↓

Windows folder

D:\Downloads

 ↓

Radarr moves file to

D:\Movies

 ↓

Jellyfin scans library

 ↓

Movie appears in Jellyfin


------------------------------------------------------------
QBITTORRENT PORTS
------------------------------------------------------------

Web UI

http://192.168.0.189:8080

BitTorrent

6881 TCP
6881 UDP


------------------------------------------------------------
PHASE 4.2 — INSTALL PROWLARR
------------------------------------------------------------

Purpose

Manage torrent indexers
Send indexers to Radarr


Create stack folder

mkdir -p ~/stacks/prowlarr
cd ~/stacks/prowlarr


Create compose file

nano docker-compose.yml


services:
  prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Asia/Kolkata
    ports:
      - "9696:9696"
    volumes:
      - /home/my_homeserver/docker/prowlarr/config:/config


Start container

docker compose up -d


Access

http://192.168.0.189:9696


------------------------------------------------------------
PHASE 4.3 — INSTALL RADARR
------------------------------------------------------------

Purpose

Automatically download and organize movies


Create stack folder

mkdir -p ~/stacks/radarr
cd ~/stacks/radarr


Create compose file

nano docker-compose.yml


services:
  radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Asia/Kolkata
    ports:
      - "7878:7878"
    volumes:
      - /home/my_homeserver/docker/radarr/config:/config
      - /home/my_homeserver/storage/Movies:/movies
      - /home/my_homeserver/storage/Downloads:/downloads


Start container

docker compose up -d


Access

http://192.168.0.189:7878


------------------------------------------------------------
CONNECT RADARR TO QBITTORRENT
------------------------------------------------------------

Settings → Download Clients

Add

qBittorrent


Host

192.168.0.189

Port

8080

Username

kalyan

Password

Vadhulasa@430

Category

radarr


------------------------------------------------------------
PHASE 4.4 — INSTALL OVERSEERR
------------------------------------------------------------

Purpose

User interface for requesting movies


Create stack folder

mkdir -p ~/stacks/overseerr
cd ~/stacks/overseerr


Create compose file

nano docker-compose.yml


services:
  overseerr:
    image: sctx/overseerr:latest
    container_name: overseerr
    restart: unless-stopped
    environment:
      - TZ=Asia/Kolkata
    ports:
      - "5055:5055"
    volumes:
      - /home/my_homeserver/docker/overseerr/config:/app/config


Start container

docker compose up -d


Access

http://192.168.0.189:5055


------------------------------------------------------------
CONNECT OVERSEERR
------------------------------------------------------------

Media Server

Jellyfin

URL

http://192.168.0.189:8096

User

kalyan

Password

Vadhulasa@430


Add Radarr

URL

http://192.168.0.189:7878


------------------------------------------------------------
FINAL AUTOMATION FLOW
------------------------------------------------------------

User searches movie in Overseerr
 ↓
Overseerr sends request to Radarr
 ↓
Radarr searches indexers via Prowlarr
 ↓
Radarr sends download to qBittorrent
 ↓
qBittorrent downloads to D:\Downloads
 ↓
Radarr moves movie to D:\Movies
 ↓
Jellyfin scans library
 ↓
Movie appears automatically


------------------------------------------------------------
SERVICE URL SUMMARY
------------------------------------------------------------

Nextcloud

http://192.168.0.189:8081


Portainer

http://192.168.0.189:9000


Jellyfin

http://192.168.0.189:8096


qBittorrent

http://192.168.0.189:8080


Prowlarr

http://192.168.0.189:9696


Radarr

http://192.168.0.189:7878


Overseerr

http://192.168.0.189:5055
-------------------------------------------
