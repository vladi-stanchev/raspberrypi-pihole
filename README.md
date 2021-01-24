# Contents

1. [ Setting Up Raspberry Pi ](#setuppi)
2. [ Securing Raspberry Pi ](#securingpi)
3. [ Installing Git ](#installinggit)
4. [ Installing Docker and Docker Compose ](#installingdocker)
5. [ Pihole and Unbound with Docker Compose ](#piholeandunboundwithdockercompose)
6. [ Flashing Smart Devices using Tuya-Convert](#tuyaconvert)

<a name="setuppi"></a>
# Setting Up Raspberry Pi

### Sources

https://www.raspberrypi.org/documentation/installation/installing-images/README.md  
https://www.raspberrypi.org/documentation/remote-access/ssh/README.md  
https://www.raspberrypi.org/documentation/configuration/wireless/headless.md  

### Preparing SD Card

On another device, download and use the "Raspberry Pi Imager" to flash RaspberryPi OS onto a micro-SD card. The Lite version does not include the desktop GUI, so use that if you are setting the raspberry pi up headless.

To enable ssh, create empty file called `.ssh` at root level on your SD card before the first boot.

I personally use ethernet, but if you want to use wifi, you can automatically connect to a wifi network on the first boot by adding a ``wpa_supplicant.conf`` file at root level before first boot with the following:

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

You can optionally change the hostname of your device as follows. Run ``sudo raspi-config`` then go to "System Options", then "Hostname". You can then type in your new hostname and reboot.

Alternatively, run ``sudo nano /etc/hosts``, change the old raspberry pi hostname to your new one, save and exit by hitting Ctrl-X and the "Y" for yes. Then run ``sudo nano /etc/hostname``, change the hostname there to the new one, save and exit. Finally reboot by running ``sudo reboot``.

### Correctly shutting down the Raspberry Pi

To restart the pi, run ``sudo reboot``.

When you want to shut down the pi, pulling power cord without properly shutting down the system increases the risk of corrupting the micro SD card. Anything running will also not save and exit gracefully. To properly shutdown, run ``sudo shutdown -h now``. Give it a second to send SIGTERM, SIGKILL signals to all running processes, and unmount all file systems. Only when the system is halted can you pull the power to the device with minimised risk. 

To start the Raspberry pi back up, simply turn on the power.

There are ways to create a power button using the GPIO pins on the board. // TODO: explore this

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

``docker ps -a``  
``docker stop <container-id>``  
``docker image ls``  
``docker image rm <image-id>``  
``docker rm -f <container-id>``  
``docker exec -it <container-id> bash``  
``docker build -t <docker-username>/<image-name>``  
``docker push <docker-username>/<image-name>``

``docker-compose pull``  
``docker-compose down``  
``docker-compose up -d``

<a name="piholeandunboundwithdockercompose"></a>
# Pihole and Unbound with Docker Compose

### Sources

https://docs.pi-hole.net/   
https://hub.docker.com/r/pihole/pihole/  
https://github.com/pi-hole/docker-pi-hole  
https://docs.pi-hole.net/guides/unbound/  
https://github.com/chriscrowe/docker-pihole-unbound  
https://github.com/anudeepND/whitelist  

### Start Pihole in container

> *Go to the next section if you want to start Pihole with Unbound. This is for starting Pihole only in a docker container.*

Using the `docker-compose.yaml` file from https://hub.docker.com/r/pihole/pihole/ as reference, create the file by running `touch docker-compose.yml`, then `sudo nano docker-compose.yml` to open the file in an editor, and finally copy pasting the contents in. 

Edit the file to uncomment the environment property `WEBPASSWORD` and point it to an environment variable - i.e. `WEBPASSWORD: $PIHOLE_PASSWORD`. Note that if we don't give it a password, a random one will be generated which you can find in the pihole startup logs.

Now pihole's web UI password will be set to whatever the environment variable `$PIHOLE_PASSWORD` is. You can set it on the pi by running: `export PIHOLE_PASSWORD=’<password>’`. A more permanent alternative is to create a file in the same directory as the docker-compose.yml file called `.env` by running `touch .env`, then go into the file using an editor by running `sudo nano .etc` and add your environment variables - e.g. `PIHOLE_PASSWORD=password`.

In the directory where the `docker-compose.yml` is in, run `docker-compose up -d`. You can find new container's id by running `docker ps -a`. Using the container id, you can use it to tail the logs as it starts up using `docker logs -f <docker container id>`.

### Start Pihole and Unbound in a single container

Clone this git repository by running `git clone https://github.com/willypapa/raspberrypi.git`. It will clone the files into `/raspberrypi` directory, where you will find the docker-compose.yml file. We will now refer to the `pihole-unbound` service in the docker-compose.yml. 

> Note that this uses my own docker image on docker hub. You should create your own and not rely on me.
>
> Go to `/raspberrypi/docker-pihole-unbound` directory where the Dockerfile is found which builds the pihole and unbound image.
>
> Log into docker using command `docker login` and supply your dockerhub username and password. 
>
> Now build and push your own image to your own DockerHub using `docker build -t <your_docker_username>/pihole-unbound`, then `docker push <your_docker_username>/pihole-unbound`. 
>
> Change the `image` property in the `docker-compose.yml` file from to your own, `image: <your_docker_username>/pihole-unbound:latest`.

In the directory where the `docker-compose.yml` is (i.e. `/raspberrypi`), create a `.env` file by running `sudo touch .env`, then populate it with the following environment variables. These are referred to within the docker-componse.yml file.

```
PIHOLE_PASSWORD=password
PIHOLE_TIMEZONE=Europe/London
PIHOLE_ServerIP=<IP address of the host raspberry pi - e.g. 192.168.0.2>
```

Again, in the same directory as the docker-compose.yml, run `docker-compose up -d`. You can find new container's id by running `docker ps -a`. Using the container id, you can use it to tail the logs as it starts up using `docker logs -f <docker container id>`.

To test unbound is working, access the bash terminal in the docker container by running ``docker exec -it <container-id-of-pihole-unbound> bash``. Once in, run these two commands to test DNSSEC validation: ``dig sigfail.verteiltesysteme.net @127.0.0.1 -p 5335`` and ``dig sigok.verteiltesysteme.net @127.0.0.1 -p 5335``. The first should give a status report of ``SERVFAIL`` and no IP address. The second should give ``NOERROR`` plus an IP address. See pihole's unbound docs for more info.

To confirm Pihole is up and pointing to unbound dns server, go to the local IP address of the host raspberry pi, e.g 192.168.0.2. You should see the Pihole dashboard at /admin. Login using the password set in the environment variable ``$PIHOLE_PASSWORD``. Go to "Settings", and under the "DNS" tab, check that it is pointing to ``127.0.0.1#5335`` - which is the port we configured the unbound DNS server to listen to (see /docker-pihole-unbound/unbound-pihole.conf file).

### Whitelist common false-positives

This is optional. The github repo we refer to here keeps a list of common false-positive domains for us to whitelist.

The whitelist is installed using a python script. However, the pihole/pihole docker image does not include a python installation. So we have to run the following on the host raspberry pi itself which should have python3 installed.

Run `git clone https://github.com/anudeepND/whitelist.git`, then `sudo python3 whitelist/scripts/whitelist.py --dir <path to /etc-pihole/ volume> --docker`

### Change Your Router's DNS Server

Log onto your home network's router (usually device 1 on your subnet - e.g. 192.168.0.1), and change the default server to point to the IP address of the raspberry pi running pihole. Instructions on how to do this will vary depending on your router.

### Set up VPN Server

Set up a OpenVPN or WireGuard VPN server so that your devices away from home can browse the internet through your locally hosted Pihole. See https://docs.pi-hole.net/guides/vpn/openvpn/overview/.

My router supports OpenVPN so I managed to set one up using that. Otherwise you could consider using PiVPN to setup a Wireguard or OpenVPN server on your raspberry pi. See https://www.pivpn.io/.

<a name="tuyaconvert"></a>
# Flashing Smart Devices with Tuya-Convert

### Sources

https://github.com/ct-Open-Source/tuya-convert  
https://www.youtube.com/watch?v=dt5-iZc4_qU  
https://templates.blakadder.com/index.html  
https://tasmota.github.io/docs/About/

### Device Compatibility

This documents what I did to successfully flash the [Gosund (UP111) UK smart plugs](https://www.amazon.co.uk/gp/product/B0856T6TJC/) over-the-air (OTA) with Tasmota firmware.

These smart plugs were bought from Amazon on January 2021. Devices purchased later may have had their firmware updated which prevents this method from working, so do your own research online to check your device's compatibility.

### Installing Tuya-Convert

Carry out this process on a fresh Raspberry Pi OS install by swapping out your micro SD card, or re-flashing it for this purpose.

Perform a ``sudo apt update`` and install git, ``sudo apt install git -y``.

 Clone the Tuya-Convert git repo, ``git clone https://github.com/ct-Open-Source/tuya-convert``, then ``cd tuya-convert``, and run ``./install_prereq.sh``.

 ### Flashing the device

Start the flashing process by running ``./start_flash.sh``. Following the instructions, type "yes" and enter. It'll ask you if it can terminate dnsmasq process to free up port 53, type "y". It'll ask you to terminate mosquitto to free up port 1883, type "y". The script should have turned the raspberry pi into a WiFi access point with SSID "vtrust-flash".

Connect a smartphone (or any device) to "vtrust-flash" WiFi access point.

Plug your IoT device in and put it in autoconfig/smartconfig/pairing mode - you should know it's in the right mode when the LED starts blinking. This is usually done by pressing and holding the primary button of the device. In my case, I pressed the power button of the plug for 5 seconds.

Press enter to start flashing.

It'll then prompt you to ask which image you'd like to load onto the device. I flashed tasmota.bin using option "2", then confirm with "y". And thats it! 

>**Remember to keep your device plugged in for the next step.**

### Configure Tasmota

The device should now broadcast a Wifi access point with SSIP ``tasmota-xxxx``. Connect to it with your phone or another device to configure Tasmota.

One connected to the Tasmota Wifi, you can configure your home WiFi credentials. Make sure to double check the credentials before pressing "Save".

After pressing "Save", the device should restart and automatically connect to your home network.
