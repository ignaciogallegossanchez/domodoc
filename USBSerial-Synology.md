# Compiling Synology kernel modules

Recently I bought a new Zigbee coordinator dongle called [Slae.sh](https://slae.sh/projects/cc2652/). This is a CC2652RB development stick, that can be used as [zigbee2mqtt](https://www.zigbee2mqtt.io/information/supported_adapters.html) coordinator.

The main problem is that I want to plug my USB dongle on a Synology NAS, and unfortunatelly Synology doesn't provide the required kernel module (by default).

Because of this we will need to build the kernel module with the needed driver.

## 🚀 Preparing the build environment

I'm using a OSX machine so the easyest way to build the kernel module is run a Docker container with Ubuntu Linux.

Let's run the Ubuntu container:

```
# docker run -it ubuntu
```

Then install some required software in the Ubuntu container:

```
root@38a7827808c0:/# apt-get update && apt-get -y install xz-utils build-essential libncurses-dev
```

## ⬇️ Download required Synology software

We will need to download two files from [Synology Open Source Project](https://sourceforge.net/projects/dsgpl/files/):

 - Synology Tool Chain: are tools like compilers, linkers, etc
 - Linux Kernel Souce code

First of all we will need to retrieve information about our Synology device, so SSH it:

```
# ssh -l admin my-synology-ip
```

And execute these two commands:

```
ash-4.3# uname -a
Linux synology 3.10.105 #25426 SMP Mon Dec 14 18:45:24 CST 2020 x86_64 GNU/Linux synology_braswell_216+II

ash-4.3# cat /etc.defaults/VERSION 
majorversion="6"
minorversion="2"
productversion="6.2.3"
buildphase="GM"
buildnumber="25426"
smallfixnumber="3"
builddate="2020/12/14"
buildtime="06:07:59"
ash-4.3#
```

Take note of this information:

 - SoC system: braswell (at the end of uname command)
 - Architecture: x86_64
 - Kernel version: 3.10.105
 - Build Number: 25426
 - DSM Version (product version): 6.2.3

With this information now we can download the software:

 1. Go to [Synology Open Source Project](https://sourceforge.net/projects/dsgpl/files/)
 2. Go to **ToolChain** -> **DSM 6.2.3 ToolChains** (DSM version) -> **Intel x86 Linux 3.10.105 (Braswell)** (or the SoC you have) -> `braswell-gcc493_glibc220_linaro_x86_64-GPL.txz` (x86_64 is the architecture I have)
 3. Go to **Synology NAS GPL Source** -> **25426branch** (or build you have) -> **x64-source** -> `linux-3.10.x.txz` (or the linux version you have)

 
## ⬆️ Put downloaded software on docker container

Now copy the software to the docker container:

```
# docker cp braswell-gcc493_glibc220_linaro_x86_64-GPL.txz 38a7827808c0:/root
# docker cp linux-3.10.x.txz 38a7827808c0:/root
```

(Be sure that your linux container id is the correct one, so change 38a7827808c0 by your container id)


## ⚙️ Compile the module

Compiling time! First uncompress the software:

```
# cd /root
# tar -xf braswell-gcc493_glibc220_linaro_x86_64-GPL.txz 
# tar -xf linux-3.10.x.txz
```

Now copy the default kernel configuration for our architecture (braswell in my case) to the kernel configuration:

```
root@38a7827808c0:~# cd linux-3.10.x/
root@38a7827808c0:~/linux-3.10.x# cp synoconfigs/braswell .config
```

Import the configuration and start menuconfig:

```
root@38a7827808c0:~/linux-3.10.x# ARCH=x86_64 CROSS_COMPILE=/root/x86_64-pc-linux-gnu/bin/x86_64-pc-linux-gnu- make oldconfig
root@38a7827808c0:~/linux-3.10.x# ARCH=x86_64 CROSS_COMPILE=/root/x86_64-pc-linux-gnu/bin/x86_64-pc-linux-gnu- make menuconfig
```

*(You will need to adapt arch, and path of toolchains to your specific case)*

And mark "**USB CP210x**" to be compiled as module (M):

```
Device Drivers  --->
    [*] USB support  --->
        <M>   USB Serial Converter support  --->
            <M>   USB CP210x family of UART Bridge Controllers
```

And save to ".config".

And finally **compile** the modules:

```
root@38a7827808c0:~/linux-3.10.x# ARCH=x86_64 CROSS_COMPILE=/root/x86_64-pc-linux-gnu/bin/x86_64-pc-linux-gnu- make -j10 modules
```

You will find the just compiled module under `drivers/usb/serial/cp210x.ko`.


## ✔️ Check that is OK

Copy the module to your Synology NAS.

Load the module executing in your synology (ssh):

```
# modprobe usbserial
# insmod cp210x.ko
```

If no errors, you should see something like this in your dmesg


```
# dmesg | tail -n 4
[ 7154.244106] usbserial: USB Serial support registered for cp210x
[ 7154.250765] cp210x 1-4:1.0: cp210x converter detected
[ 7154.378758] usb 1-4: reset full-speed USB device number 2 using xhci_hcd
[ 7154.419994] usb 1-4: cp210x converter now attached to ttyUSB0
```


## ⚙️ Loading module at boot time

We should be able to make this in a easy way, but Synology lacks of a lot of normal linux distribution executables, so we must load the new .ko file using a script.

First move the kernel module file where the Synology kernel modules are:

```
mv cp210x.ko /lib/modules/
```

And create the file **/usr/local/etc/rc.d/cp210xkmod.sh** with this content:

```
#!/bin/sh

. /etc/rc.subr

case "$1" in
start)
    SYNOLoadModules usbserial cp210x
    ;;
stop)
    SYNOUnloadModules usbserial cp210x
    ;;
*)
    echo "usage: $0 { start | stop }" >&2
    exit 1
    ;;
esac
```

Finally put the correct permissions in the new created file with the command:

```
chmod 755 /usr/local/etc/rc.d/cp210xkmod.sh 
```


## 📌 Persistent device name 

Now, our device is detected as "**ttyUSB0**", but this name is not persistent and can change depending of which devices are connected, the boot order, or other external factors.


So it's a good idea to make udev to create a symbolic link to our ttyUSBx device with a fixed name.

To make this first get all neded information about our USB stick:

```
# udevadm info -a -p  $(udevadm info -q path -n /dev/ttyUSB0) | grep -m1 {idVendor}
    ATTRS{idVendor}=="10c4"
# udevadm info -a -p  $(udevadm info -q path -n /dev/ttyUSB0) | grep -m1 {idProduct}
    ATTRS{idProduct}=="ea60"
# udevadm info -a -p  $(udevadm info -q path -n /dev/ttyUSB0) | grep -m1 {serial}
    ATTRS{serial}=="00_12_4B_00_21_CC_4D_BA"    
```

Now we will create a new udev rule to make the link with the name we choose, in this case "zigbeedongle". To make this first edit the file **/usr/lib/udev/rules.d/99-slaesh.rules** with the content:

```
SUBSYSTEM=="tty", ATTRS{idVendor}=="10c4", ATTRS{idProduct}=="ea60", ATTRS{serial}=="00_12_4B_00_21_CC_4D_BA", SYMLINK+="zigbeedongle"
```

We can check the new configuration reloading the new configuration:

```
udevadm control --reload
```

And unpluging and pluging again the USB dongle. We should see that the symbolic link is correctly created:

```
# ls -l /dev/zigbeedongle 
lrwxrwxrwx 1 root root 7 Mar 25 20:05 /dev/zigbeedongle -> ttyUSB0
#
```

Now we have the the dongle available for docker under **/dev/zigbeedongle** ready to be used 

## 📌 Persistent device name and Docker 🐳

**Unfortunatelly**, if you're trying to use the symbolic link from docker using the -device flag, you will see that cannot be used (not found error).

For example I'm getting this error from zigbee2mqtt docker:

```
Zigbee2MQTT:error 2021-03-25 22:10:45: Error: Error while opening serialport 'Error: Error: No such file or directory, cannot open /dev/zigbeedongle
```

Why? The symbolink link device is, as name says, a symbolink link from `/dev/zigbeedongle` to the real device which we have not access to from docker container. In this case `/dev/ttyUSB`.

In other words, you should make /dev/ttyUSB0 accesible also... so we are depending again of the original name.

Options?

### Make udev change the device name instead symlink'ing it 😞

I've configured udev to change the name of the device with **NAME** directive, but seems not supported.

I'm getting this very explicative error.

```
# udevadm test /devices/pci0000:00/0000:00:14.0/usb1/1-4/1-4:1.0/ttyUSB0/tty/ttyUSB0
...
NAME="zigbeedongle" ignored, kernel device nodes can not be renamed; please fix it in /usr/lib/udev/rules.d/99-slaesh.rules:1
...
```

### Make /dev also available for docker

The other option is make all /dev path available for container with `-v /dev:/dev` directive when executing the zigbee2mqtt docker container.

I've not choose this because my *host* device is a NAS, and I dont want unexpected behaviors with devices.

</br>
</br>


```
Ignacio Gallegos Sánchez
ignacio.gallegos@gmail.com
```
