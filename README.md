# Contents

1. [ Setting Up Raspberry Pi ](#setuppi)
2. [ Securing Raspberry Pi ](#securingpi)
3. [ Installing Git ](#installinggit)
4. [ Installing Docker and Docker Compose ](#installingdocker)
5. [ Pihole and Unbound with Docker Compose ](#piholeandunboundwithdockercompose)
6. [ Log2Ram ](#log2ram)

<a name="setuppi"></a>

# Setting Up Raspberry Pi

### Sources

https://www.raspberrypi.org/software/  
https://www.raspberrypi.org/documentation/installation/installing-images/README.md  
https://www.raspberrypi.org/documentation/remote-access/ssh/README.md  
https://www.raspberrypi.org/documentation/configuration/wireless/headless.md

### Preparing SD Card

On another device, download and use the [Raspberry Pi Imager](https://www.raspberrypi.org/software/) to write the RaspberryPi OS image onto a micro-SD card. The Lite version does not include the desktop GUI, so use that if you are setting the raspberry pi up headless.

To enable ssh, create empty file called `ssh` at root level on your SD card before the first boot.

I personally use ethernet, but if you want to use wifi, you can automatically connect to a wifi network on the first boot by adding a `wpa_supplicant.conf` file at root level before first boot with the following:

```
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1
country=<Insert 2 letter ISO 3166-1 country code here>

network={
 ssid="<Name of your wireless LAN>"
 psk="<Password for your wireless LAN>"
}
```

### First boot

Put the micro-SD card into the raspberrypi and power it on.

Wait a little for it to start up, then you can ssh into the raspberry pi using the default username "pi" and password "raspberry". Use your router or another network analyser tool (like Fing app) to find the IP address of the pi and run this command `ssh pi@<IP address>`.

### Change hostname

You can optionally change the hostname of your device as follows. Run `sudo raspi-config` then go to "System Options", then "Hostname". You can then type in your new hostname and reboot.

Alternatively, run `sudo nano /etc/hosts`, change the old raspberry pi hostname to your new one, save and exit by hitting Ctrl-X and the "Y" for yes. Then run `sudo nano /etc/hostname`, change the hostname there to the new one, save and exit. Finally reboot by running `sudo reboot`.

### Correctly shutting down the Raspberry Pi

To restart the pi, run `sudo reboot`.

When you want to shut down the pi, pulling power cord without properly shutting down the system increases the risk of corrupting the micro SD card. Anything running will also not save and exit gracefully. To properly shutdown, run `sudo shutdown -h now`. Give it a second to send SIGTERM, SIGKILL signals to all running processes, and unmount all file systems. Only when the system is halted can you pull the power to the device with minimised risk.

To start the Raspberry pi back up, simply turn on the power.

There are ways to create a power button using the GPIO pins on the board. // TODO: explore this

### Power Consumption Tweaks

If you are running a headless Raspberry Pi, then according to [this blog by Jeff Geering](https://www.jeffgeerling.com/blogs/jeff-geerling/raspberry-pi-zero-conserve-energy), you can save a little bit of power by disabling the HDMI display circuitry.

Run the command `/usr/bin/tvservice -o` to disable HDMI. Also run `sudo nano /etc/rc.local` and add the command there too in order to disable HDMI on boot.

(To enable again, run `/usr/bin/tvservice -p`, and remove from `/etc/rc.local`).

<a name="securingpi"></a>

# Securing Raspberry Pi

### Sources

https://www.raspberrypi.org/documentation/raspbian/updating.md  
https://www.raspberrypi.org/documentation/configuration/security.md  
https://www.youtube.com/watch?v=ukHcTCdOKrc

### Change password for pi user

Run `sudo raspi-config`, go to “System Options”, and then “Password”.

### Set up ssh

Either create empty file named “ssh” at root level on your SD card before the first boot, or run `sudo raspi-config`, then go to “Interface Options” and then “SSH”.

### Create a new administrative superuser account

Run `sudo adduser <account_name>`, then `sudo gpasswd -a <account_name> adm`, and then `sudo gpasswd -a <account_name> sudo`.

Finally, check the new user is capable of logging in to the raspberrypi using a new terminal and is able to use sudo by running `sudo whoami`.

### Lock the pi account

We could delete the pi accound instead of locking it, but some software still relies on the pi account to work.

Log onto the administrative superuser account set up in the previous step, then run `sudo passwd --lock pi`.

### Updating and upgrading rasp pi OS

Run `sudo apt update`, then `sudo apt full-upgrade -y`, and finally `sudo apt clean` to clean up the downloaded package files.

### Kill unnecessary system services

To reduce idling power usage, and also reduce the areas for compromise, we can list all running services and disable services you don't need - e.g. wifi, bluetooth or sound card drivers.

To see all active services, run `sudo systemctl --type=service --state=active`.

To disable the wifi service or the bluetooth service now, run `sudo systemctl disable --now wpa_supplicant.service` or `sudo systemctl disable --now bluetooth.service` respectively.

If you wanted to enable a service again, run `sudo systemctl enable --now bluetooth.service`.

### Restrict ssh accounts

Run `sudoedit /etc/ssh/sshd_config`.

Under the line “# Authentication”, add `AllowUsers <account_name1> <account_name2>`.

After the change, you will need to restart the sshd service using `sudo systemctl restart ssh` or rebooting.

### Firewall

Use ufw (uncomplicated firewall). Need to be careful not to lock yourself out. See links to rasp pi doc and YouTube video above.

### Brute-force detection

Use fail2ban which watchs system logs for repeated login attempts and add a firewall rule to prevent further access for a specified time. See links to rasp pi doc and YouTube video above.

### Automatic package update and upgrade

Not for production because of potential compatibility problems that may arise.

Use unattended-upgrades with raspberry pi specific config.
Alternatively, set up a cron job to run the update/upgrade commands.

See link to YouTube video above for more details.

<a name="installinggit"></a>

# Installing Git

### Sources

https://projects.raspberrypi.org/en/projects/getting-started-with-git/3

### Install git

Run `sudo apt install -y git`, then check the installation by running `git --version`.

### Configuring git

Set up your username and email by running:
`git config --global user.name "Harry Potter"`, then `git config --global user.email "h.potter@hogwarts.prof"`.

Check the config by running `git config --list`.

The configuration is stored in the `~/.gitconfig` file. Edit this directly, or via `git config` to make further changes if required.

You can also tell git what text editor you'd like to use, for example this sets it to nano: `git config --global core.editor nano`.

<a name="installingdocker"></a>

# Installing Docker and Docker Compose

### Sources

https://www.raspberrypi.org/blog/docker-comes-to-raspberry-pi/  
https://blog.alexellis.io/getting-started-with-docker-on-raspberry-pi/  
https://sanderh.dev/setup-Docker-and-Docker-Compose-on-Raspberry-Pi/  
https://dev.to/rohansawant/installing-docker-and-docker-compose-on-the-raspberry-pi-in-5-simple-steps-3mgl  
https://www.zuidwijk.com/blog/installing-docker-and-docker-compose-on-a-raspberry-pi-4/

### Install Docker

Run `curl -sSL https://get.docker.com | sh`.

Then add your user to the docker usergroup by running `sudo gpasswd -a pi docker` - replacing pi with whatever account you use docker on your raspberry pi. Logout using `logout` command and log back in to make sure the group setting is applied. (You can check that your username is in the right groups using this command `grep '<username>' /etc/group`.)

Now test docker is installed with `docker version`.

You can also check that your user can run a docker container by running the hello-world container, `docker run hello-world`. Afterwards, clean up by removing the container and the hello-world image by firstly getting the container id by running `docker ps -a`. Then force remove the container by running `docker rm -f <container id>` which will stop and remove it. Finally, remove the downloaded image by running `docker image rm hello-world`.

### Install Docker Compose

Run `sudo apt install -y libffi-dev libssl-dev python3 python3-pip`, then `sudo apt remove python-configparser`, and finally run `sudo pip3 -v install docker-compose`.

Reboot the pi by running `sudo reboot`.

### Dump of Common Docker Commands

`docker ps -a`  
`docker stop <container-id>`  
`docker image ls`  
`docker image rm <image-id>`  
`docker rm -f <container-id>`  
`docker exec -it <container-id> bash`  
`docker build -t <docker-username>/<image-name>`  
`docker push <docker-username>/<image-name>`

`docker-compose pull`  
`docker-compose down`  
`docker-compose up -d`

<a name="piholeandunboundwithdockercompose"></a>

# Pihole and Unbound with Docker Compose

### Sources

https://hub.docker.com/r/pihole/pihole/  
https://github.com/pi-hole/docker-pi-hole  
https://docs.pi-hole.net/guides/dns/unbound/
https://github.com/chriscrowe/docker-pihole-unbound  
https://github.com/anudeepND/whitelist  
https://discourse.pi-hole.net/t/solved-dns-resolution-is-currently-unavailable/33725/3

### Option 1 - Start Pihole in container

> _Go to the next section if you want to start Pihole with Unbound. This is for starting Pihole only in a docker container._

Using the `docker-compose.yaml` file from https://hub.docker.com/r/pihole/pihole/ as reference, create the file by running `touch docker-compose.yml`, then `sudo nano docker-compose.yml` to open the file in an editor, and finally copy pasting the contents in.

Optionally uncomment the environment property `WEBPASSWORD` and point it to an environment variable - i.e. `WEBPASSWORD: $PIHOLE_PASSWORD`. (Note that if we don't give it a password, a random one will be generated which you can find in the pihole startup logs). Now Pihole's web UI password will be set to whatever the environment variable `$PIHOLE_PASSWORD` is. You can set this environment variable on the pi by running: `export PIHOLE_PASSWORD=’<password>’`. A more permanent alternative is to create a file in the same directory as the docker-compose.yml file called `.env` by running `touch .env`, then go into the file using an editor by running `sudo nano .env` and add your environment variables - e.g. `PIHOLE_PASSWORD=password`.

In the directory where the `docker-compose.yml` is in, run `docker-compose up -d`. You can find the newly created container's id by running `docker ps -a`. Using the container id, you can use it to tail the logs as it starts up using `docker logs -f <docker container id>`.

### Option 2 - Start Pihole and Unbound in a single container

Clone this git repository by running `git clone https://github.com/willypapa/raspberrypi.git`. It will clone the files into `/raspberrypi` directory. Change directory to `cd docker-pihole-unbound`. Follow the `README.md` in this directory.

### Whitelist common false-positives

This is optional. Ths [github repo](https://github.com/anudeepND/whitelist.git) keeps a list of common false-positive domains for us to whitelist.

The whitelist is installed using a python script. However, the pihole/pihole docker image does not include a python installation. So we have to run the following on the host raspberry pi itself which should have python3 installed.

Run `git clone https://github.com/anudeepND/whitelist.git`, then `sudo python3 whitelist/scripts/whitelist.py --dir <path to /etc-pihole/ volume> --docker`

### Change Your Router's DNS Server

Log onto your home network's router (usually device 1 on your subnet - e.g. 192.168.0.1), and change the default DNS server to be the IP address of the raspberry pi running pihole. Instructions on how to do this will vary depending on your router.

### Verify that it works

Use this [ad blocker test](https://ads-blocker.com/testing/).

### Stopping the container

To stop the pihole-unbound container, go to the directory with the docker-compose.yml file and run `docker-compose down`.

### Updating Pihole-Unbound

When there is a new version available for pihole, you can update by first stopping the container by running `docker-compose down`, then rebuild and restart the docker container with `docker-compose up --build -d`. The build flag forces it to rebuild the image first which will pull the latest official pihole docker image.

### Set up VPN Server

Set up a OpenVPN or WireGuard VPN server so that your devices away from home can browse the internet through your locally hosted Pihole. See https://docs.pi-hole.net/guides/vpn/openvpn/overview/.

My router supports OpenVPN so I managed to set one up using that. Otherwise you could consider using PiVPN to setup a Wireguard or OpenVPN server on your raspberry pi. See [PiVPN](https://www.pivpn.io/) and [their docs](https://docs.pivpn.io/).

### Troubleshooting: DNS resolution is currently unavailable

I encountered [this issue](https://discourse.pi-hole.net/t/solved-dns-resolution-is-currently-unavailable/33725) a couple of times. When starting the pihole-unbound docker container, I see in the logs that the "DNS resolution is currently unavailable".

I followed the instructions in (this comment)[https://discourse.pi-hole.net/t/solved-dns-resolution-is-currently-unavailable/33725/7] which appeared to solve it. This changes the nameserver in the `resolv.conf` within the container from `127.0.0.11` to `127.0.0.1`.

<a name="log2ram"></a>

# Log2Ram

### Sources

https://github.com/azlux/log2ram  
https://levelup.gitconnected.com/extend-the-lifespan-of-your-raspberry-pis-sd-card-with-log2ram-5929bf3316f2  
https://www.geekbitzone.com/posts/log2ram/log2ram-raspberry-pi/

### Purpose

Note: (This step isn't really needed for a Raspberry Pi running Pihole)[https://www.reddit.com/r/pihole/comments/ltiet8/comment/goygick/?utm_source=share&utm_medium=web2x&context=3]. I'm just keeping notes here for the record.

Log2ram is software that redirects logs to memory instead of the micro-SD card, only writing to the micro-SD card at set intervals or during system shutdown. By default, the interval is once a day. This supposedly extends the lifespan of micro-SD cards by reducing the number of writes to disk.

> If you use Docker on your Raspberry Pi, note that each container has its logs written inside their respective containers rather than `/var/log`, so won't benefit from log2ram.
>
> I've yet to explore a way to map them to /var/log to get benefit from log2ram (e.g. as suggested in [this github issue](https://github.com/gcgarner/IOTstack/issues/8)).

### Installing Log2Ram

Add Log2Ram repository to our apt sources list, `echo "deb http://packages.azlux.fr/debian/ buster main" | sudo tee /etc/apt/sources.list.d/azlux.list`.

Download the public key to allow us to install Log2Ram, `wget -qO - https://azlux.fr/repo.gpg.key | sudo apt-key add -`.

Update your apt packages, `sudo apt update`, then install log2ram, `sudo apt install log2ram`.

Reboot once installed, `sudo reboot`.

### Verify that it works

After reboot, check that log2ram is mounted on `/var/log` by runninng `df -h`.

Also verify that log2ram is mounted to `/var/log` by running `mount`.

### Uninstall

To uninstall, run `sudo apt remove log2ram --purge`. The purge option removes the config files as well. Using the verify steps above, check that log2ram has unmounted.

