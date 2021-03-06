# Set up helium DIY pi

## Image your Raspberry Pi microsd card
+ Download the Raspberry Pi OS 64 bit lite image
` http://downloads.raspberrypi.org/raspios_lite_arm64/images/`

+ Flash it to your micro sd card using using a tool such as Etcher or Raspberry Pi imager.

+ Create new (empty) file on root of SD card called `ssh`

+ Add wifi login info if desired, to SD card at `etc/wpa_supplicant/wpa_supplicant.conf`

```
country=countryCode
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1
 
network={
    ssid="YOURSSID"
    psk="YOURPASSWORD"
    scan_ssid=1
}
```

## Pi basic Config

+ SSH into raspberry pi
```console
ssh pi@raspberrypi.local
```
or
```console
ssh pi@IP-ADDRESS' where IP-ADDRESS is the local IP of the raspberry pi
```
The default password is `raspberry`

+ Change your password 
```console
passwd
```

+ Launch Raspbery Pi configurator
```console
sudo raspi-config
```

    - Select `Interfacing Options`
        - Select `SPI`
        - Select `Yes`
    - Select `Interfacing Options`
        - Select `IC2`
        - Select `Yes`
    - Select `Interfacing Options`
        - Select `Serial Communictioans`
        - Select `No` for shell access
        - Select `Yes` for serial port hardware
        - Select `Yes` to confirm
    - Set hostname if desired
    - Save changes and reboot by selecting `Finish`

+ SSH back in

+ Increase the swap file size
    - Turn off swap

    ```console
    sudo dphys-swapfile swapoff
    ```

    - Edit config file
    ```console
    sudo nano /etc/dphys-swapfile
    ```

    - Edit the line 'CONF_SWAPSIZE=100' to 'CONF_SWAPSIZE=1024"
    - Press CTRL-X, and then Y, and then Enter to save changes
    - Reboot with `sudo reboot`

+ Update software on the pi
    - Run `sudo apt update` to update the software repos
    - Run `sudo apt upgrade` to perform the updates
    - Install git and jq - `sudo apt install git jq` 

+ Install Docker using convience script from https://docs.docker.com/engine/install/debian/

    - Download the script - `curl -fsSL https://get.docker.com -o get-docker.sh`
    - Run the install script - `sudo sh get-docker.sh`
    - Remove the install script - 'rm get-docker.sh'
    - Allow pi user to run docker by adding pi to the docker group - `sudo usermod -aG docker pi`
    - Reboot for changes to take affect `sudo reboot`

## Set up Miner on Pi
Source: https://developer.helium.com/blockchain/run-your-own-miner

+ SSH into raspberry pi - `ssh pi@raspberrypi.local`

+ Get the filename for the most recent version of the miner docker image from quay.io/team-helium/miner. Make sure to get the arm64 version for the pi. They are in the format `miner-xxxNN_YYYY.MM.DD` to the current one at the time of this document is `miner-arm64_2020.09.08.0_GA`.

+ Create directory for miner data - `mkdir ~/miner_data`

+ This docker command will download docker image and set it to always start up. Be sure to swap out `miner-xxxNN_YYYY.MM.DD.0_GA` for the current image name. Make sure that REGION_OVERRIDE variable matches your region. 

```
sudo docker run -d \
  --env REGION_OVERRIDE=US915 \
  --restart always \
  --publish 1680:1680/udp \
  --publish 44158:44158/tcp \
  --name miner \
  --mount type=bind,source=/home/pi/miner_data,target=/var/data \
  quay.io/team-helium/miner:miner-xxxNN_YYYY.MM.DD.0_GA
```

+ Verify that your container has started - `docker ps`

For maxium effort, set up port forwards on your internet router to the pi. Outside port '44158/TCP' should forward to the internal IP of the pi.

## Set up Packet Forwarder

+ Make sure your are in your home directory - `cd ~`

+ Clone the packet forwarer for sx1302 based gateways like the RAK2287 - `git clone https://github.com/helium/sx1302_hal`

+ Move to the project folder - `cd sx1302_hal`

+ Compile the project - `make clean all`

+ Make the executables - `make install`
Note you can change parameters for this step in `target.cfg`

+ Install the config files - `make install_conf`

+ Move into bin directory - `cd bin`

+ Make a copy of the conf file for your reggion and name it as the default- `cp global_conf.json.sx1250.US915 global_conf.json`

+ Change the port in the conf file via - `nano global_conf.json`
Change this text that is close to the end of the file:
```
        "serv_port_up": 1730,
        "serv_port_down": 1730,
```
to 

```
        "serv_port_up": 1680,
        "serv_port_down": 1680,
```

+ Change the pins that reset the concentrator - `nano reset_lgw.sh`
edit the first 2 lines to this

```
SX1302_RESET_PIN=17	
SX1302_POWER_EN_PIN=2
```

+ Add pi user to gpio group - `sudo usermod -aG gpio pi`

+ Reboot for changes to take effect - `sudo reboot`

+ SSH back in to pi - `ssh pi@raspberrypi.local`

+ Move to lora packet folder bin directory - `cd sx1302_hal/bin`

+ Test out the packet forwarder - `./lora_pkt_fwd`
It's okay if you see an error for GPS. GPS is not used.

+ Stop the packet forwarder by pressing CTRL-c


## Set up auto start scripts
Source: https://github.com/helium/sx1302_hal/tree/helium/hotspot/tools/systemd

+ Create a systemd service script - `sudo nano /etc/systemd/system/lora_pkt_fwd.service`

+ Paste the following into new text file:

```
[Unit]
Description=LoRa Packet Forwarder
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
WorkingDirectory=/home/pi/sx1302_hal/bin
ExecStart=/home/pi/sx1302_hal/bin/lora_pkt_fwd -c /home/pi/sx1302_hal/bin/global_conf.json
Restart=always
RestartSec=30
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=lora_pkt_fwd
User=pi

[Install]
WantedBy=multi-user.target
```

+ Save the file by pressing CTRL-X, and then Y, and then Enter to save changes

+ Reload the daemon 

```console
sudo systemctl daemon-reload
```

+ Enable the service 

```console
sudo systemctl enable lora_pkt_fwd.service
```


### The following commands to disable the service, manually start/stop it and check status:

```console
sudo systemctl disable lora_pkt_fwd.service
```

```console
sudo systemctl start lora_pkt_fwd.service
```

```console
sudo systemctl stop lora_pkt_fwd.service
```

```console
sudo systemctl status lora_pkt_fwd.service
```

+ Configure rsyslog to redirect the packet forwarder logs into a dedicated file

```console
sudo nano /etc/rsyslog.d/lora_pkt_fwd.conf
```

+ Paste the following into the text editor:
```
if $programname == 'lora_pkt_fwd' then /var/log/lora_pkt_fwd.log
if $programname == 'lora_pkt_fwd' then ~
```

+ Save the file by pressing CTRL-X, and then Y, and then Enter to save changes

+ Restart the rsyslog service - `sudo systemctl restart rsyslog`

### See the Lora Packet Forwarder logs

```console
tail /var/log/lora_pkt_fwd.log
```
