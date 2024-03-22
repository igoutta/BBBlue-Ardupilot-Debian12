# Instalación de Ardupilot en Beaglebone Blue con Debian 12.X (Bookworm)

> [!CAUTION]
> Actualmente no ha funcionado

## Configuración del Sistema base

### Descarga

[Snapshots usadas aquí](https://forum.beagleboard.org/t/debian-12-x-bookworm-monthly-snapshot-2023-10-07/36175)

### Primer inicio y conexión SSH al sistema base

#### Host

En caso de cambio de kernel o reset/flash completo, borre el ssh anterior con el siguiente comando:

```sh
ssh-keygen -R "192.168.7.2"
```

`ssh debian@192.168.7.2` and enter the password 'temppwd'.

#### [BBBlue Security Overwrite](https://elinux.org/Beagleboard:BeagleBoneBlack_Debian#i_take_full_responsibility_for_knowing_my_beagle_is_now_insecure)

```sh
echo "debian ALL=(ALL) NOPASSWD: ALL" | sudo tee -a /etc/sudoers.d/debian >/dev/null
```

```sh
sudo -s
passwd debian
123
123
exit
```

#### [Wifi](https://wiki.debian.org/WiFi/HowToUse)

```sh
sudo sed -i 's/#EnableNetworkConfiguration=true/EnableNetworkConfiguration=true/g' /etc/iwd/main.conf
```

The next configuration is made for an WPA2-PSK secured network, to refer other cases use the next [link](https://wiki.archlinux.org/title/Iwd).

```sh
sudo nano /var/lib/iwd/<Put your SSID here>.psk
```

If the SSID contains weird symbology, read this [guide](https://www.reddit.com/r/archlinux/comments/v7k25o/comment/ice0wa4/?utm_source=share&utm_medium=web3x&utm_name=web3xcss&utm_term=1&utm_content=share_button).

```properties
[Security]
Passphrase=<Put your network passwd here>
```

You can verify you have internet by typing: `ping -c 3 google.com`.  
To speed up the connection, reload the iwd service by command: `sudo systemctl daemon-reload`.  
When the Wifi LED is green, Check the IP by using: `ip addr show wlan0`.  
Then reboot the BBBlue with: `sudo reboot` and reconnect via SSH with the new IP.

#### Update System and Kernel

Install locales (a lot of programs complain otherwise) and set them:

```sh
sudo apt update -y
sudo apt install -y locales
sudo dpkg-reconfigure locales
```

Choose a locale (e.g. en_US.UTF-8 = English, United States, UTF8). This may take a while.

```sh
sudo apt dist-upgrade -y
sudo dpkg-reconfigure tzdata
```

[Choose Kernel](https://forum.beagleboard.org/t/armhf-debian-10-x-11-x-12-x-kernel-updates/30928)

```sh
sudo apt-mark manual cryptsetup
sudo apt purge -y cryptsetup-initramfs
sudo apt autoremove -y
sudo apt install -y bbb.io-kernel-5.10-ti-rt-am335x
sudo apt remove -y bbb.io-kernel-5.10-ti-am335x --purge
sudo sed -i 's/GOVERNOR="ondemand"/GOVERNOR="performance"/g' /etc/init.d/cpufrequtils
sudo reboot
```

`cpufreq-info`

#### Try 1

[Built-in Overlays](https://forum.beagleboard.org/t/arm64-debian-12-x-bookworm-monthly-snapshots-2023-10-07/35565)

```sh
sudo nano /boot/uEnv.txt 
```

```sh
sudo sed -i 's/#uboot_overlay_addr0=<file0>.dtbo/uboot_overlay_addr0=BONE-PWM0.dtbo/g' /boot/uEnv.txt
sudo sed -i 's/#uboot_overlay_addr1=<file1>.dtbo/uboot_overlay_addr1=BONE-PWM1.dtbo/g' /boot/uEnv.txt
```

Fail: Only in BBIA - ARM64

#### Try 2 & 3

```sh
git clone https://openbeagle.org/beagleboard/librobotcontrol.git
cd librobotcontrol/
make |& tee build.log
sudo make install |& tee install.log
```

```sh
git clone --depth=1 -b "v5.10.x-ti-unified" https://openbeagle.org/beagleboard/BeagleBoard-DeviceTrees.git
cd BeagleBoard-DeviceTrees/
make |& tee build.log
sudo make install_arm |& tee install.log
```

FAIL: Doesn't reboot after installed the Overlays. librobotcontrol restarted correctly.

### Devops

`sudo systemctl list-units --type=service --state=active`

`systemctl list-unit-files --type=service --state=enabled`

Buitl-in Debian 12: `sudo show-pins | sort`

`lsmod | grep uio`

`ls /sys/class/pwm`

`find /usr /boot -type f -name "*.dtbo"`

Chequear con:

```sh
sudo beagle-version
```

## Configuracion del servicio de Ardupilot

```sh
sudo nano /etc/default/ardupilot 
```

```shell
# WiFi Telemetry
# Used my Workbook IP  192.168.1.75 here
SERIAL0="-A udp:<target IP address here>:14550"

# Radio Telemetry (UART1 - "UT1")
SERIAL1="-C /dev/ttyS1"

# GPS1 (UART2 - "GPS")
SERIAL3="-B /dev/ttyS2"

```

```sh
sudo mkdir -p /usr/bin/ardupilot/
sudo wget -O /usr/bin/ardupilot/arduplane https://firmware.ardupilot.org/Plane/stable-4.4.4/blue/arduplane
sudo chmod 0755 /usr/bin/ardupilot/arduplane
```

### Outdated (Doesn´t work right now)

```sh
sudo nano /usr/bin/ardupilot/aphw
```

```shell
#!/bin/bash
# aphw
# ArduPilot hardware configuration.

echo 80 >| /sys/class/gpio/export
echo out >| /sys/class/gpio/gpio80/direction
echo 1 >| /sys/class/gpio/gpio80/value
echo pruecapin_pu >| /sys/devices/platform/ocp/ocp:P8_15_pinmux/state
```

```sh
sudo chmod 0755 /usr/bin/ardupilot/a*
```

### Keep Going

```sh
mkdir $HOME/scripts
sudo ln -s $HOME/scripts /scripts
```

```sh
sudo nano /lib/systemd/system/arduplane.service 
```

```properties
[Unit]
Description=ArduPlane Service
BindsTo=sys-subsystem-net-devices-wlan0.device
After=sys-subsystem-net-devices-wlan0.device
StartLimitIntervalSec=0
Conflicts=arducopter.service ardurover.service antennatracker.service

[Service]
EnvironmentFile=/etc/default/ardupilot
ExecStart=/usr/bin/ardupilot/arduplane $SERIAL0 $SERIAL1 $SERIAL3
Restart=on-failure
RestartSec=1

[Install]
WantedBy=multi-user.target

```

```sh
sudo systemctl enable arduplane.service
sudo reboot
```

`sudo systemctl status arduplane.service`  
`sudo systemctl stop arduplane.service`  
`sudo systemctl start arduplane.service`  
`sudo systemctl restart arduplane.service`

### Bluetooth

```sh
sudo apt install -y python3-pip libbluetooth3
sudo pip3 install pyserial bless
```
