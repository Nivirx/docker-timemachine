### docker-timemachine

This is a quick and dirty Time Machine setup using Docker compose, [Avahi](https://git.chtm.us/nivirx/docker-avahi/), and [Samba](https://git.chtm.us/nivirx/docker-samba/)

***I am working on updating and consolidating all three projects into one build process***

## How it works

Two services are needed on the Linux server:

- Samba, which hosts file shares on a network, and is configured to support AAPL SMB extentions
- Avahi, which advertises the network shares on the network to allow automatic discovery

This project uses my Avahi Docker container on [gitea](https://git.chtm.us/nivirx/docker-avahi/), and the Samba Docker container on [gitea](https://git.chtm.us/nivirx/docker-samba/), the latter of which comes pre-configured with the Apple extensions for use with Time Machine.

We'll assume a user `apple` with password `timemachine`, and that the backups will be stored in `/mnt/tm/data`, but you can change those details as needed. Make sure the filesystem you want to use is mounted at `/mnt/tm/data` before running `docker compose`.

# Getting docker compose setup
>*Some commands are prefixed here with sudo, but if you're logged into the Linux box as the root user, the sudo prefix can be removed.*

Install docker and docker-compose and git (optional) using your package manager. For example, on Debian or Ubuntu:
```bash
    sudo apt install docker.io docker-compose git
```

Create a new directory to store the configuration and backups. Here, we'll store the configuration and backup data in two subdirectories, `config` and `data`:
```bash
    mkdir -p /mnt/tm/{config,data}
    cd /mnt
```

==Or if you would like a pre-configured setup==
```bash
    git clone https://git.chtm.us/Nivirx/docker-timemachine.git tm
```

Edit the file called docker-compose.yml in the /mnt/tm/config directory with the following contents.
you will need to update the image name's, TZ, and can set the directory to use for TM backups, and the Samba username and password here.
>if building localy, you will use local/docker-avahi:[tag] & local/docker-samba:[tag]
```yml
    version: '3.4'

    services:
     avahi:
       container_name: avahi
       image: nivirx/docker-avahi:0.9
       network_mode: host
       volumes:
         - ./avahi:/etc/avahi:ro
       restart: unless-stopped
     samba:
       container_name: samba
       image: nivirx/docker-samba:0.3
       environment:
         TZ: 'America/New_York'
       networks:
         - default
       ports:
         - "137:137/udp"
         - "138:138/udp"
         - "139:139/tcp"
         - "445:445/tcp"
       read_only: true
       tmpfs:
         - /tmp
       restart: unless-stopped
       stdin_open: true
       tty: true
       volumes:
         - /mnt/tm/data:/backup:z
       command: '-s "Time Machine Backup;/backup;yes;no" -u "apple;timemachine"'
```

# Avahi configuration
==*not required if you are using the config files from the git repo under config/avahi/*==

---
Initialise the Avahi configuration files
```bash
    sudo docker create --name avahi-config nivirx/docker-avahi:0.8
    sudo docker cp avahi-config:/etc/avahi .
    sudo docker rm avahi-config
```
Disable DBUS on Avahi (DBUS is not needed)
```bash
    sed -i 's/#enable-dbus=yes/enable-dbus=no/' avahi/avahi-daemon.conf
```
Create Avahi configuration to advertise Samba on the network
```bash
    cat <<EOT >> avahi/services/smb.conf
    <?xml version="1.0" standalone='no'?>
    <!DOCTYPE service-group SYSTEM "avahi-service.dtd">
    <service-group>
    <name replace-wildcards="yes">%h</name>
    <service>
      <type>_adisk._tcp</type>
      <txt-record>sys=waMa=0,adVF=0x100</txt-record>
      <txt-record>dk0=adVN=Time Capsule,adVF=0x82</txt-record>
    </service>
    <service>
      <type>_smb._tcp</type>
      <port>445</port>
    </service>
    <service>
      <type>_device-info._tcp</type>
      <port>0</port>
      <txt-record>model=RackMac</txt-record>
    </service>
    </service-group>
    EOT
```

---

# Finishing up
Mount your perfered storage on the mount point specified in the configuration. `/mnt/tm/data` if you have been following the guide.

Start the Samba and Avahi services
```bash
    sudo docker-compose up -d
```
---
## Mac configuration

On the Mac, you should now see the Linux machine in the sidebar of Finder.

- Samba network share in Finder

- To configure the Mac to use the Time Machine backup, go to System Preferences, then Time Machine. Click Select Disk, and choose the name of the Linux machine.

- Select disk in Time Machine


#### *Bonus: altering the Time Machine schedule*

By default, Time Machine backs up hourly. This might be too frequent for your needs. Personally, I prefer to do a daily overnight backup. A free utility called Time Machine Editor can help you edit this configuration.

Time Machine Editor allows custom schedules

To enable the Mac to backup from sleep, you'll have to make sure a feature called Power Nap is enabled. From System Preferences, select Energy Saver, then the Power Adapter tab. Ensure the Power Nap feature is enabled. This allows the Mac to backup overnight even when it's asleep, provided it's connected to mains power. You'll have to make sure the Linux machine is awake, too.

Ensure Power Nap is enabled to wake Mac from sleep for backups