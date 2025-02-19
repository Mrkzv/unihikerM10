# unihikerM10
unihiker M10 tinkering

# Configure your unihiker to act like a basic linux OS with console (starting point for other project)

Board information: Manufacturer:  - Model: Radxa ROCK Pi S - Architecture: aarch64 - OS: Debian GNU/Linux 10 (buster)

## 1/ Update OS
First of all ensure that the firmware is up to date. 
### Download latest firmware and tools
https://www.unihiker.com/wiki/SystemAndConfiguration/UnihikerOS/unihiker_os_image/  
Ubuntu (AMD64): Tested Platform: i9-12900 + Ubuntu 20.04 LTS Download Link: https://dfimg.dfrobot.com/64228321aa9508d63a42c28b/wiki/1cb234325cca25ec876cd0ff70850217.7z  
MacOS (arm64): Tested Platform: Mac mini(M1) Download Link: https://dfimg.dfrobot.com/64228321aa9508d63a42c28b/wikien/e3053945dcf6a62e1c7c38555ce3ac1b.7z  
Windows: Tested Platform: i3-12100 + Win11 64bit Download Link: https://dfimg.dfrobot.com/64228321aa9508d63a42c28b/wiki/b4ab152645022f4cf66d4f392e1d5a92.7z  
Follow instructions on unihiker wiki on how to burn the OS to the board.

## 2/ First Unihiker boot
- Connect unihiker to a computer (one that is capable of ssh) ;
- Wait for startup ;
- Select your language using the unihiker UI (if you're not a chinese speaker select english, you'll be able to select another locale with the setup steps below) ;
<b> DO NOT GO TO http://10.1.2.3 & DO NOT SETUP WIFI USING THE WEB INTERFACE / UNIHIKER UI </b>

## 3/ SSH and new user setup
- Connect to unihiker board with ssh ;
  ```console
  ssh root@10.1.2.3 
  # use password : dfrobot
  ```
- passwd the root access ;
  ```console
  passwd
  ```
- There is a default account named rock, from root user also passwd rock ;
  ```console
  passwd rock
  ```
- You can setup a personnal account ;
  ```console
  adduser new_user
  # Where new_user is your user name
  ```
- You should also put the new user in the sudo group
  ```console
  su -
  usermod -aG sudo new_user
  exit
  exit
  ```
<b>Exit root ssh and login as the newly created user (ssh new_user@10.1.2.3)</b>
- You should modify the /home/new_user/.bashrc (with nano) to uncomment the alias ll ;
  ```console
  alias ll="ls -la"
  ```
- Also add the PATH to sbin in this file ;
  ```console
  export PATH=$PATH:/usr/local/sbin
  export PATH=$PATH:/usr/sbin
  ```
<b>Exit user ssh and log back in to update your user profile</b>

## 4/ Remove unihiker UI
@startup the unihiker start a lighdm desktop, with root autologin.  
When root log in it starts the PyboardUI ... see below  
In /root/.xsession you'll find this :
```console
echo 20 > /sys/class/gpio/export
echo out > /sys/class/gpio/gpio20/direction
echo 1 > /sys/class/gpio/gpio20/value
systemctl --user start PyboardUI
```
You can keep the lightdm desktop (not recommended) and launch an other python script by replacing the last line  
Or you can fully disable the lighdm desktop (which avoid using ressources)
```console
# [from a non-root user]
su -
systemctl disable lightdm.service
exit
```

## 5/ Setup wifi without web interface (recommended)
First you have to follow the step of making the ssh conf, the new user, and disabling lightDM  
```console
ssh new_user@10.1.2.3
wpa_passphrase SSID WPA_PASSPHRASE | sudo tee /etc/wpa_supplicant/wpa_supplicant.conf
sudo pkill wpa_supplicant
sudo pkill dhclient
sudo wpa_supplicant -i wlan0 -c /etc/wpa_supplicant/wpa_supplicant.conf &
sudo dhclient -v wlan0
ifconfig
```
This last line will display the wifi IP

## 6/ Add wifi auto-connect @ startup
Edit rc.local
```console
sudo nano /etc/rc.local
```
Add this at the end of the file, before exit 0
```console
date >> /var/log/wpa_startup.log
echo "[COMMAND] pkill wpa_supplicant" >> /var/log/wpa_startup.log
pkill wpa_supplicant >> /var/log/wpa_startup.log &
wait
echo "[COMMAND] wpa_supplicant" >> /var/log/wpa_startup.log
echo "[============================================================================]" >> /var/log/$(date -d "today" +"%Y%m%d%H%M")-WPA.log 
wpa_supplicant -i wlan0 -c /etc/wpa_supplicant/wpa_supplicant.conf >> /var/log/$(date -d "today" +"%Y%m%d%H%M")-WPA.log 2>&1 &
sleep 10
echo "[COMMAND] dhclient" >> /var/log/wpa_startup.log
dhclient -v wlan0 >> /var/log/wpa_startup.log 2>&1 &
```
Test by rebooting the unihiker
```console
sudo reboot
```
Remove the echo and append (>>) if you don't want growing logs files (can be useful for diagnostic)  
If you want a fixed IP addr, you can modify your dhclient config file in order to use a fixed IP

## 7/ Update apt sources and upgrade the system :
```console
sudo nano /etc/apt/sources.list
```
Replace all "httpredir.debian.org" by "archive.debian.org"  
Then :
```console
apt update && apt upgrade -y && apt dist-upgrade -y && apt autoremove -y
```

## 8/ Boot infos and console screen rotation (landscape usage) :
Unihiker uses the u boot bootloader  
Bootloader conf and overlays can be changed by editing
```console
sudo nano /boot/uEnv.txt
```
Overlays are available inside the device tree description (also there's a readme in order to properly manipulate them) :
```console
cd /boot/dtbs/4.4.143-67-rockchip-g01bbbc5d1312/rockchip/overlay
cat README.rockchip-overlays
```
The screen is connected to the spi interface and uses the fbtft driver (see /etc/modules for loaded modules).  
To change the screen behavior you have to edit the conf file inside the modprobe directory :
```console
sudo nano /etc/modprobe.d/fbtft_device.conf

# Somewhere in this file you'll find :
# options fbtft_device name=rpi-display gpios="reset:44,dc:46" busnum=2 width=240 height=320 rotate=0
# ...
```
Then edit rotate=0 to rotate=1 (Clockwise rotation) 

## 9/ Locales/Timezone/Keyboard layout - Install console-setup and armbian-config for easier configuration
### Console-setup
```console
# Install pkg
sudo apt install console-setup

# Setup font and font size (recommended : terminal with 6x12 font size)
sudo dpkg-reconfigure console-setup

# You can also edit this file, the wizard is easier ...
sudo nano /etc/default/console-setup
```

### armbian-config for timezone, locales and layout
```console
wget -qO - https://apt.armbian.com/armbian.key | gpg --dearmor | sudo tee /usr/share/keyrings/armbian.gpg > /dev/null
cat << EOF | sudo tee /etc/apt/sources.list.d/armbian-config.sources > /dev/null
Types: deb
URIs: https://github.armbian.com/configng
Suites: stable
Components: main
Signed-By: /usr/share/keyrings/armbian.gpg
EOF
sudo apt update
sudo apt -y install armbian-config
armbian-config 
```

### Bonus : If keyboard has still a bad layout
You must login on the tty of the Unihiker board for this step (not done through ssh)
```console
sudo dpkg-reconfigure keyboard-configuration
setupcon
sudo reboot
```
