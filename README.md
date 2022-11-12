# Contents

1. [ Setting Up Raspberry Pi ](#setuppi)
2. [ Securing Raspberry Pi ](#securingpi)
3. [ Installing Git ](#installinggit)
4. [ Installing Docker and Docker Compose ](#installingdocker)
5. [ Pihole and Unbound with Docker Compose ](#piholeandunboundwithdockercompose)

<a name="setuppi"></a>

# Setting Up Raspberry Pi

### Preparing SD Card

On another device, download the [Raspberry Pi Imager](https://www.raspberrypi.com/documentation/computers/getting-started.html#using-raspberry-pi-imager). For a headless Raspberry Pi setup, choose the latest RaspberryPi Lite OS image. The Lite version does not include the desktop GUI and many other apps in the full version. Select your SD card. 

Before we write the OS image onto the SD card, go into the advanced menu cogwheel to create a new user and password. In the past, there was a default user `pi` with password `raspberry`, but this is no longer the case for security reasons. Still in the advanced menus, enable SSH either using password authentication or by key-based authentication only by supplying a public key. I personally use an ethernet cable to connect it to the network, but you can also configure Wi-Fi here so that the Raspberry Pi will automatically connect to it on the first boot.

With the advanced options set, we can now write the Raspberry Pi OS image onto the SD card.

### First boot

Put the micro-SD card into the Raspberry Pi and then power it on.

Wait a little for it to start up, then either [use your router](https://www.raspberrypi.com/documentation/computers/remote-access.html#ip-address) or use any other means to find the IP address of the Raspberry Pi on your local network. Then ssh into it by running this command `ssh <username>@<IP address>`.

### Correctly shutting down the Raspberry Pi

To restart the pi, run `sudo reboot`.

When you want to shut down the pi, pulling power cord without properly shutting down the system increases the risk of corrupting the micro SD card. Anything running will also not save and exit gracefully. To properly shutdown, run `sudo shutdown -h now`. Give it a second to send SIGTERM, SIGKILL signals to all running processes, and unmount all file systems. Only when the system is halted can you pull the power to the device with minimised risk.

To start the Raspberry pi back up, simply turn on the power.

### Optional minor power consumption tweaks

If you are running a headless Raspberry Pi, then according to [this blog by Jeff Geering](https://www.jeffgeerling.com/blogs/jeff-geerling/raspberry-pi-zero-conserve-energy), you can save a little bit of power by disabling the HDMI display circuitry.

Run the command `/usr/bin/tvservice -o` to disable HDMI. Also run `sudo nano /etc/rc.local` and add the command there too in order to disable HDMI on boot. (To enable again, run `/usr/bin/tvservice -p`, and remove from `/etc/rc.local`).

<a name="securingpi"></a>

# Securing Raspberry Pi

## Change away from pi user

If you are still using the default username `pi` with default password `raspberry`, make sure to [change your username and password](https://www.raspberrypi.com/documentation/computers/configuration.html#changing-your-username).

### Change password for pi user

- Run `sudo raspi-config`, go to “System Options”, and then “Password”.

### Create a new administrative superuser account other than pi

- Create new user by running `sudo adduser <account_name>`
- Give sudo by running `sudo usermod -a -G adm,dialout,cdrom,sudo,audio,video,plugdev,games,users,input,netdev,gpio,i2c,spi <account_name>`
- Check sudo by running `sudo su - alice`

Finally, check the new user is capable of logging in to the raspberrypi using a new terminal and is able to use sudo by running `sudo whoami`.

### Lock the pi account

We could delete the pi accound instead of locking it, but some software may still rely on the pi account to work. Log onto the administrative superuser account set up in the previous step, then run `sudo passwd --lock pi`.

## [Make sudo require a password](https://www.raspberrypi.com/documentation/computers/configuration.html#make-sudo-require-a-password)

- Run `sudo visudo /etc/sudoers.d/010_pi-nopasswd`.
- Make sure usernames with superuser rights have this entry: `<username> ALL=(ALL) PASSWD: ALL`

## Updating OS

- Run `sudo apt update`
- Run `sudo apt full-upgrade -y`
- Run `sudo apt clean` to clean up the downloaded package files.

## [Improving SSH](https://www.raspberrypi.com/documentation/computers/configuration.html#improving-ssh-security)

### Update SSH

Automatically update ssh everyday at midnight:

- Run `sudo crontab -e`, select 1
- Add this to bottom of file: `0 0 * * * apt install openssh-server`

### Restrict ssh users 

Allow or deny specific users by altering the sshd configuration:

- Run `sudo nano /etc/ssh/sshd_config` to open config
- At the bottom, allow users to ssh like this, `AllowUsers alice bob`, or deny users to ssh like this, `DenyUsers jane john`
- Restart SSH by running`sudo systemctl restart ssh`

### Use key-based authentication

#### Set up public-private key

See [here](https://www.raspberrypi.com/documentation/computers/configuration.html#using-key-based-authentication).

#### Disable password authentication

- Run `sudo nano /etc/ssh/sshd_config`
- Change the following properties to `no`:
  ```
  ChallengeResponseAuthentication no
  PasswordAuthentication no
  UsePAM no
  ```
- Reload ssh `sudo service ssh reload`

## Kill unnecessary system services

To reduce idling power usage, and also reduce the areas for compromise, we can list all running services and disable services you don't need - e.g. wifi, bluetooth or sound card drivers.

To see all active services, run `sudo systemctl --type=service --state=active`.

To disable the wifi service or the bluetooth service now, run `sudo systemctl disable --now wpa_supplicant.service` or `sudo systemctl disable --now bluetooth.service` respectively.

If you wanted to enable a service again, run `sudo systemctl enable --now bluetooth.service`.

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

### Start Pihole and Unbound in a single container

Clone this git repository by running `git clone https://github.com/willypapa/raspberrypi-pihole-unbound-docker.git`. It will clone the files into `/raspberrypi-pihole-unbound-docker` directory. Change directory to `cd docker-pihole-unbound`. Follow the `README.md` in that directory.
