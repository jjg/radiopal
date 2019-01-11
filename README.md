# Radiopal

Digital music player companion for your desktop radio.

## Description

Radiopal shuffles a collection of digital music files.  Playback controls consists of two pushbuttons (`play/pause` and `reject`) and an analog storage capacity guage.

~~The storage guage is driven using a binary weighted resistor digital-to-analog converter.  The reason for this is that the Raspberry Pi has no built-in analog/PWM GPIO, and readily-avaliable DAC add-ons do not provide the 0-10 volts required by the analog meter used in this project.  So instead of adding a DAC, amplifier, etc. I chose to build a suitable DAC from discreete passive components.~~

Looks like the Raspberry Pi *can* to PWM.  That being the case all that is needed to drive the 0-10v meter is a [suitable amplifier](https://www.raspberrypi.org/forums/download/file.php?id=11996&sid=553d761c5af52b6649be49f1df5d2a40).


## Hardware

### Components

* [Raspberry Pi 3 Model A+](https://www.raspberrypi.org/products/raspberry-pi-3-model-a-plus/)
* SD Card (the larger the better)
* 1-10VDC Linear voltmeter
* PN2907 General Purpose transistor
* 27k resistor
* Zener diode
* 12VDC power supply

### Configuration

(schematic goes here)


## Software

* [Raspbian Stretch Lite](https://www.raspberrypi.org/downloads/raspbian/)

## Configuration

Download Raspbian
Unzip Raspbian
Burn Raspbian to SD card
`sudo dd if=2018-11-13-raspbian-stretch-lite.img of=/dev/sdb bs=1MB`

Mount SD card and configure network
Create `/boot/wpa_supplicant.conf`

```
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1
country=«your_ISO-3166-1_two-letter_country_code»

network={
    ssid="«your_SSID»"
    psk="«your_PSK»"
    key_mgmt=WPA-PSK
}
```

Add SSH
Create an empty file called `/boot/ssh`

Unmount SD card, insert into Raspberry Pi & boot

SSH into Raspberry Pi
Change the default password (`passwd`)
Use `raspi-config` for basic O/S configuration:

* Advanced Options -> Expand Filesystem
* Advanced Options -> Audio -> Force 3.5mm
* Network Options -> Hostname

Exit `raspi-config` and reboot

Try ssh'ing into the Raspberry Pi using the new hostname.
Update the O/S (`sudo apt-get update && apt-get upgrade`)
Install vim: (`sudo apt-get install vim`)

### Optional: install and configure Samba

`sudo apt-get install samba samba-common-bin`
`sudo mkdir -m 1777 /music`
`sudo vim /etc/samba/smb.conf`

Create a network share for the music directory (note: this shares the directory with everyone on the network, you can tighten security if you know what you're doing):

```
[music]
comment = Radiopal music directory
path = /music
browseable = yes
writable = yes
only guest = no
create mask = 0777
directory mask = 0777
public = yes
guest ok = yes
```

`sudo smbpasswd -a pi`
`sudo /etc/init.d/samba restart`

At this point you should be able to browse to the `music` share on the Raspberry Pi.

### Optional: install and configure SyncThing

Now you should copy some music over (start with just a song or two until everything checks-out)

### Install MDP

`sudo apt-get install mpd`
`sudo apt-get install mpc`
`sudo vim /etc/mpd.conf`

Change the music directory to use the shared directory setup above:
`music_directory    "/music"`

`sudo systemctl restart mpd`

Test MPD:

`mpc update`
`mpc ls | mpc add`
`mpc volume 100`
`mpc random`
`mpc repeat`
`mpc play`

You should hear music.

### Automate music library updates

MPD needs to be told when new music files have been copied to the music directory.  Simplest way to do this is to schedule the `mpc` command:

`crontab -e`

Add the following to the crontab:

```
`*/5 * * * * /usr/bin/mpc update`
`*/5 * * * * /usr/bin/mpc ls | /usr/bin/mpc add`
```

### Firmware

Python script to test driving the meter:

```
import RPi.GPIO as GPIO
from time import sleep

GPIO.setmode(GPIO.BCM)
GPIO.setup(25, GPIO.OUT)    # set GPIO 25 as output for the PWM signal
D2A = GPIO.PWM(25, 1000)    # create object D2A for PWM on port 25 at 1KHz
D2A.start(0)                # start the PWM with a 0 percent duty cycle (off)

try:

    while True:
        dutycycle = input('Enter a duty cycle percentage from 0-100 : ')
        print "Duty Cycle is : {0}%".format(dutycycle)
        D2A.ChangeDutyCycle(dutycycle)
        sleep(2)

except (KeyboardInterrupt, ValueError, Exception) as e:
    print(e)
    D2A.stop()     # stop the PWM output
    GPIO.cleanup() # clean up GPIO on CTRL+C exit


def main():
    pass

if __name__ == '__main__':
    main()
```


# References

* [Prepare SD card for Wifi on Headless Pi](https://raspberrypi.stackexchange.com/questions/10251/prepare-sd-card-for-wifi-on-headless-pi)
* [Set up a Raspberry Pi as a File Server](https://www.raspberrypi.org/magpi/samba-file-server/)
* [Binary Weighted Resistor DAC](https://www.electronics-tutorial.net/analog-integrated-circuits/data-converters/binary-weighted-resistor-dac/index.html)
* [Tutorial: Digital to Analog Conversion](https://www.tek.com/blog/tutorial-digital-analog-conversion-r-2r-dac)
* [Arduino Drum Machine (example of resistor DAC)](http://little-scale.blogspot.com/2008/04/arduino-drum-machine.html)
* [GPIO - Raspberry Pi Documentation](https://www.raspberrypi.org/documentation/usage/gpio/)
* [HOWTO: Add an analog output to the Pi](https://www.raspberrypi.org/forums/viewtopic.php?p=833519)
* [PN2907 Transistor Datasheet](http://www.mouser.com/ds/2/149/pn2907-305713.pdf)
