# Instalación de Ardupilot en Beaglebone Blue con Debian 10.X (Buster)

> [!WARNING]
> Funcionó una sola vez

## Configuración del Sistema base

### Descarga

[Imagénes de disco usadas aquí](https://rcn-ee.net/rootfs/bb.org/testing/)

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
printf "[service_%s]\n" $(sudo connmanctl services | grep '<your SSID here>' | grep -Po 'wifi_[^ ]+') |& sudo tee /var/lib/connman/wifi.config
```

```sh
sudo nano /var/lib/connman/wifi.config

```

```sh
Type = wifi
Security = wpa2
Name = <your SSID here>
Passphrase = <your WiFi password here>
```

The next configuration is made for an WPA2-PSK secured network, to refer other cases use the next [link](https://wiki.archlinux.org/title/ConnMan).

You can verify you have internet by typing: `ping -c 3 google.com`.  
To speed up the connection, reload the iwd service by command: `sudo systemctl daemon-reload`.  
When the Wifi LED is green, Check the IP by using: `ip addr show wlan0`.  
Then reboot the BBBlue with: `sudo reboot` and reconnect via SSH with the new IP.

#### Update System and Kernel

Install locales (a lot of programs complain otherwise) and set them:

```sh
sudo apt update -y
sudo apt install -y locales

```

```sh
sudo dpkg-reconfigure locales
```

Choose a locale (e.g. en_US.UTF-8 = English, United States, UTF8). This may take a while.

```sh
sudo apt install -y librobotcontrol
sudo apt dist-upgrade -y

```

```sh
sudo dpkg-reconfigure tzdata
```

Choose the closest to your location.

[Choose Kernel](https://forum.beagleboard.org/t/armhf-debian-10-x-11-x-12-x-kernel-updates/30928)

```sh
sudo apt-mark manual cryptsetup
sudo apt purge -y cryptsetup-initramfs
sudo apt autoremove -y
sudo apt install -y bbb.io-kernel-4.19-ti-rt-am335x 
#sudo apt install -y bbb.io-kernel-5.10-bone-rt 
sudo apt purge -y bbb.io-kernel-4.19-ti
sudo reboot
```

```sh
sudo sed -i 's/GOVERNOR="ondemand"/GOVERNOR="performance"/g' /etc/init.d/cpufrequtils
sudo sed -i 's|#dtb=|dtb=am335x-boneblue.dtb|g' /boot/uEnv.txt
sudo sed -i 's|#dtb_overlay=<file8>.dtbo|dtb_overlay=/lib/firmware/BB-I2C1-00A0.dtbo\ndtb_overlay=/lib/firmware/BB-UART4-00A0.dtbo\ndtb_overlay=/lib/firmware/BB-ADC-00A0.dtbo|g' /boot/uEnv.txt
sudo sed -i 's|uboot_overlay_pru=AM335X-PRU-RPROC-4-19-TI-00A0.dtbo|#uboot_overlay_pru=AM335X-PRU-RPROC-4-19-TI-00A0.dtbo|g' /boot/uEnv.txt
sudo sed -i 's|#uboot_overlay_pru=AM335X-PRU-UIO-00A0.dtbo|uboot_overlay_pru=AM335X-PRU-UIO-00A0.dtbo|g' /boot/uEnv.txt

```

`cpufreq-info` and `sudo reboot`

### Devops

```sh
cd /usr/local/sbin/
sudo wget -N https://raw.githubusercontent.com/mvduin/bbb-pin-utils/master/show-pins
sudo chmod 0755 show-pins 
cd $HOME
sudo show-pins | sort
```

Please note that if you experience an RCOutputAioPRU.cpp:SIGBUS error, there's a couple of things to try. Wiping the eMMC boot sector with `sudo dd if=/dev/zero of=/dev/mmcblk1 bs=1M count=10`.

`sudo systemctl list-units --type=service --state=active`

`systemctl list-unit-files --type=service --state=enabled`

Need a 2S LiPo Battery to use the command `rc_test_servos -h` and then `sudo rc_test_servos -c 1 -w 1500`.

`lsmod | grep uio`

`ls /sys/class/pwm`

`find /usr /boot -type f -name "*.dtbo"`

Chequear con:

```sh
sudo /opt/scripts/tools/version.sh
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
sudo wget -O /usr/bin/ardupilot/arduplane https://firmware.ardupilot.org/Plane/stable-4.1.6/blue/arduplane
sudo chmod 0755 /usr/bin/ardupilot/arduplane

```

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
sudo chmod 0755 /usr/bin/ardupilot/aphw
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
ExecStartPre=/usr/bin/ardupilot/aphw
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
sudo pip3 install pyserial pybluez2==0.42
```
