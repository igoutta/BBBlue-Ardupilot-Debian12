# Instalación de Ardupilot en Beaglebone Blue con Debian 12.X (Bookworm)

## Configuración del Sistema base

### Descarga

[Enlace al sitio](https://forum.beagleboard.org/t/debian-12-x-bookworm-monthly-snapshot-2023-10-07/36175)

### Primer inicio y conexión SSH al sistema base

En caso de cambio de kernel o reset/flash completo, borre el ssh anterior con el siguiente comando:

```bash
ssh-keygen -R "192.168.7.2"
```

`ssh debian@192.168.7.2` and enter the password 'temppwd'.

```bash
echo "debian ALL=(ALL) NOPASSWD: ALL" | sudo tee -a /etc/sudoers.d/debian >/dev/null
```

```bash
sudo sed -i 's/#EnableNetworkConfiguration=true/EnableNetworkConfiguration=true/g' /etc/iwd/main.conf
```

The next configuration is made for an WPA2-PSK secured network, to refer other cases use the next [link](https://wiki.archlinux.org/title/Iwd).

`sudo nano /var/lib/iwd/<your SSID>.psk` If the SSID contains weird symbology, read this [guide](https://www.reddit.com/r/archlinux/comments/v7k25o/comment/ice0wa4/?utm_source=share&utm_medium=web3x&utm_name=web3xcss&utm_term=1&utm_content=share_button).

```properties
[Security]
Passphrase=<your password>
```

You can verify you have internet by typing: `ping -c 3 google.com`.  
To speed up the connection, reload the iwd service by command: `sudo systemctl daemon-reload`.  
When the Wifi LED is green, Check the IP by using: `ip addr show wlan0`.  
Then reboot the BBBlue with: `sudo reboot` and reconnect via SSH with the new IP.

Install locales (a lot of programs complain otherwise) and set them:

```bash
sudo apt update -y
sudo apt install -y locales
sudo dpkg-reconfigure locales
```

Choose a locale (e.g. en_US.UTF-8 = English, United States, UTF8). This may take a while.

```shell
sudo apt dist-upgrade -y
#sudo apt install -y git cpufrequtils 
sudo apt install -y librobotcontrol #broken rightnow
```

```shell
sudo apt install -y bbb.io-kernel-4.19-bone-rt
```

### Devops

```shell
cd /usr/local/sbin/
sudo wget -N https://raw.githubusercontent.com/mvduin/bbb-pin-utils/master/show-pins
sudo chmod 0755 show-pins 
cd $HOME
sudo show-pins | sort
```

Chequear que:

```shell
sudo /opt/scripts/tools/version.sh
```

## Configuracion del servicio de Ardupilot

```shell
sudo nano /etc/default/ardupilot 
```

```shell
# WiFi Telemetry
SERIAL0="-A udp:192.168.1.75:14550"

# Radio Telemetry (UART1 - "UT1")
SERIAL1="-C /dev/ttyS1"

# GPS1 (UART2 - "GPS")
SERIAL3="-B /dev/ttyS2"

```

```shell
sudo mkdir -p /usr/bin/ardupilot/
sudo wget -O /usr/bin/ardupilot/arduplane https://firmware.ardupilot.org/Plane/stable-4.4.1/blue/arduplane


sudo nano /usr/bin/ardupilot/aphw
```

```shell
#!/bin/bash
# aphw
# ArduPilot hardware configuration.

echo 80 >| /sys/class/gpio/export
echo out >| /sys/class/gpio/gpio80/direction
echo 1 >| /sys/class/gpio/gpio80/value
#echo pruecapin_pu >| /sys/devices/platform/ocp/ocp:P8_15_pinmux/state

```

```shell
mkdir $HOME/scripts
sudo ln -s $HOME/scripts /scripts
```

```shell
sudo nano /lib/systemd/system/arduplane.service 
```

```properties
[Unit]
Description=ArduPlane Service
BindsTo=sys-subsystem-net-devices-wlan0.device
After=sys-subsystem-net-devices-wlan0.device
StartLimitIntervalSec=0
Conflicts=arducopter.service arduplane.service antennatracker.service

[Service]
EnvironmentFile=/etc/default/ardupilot
ExecStartPre=/usr/bin/ardupilot/aphw
ExecStart=/usr/bin/ardupilot/arduplane $SERIAL0 $SERIAL1 $SERIAL3
Restart=on-failure
RestartSec=1

[Install]
WantedBy=multi-user.target

```

### Bluetooth

```shell
sudo apt install -y python3-pip libbluetooth3
sudo pip3 install pyserial pybluez2==0.42
```
