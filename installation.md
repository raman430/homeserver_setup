# Home Server Setup Guide
## Windows + VirtualBox + Ubuntu + Docker + Nextcloud

This document describes the complete setup of a personal home server environment.

Host Machine:
Windows 11

Virtualization:
Oracle VirtualBox

Guest OS:
Ubuntu Server

Services:
Docker
Nextcloud
MariaDB
Redis

Storage:
Windows D:\ drive shared to Ubuntu VM via CIFS.

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
│             └─ Redis
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
password: <created during install>

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

Install:
VirtualBox
Extension Pack

Windows may show warning:

"This program might not have installed correctly"

Select:

This program installed correctly


------------------------------------------------------------
PHASE 2 — CREATE VIRTUAL MACHINE
------------------------------------------------------------

Open VirtualBox
Click NEW

Name:
HomeServer

Type:
Linux

Version:
Ubuntu (64-bit)

RAM:
8192 MB

CPU:
4 cores

Disk:
120 GB

Disk Type:
VDI

Allocation:
Dynamically allocated


------------------------------------------------------------
PHASE 3 — NETWORK CONFIGURATION
------------------------------------------------------------

VirtualBox → Settings → Network

Adapter 1

Attached to:
Bridged Adapter

Select actual adapter.

Example:

Intel WiFi 6 AX210


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

Language:
English

Storage:

Use entire disk
Enable LVM

Proxy:
Leave blank

Ubuntu Pro:
Skip

SSH:
Install OpenSSH server

Snaps:
Skip all


------------------------------------------------------------
PHASE 5 — FIND SERVER IP
------------------------------------------------------------

After installation run:

ip a

Example output:

192.168.0.189


------------------------------------------------------------
PHASE 6 — SSH ACCESS
------------------------------------------------------------

From Windows terminal:

ssh my_homeserver@192.168.0.189


------------------------------------------------------------
PHASE 7 — FIX APT MIRROR ISSUE
------------------------------------------------------------

During install mirrors sometimes fail.

Fix by editing:

sudo nano /etc/apt/sources.list

Use mirror:

archive.ubuntu.com

Update packages:

sudo apt update


------------------------------------------------------------
PHASE 8 — INSTALL DOCKER
------------------------------------------------------------

Install docker:

curl -fsSL https://get.docker.com | sudo sh

Verify docker:

docker ps

Add user to docker group:

sudo usermod -aG docker $USER

Logout and login again.


------------------------------------------------------------
INSTALL DOCKER COMPOSE
------------------------------------------------------------

sudo apt install docker-compose-plugin


------------------------------------------------------------
PHASE 9 — WINDOWS STORAGE SETUP
------------------------------------------------------------

Create folders on Windows:

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

Create user:

nasuser

Grant full access.


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

Edit fstab

sudo nano /etc/fstab

Add line:

//192.168.0.59/DDrive /home/my_homeserver/storage cifs username=nasuser,password=Cisco@430,uid=1000,gid=1000 0 0

Apply mount

sudo mount -a


------------------------------------------------------------
PHASE 11 — DEPLOY NEXTCLOUD
------------------------------------------------------------

Create stack folder

mkdir -p ~/stacks/nextcloud
cd ~/stacks/nextcloud

Create compose file

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

Verify

docker ps


------------------------------------------------------------
PHASE 12 — ACCESS NEXTCLOUD
------------------------------------------------------------

Open browser

http://192.168.0.189:8081

Create user

username: kalyan
password: Vadhulasa@430


------------------------------------------------------------
ERROR FIX — DATA DIRECTORY NOT WRITABLE
------------------------------------------------------------

Error seen:

Cannot write to data directory

Fix:

sudo chown -R 33:33 ~/storage/NextcloudData
sudo chmod -R 770 ~/storage/NextcloudData


------------------------------------------------------------
ERROR FIX — DATA DIRECTORY READABLE BY OTHERS
------------------------------------------------------------

Fix:

sudo chmod 770 ~/storage/NextcloudData


------------------------------------------------------------
RESET NEXTCLOUD PASSWORD
------------------------------------------------------------

List users

docker exec -it nextcloud-app php occ user:list

Reset password

docker exec -it nextcloud-app php occ user:resetpassword kalyan


------------------------------------------------------------
PHASE 13 — PHONE AUTO UPLOAD
------------------------------------------------------------

Install Nextcloud mobile app

Enable:

Settings
Auto Upload
Camera Folder

Photos stored in:

NextcloudData/kalyan/files


------------------------------------------------------------
PHASE 14 — AUTO START VM
------------------------------------------------------------

Press:

Win + R

Run:

shell:startup

Create shortcut

"C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" startvm "HomeServer" --type headless


------------------------------------------------------------
PHASE 15 — GRACEFUL VM SHUTDOWN
------------------------------------------------------------

Windows Home does not support gpedit.

Use Task Scheduler.


Create script

C:\Scripts\shutdown_vm.bat


"C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" controlvm "HomeServer" acpipowerbutton
timeout /t 20


Create task

Shutdown HomeServer VM

Trigger:

On Event

Log:
System

Source:
USER32

Event ID:
1074

Action:

C:\Scripts\shutdown_vm.bat


------------------------------------------------------------
SERVER STARTUP FLOW
------------------------------------------------------------

Windows boot
↓
Startup shortcut runs
↓
VirtualBox VM starts
↓
Ubuntu boots
↓
Docker starts containers
↓
Nextcloud available


------------------------------------------------------------
SERVER SHUTDOWN FLOW
------------------------------------------------------------

Windows shutdown
↓
Task Scheduler runs script
↓
VirtualBox sends ACPI signal
↓
Ubuntu shuts down
↓
Docker containers stop


------------------------------------------------------------
NEXT STEPS
------------------------------------------------------------

Install:

Jellyfin
Portainer
Pi-hole
Home Assistant
Automated backups


------------------------------------------------------------
SERVER ACCESS
------------------------------------------------------------

Nextcloud

http://192.168.0.189:8081


SSH

ssh my_homeserver@192.168.0.189
