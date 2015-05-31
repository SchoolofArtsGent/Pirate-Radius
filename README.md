# Pirate-Radius
An anonymous and completely open offline local wireless network for filesharing and chat. All uploaded soundfiles are put in a loop and broadcasted locally on an FM frequency of choice. A tool of convenience for every co-working space and studio. Based on [PirateBox](http://piratebox.cc) and [PiFM](http://www.icrobotics.co.uk/wiki/index.php/Turning_the_Raspberry_Pi_Into_an_FM_Transmitter).

## Hardware requirements

* A Raspberry Pi. v1 works for sure, other models need testing.
* An SD card for the Pi.
* A powersupply for the Pi.
* An ethernet cable and working internet connection. (Only needed during installation.)
* A USB wifi dongle with support for access point mode. See http://elinux.org/RPi_USB_Wi-Fi_Adapters. We used this one: http://www.dhgate.com/product/150m-usb-wifi-wireless-network-card-lan-adapter/138478687.html. We also tested the TP LINK TL-WN821N v4 (ID 0bda:8178, chipset RTL8192CU) and it works with a small modification. See  http://www.tp-link.com/en/products/details/cat-11_TL-WN821N.html.
* A 20cm piece of electric wire, for the antenna.
* An FM radio for listening.
* A nice enclosure. We built a birdhouse.

Not strictly needed, but makes installing easier: a screen with HDMI cable, and a USB keyboard and mouse.

## Preparing the Pi

* Install Raspbian onto the SD card: http://www.raspberrypi.org/documentation/installation/installing-images/.
* If you have them available: connect screen, keyboard and mouse.
* Connect the wire for the antenna to pin 7. (This works on a V1 Pi, other models might be different.)
* Insert the card in the Raspberry Pi, connect ethernet, USB wifi and finally the power supply.
* If you don't have a screen, keyboard and mouse: use SSH to connect to the Pi. You'll need to find out it's IP address. See http://www.raspberrypi.org/documentation/troubleshooting/hardware/networking/ip-address.md.
* Once you're inside the Pi, expand the file system using raspi-config and reboot.

## Installing Raspberry Pi(rate)Box

The instructions below are copied from [PirateBox wiki](http://piratebox.cc/raspberry_pi:diy:manual). The material on the wiki is available under a CC license and is reproduced here for clarity.

First we turn the Raspberry Pi into a wireless access point:

	sudo apt-get update
	sudo apt-get -y install lighttpd
	sudo /etc/init.d/lighttpd stop
	sudo update-rc.d lighttpd remove
	sudo apt-get -y install dnsmasq
	sudo /etc/init.d/dnsmasq  stop
	sudo update-rc.d dnsmasq remove
	sudo apt-get -y  install hostapd
	sudo /etc/init.d/hostapd  stop
	sudo update-rc.d hostapd remove
	sudo apt-get -y install iw
	sudo rm /bin/sh
	sudo ln /bin/bash /bin/sh
	sudo chmod a+rw /bin/sh

Replace the contents of /etc/network/interfaces on the Pi with:

	auto lo

	iface lo inet loopback
	iface eth0 inet dhcp

	iface wlan0 inet manual
	### disalbed for PirateBox
	#allow-hotplug wlan0
	#wpa-roam /etc/wpa_supplicant/wpa_supplicant.conf
	#iface default inet dhc

Install the Piratebox sofware on the Pi:

	wget  http://downloads.piratebox.de/piratebox-ws_current.tar.gz
	tar xzf piratebox-ws_current.tar.gz
	cd piratebox
	sudo mkdir -p  /opt
	sudo cp -rv  piratebox /opt
	cd /opt/piratebox
	sudo sed 's:DROOPY_USE_USER="no":DROOPY_USE_USER="yes":' -i  /opt/piratebox/conf/piratebox.conf
	sudo ln -s /opt/piratebox/init.d/piratebox /etc/init.d/piratebox
	sudo update-rc.d piratebox  defaults
	sudo /etc/init.d/piratebox start

If everything went ok you should see a wifi network now: 'PirateBox - Share Freely'. If you connect to it, any webpage should redirect you to the PirateBox interface. From there you can chat and upload files.

## Alternative: Installing Raspberry Pi(rate)Box from GitHub
git clone https://github.com/SchoolofArtsGent/PirateBoxScripts_Webserver
cd /home/pi/PirateBoxScripts_Webserver/piratebox
sudo ./install.sh
sudo update-rc.d piratebox  defaults

# Using the TP LINK TL-WN821N v4 wifi stick

See also: https://www.raspberrypi.org/forums/viewtopic.php?f=36&t=82403.

Check if this the model you have

	lsusb

Stick should show up as:

	ID 0bda:8178 Realtek Semiconductor Corp. RTL8192CU 802.11n WLAN Adapter

Needs custom version of hostapd

	sudo mv /usr/sbin/hostapd /usr/sbin/hostapd_orig
	wget http://dl.dropbox.com/u/1663660/hostapd/hostapd.zip
	unzip hostapd.zip
	sudo mv hostapd /usr/sbin/
	sudo chmod +x /usr/sbin/hostapd

Optional: test with

	sudo hostapd -d /opt/piratebox/conf/hostapd.conf


## Installing PiFM

Now let's set up the radio part. This will broadcast all uploaded audio files on FM. Range is between 50 and 100 meters.

Download the PiFM software.

	cd ~
	wget http://omattos.com/pifm.tar.gz
	tar xvzf pifm.tar.gz

We will use a script to automate the radio (convert format and add to playlist). There are a couple of examples floating around on the internet. We used http://bytesare.us/cms/index.php/geeky-toys/pi-as-fm-radio-mp3-transmitter. (Author unknown, if you read this let us know so we can give credit.) First install the required software:

	sudo apt-get install libav-tools sox libsox-fmt-all

Create radio.sh:

	nano radio.sh

Copy/pase this script into radio.sh, and then save the file with Control-X, Y, and enter. Change the settings for the broadcast frequency and shuffling as needed.

	#!/bin/bash

	MUSIC_ROOT="/opt/piratebox/share/Shared"
	PIFM_BINARY="/home/pi/pifm/pifm"
	PIFM_FREQUENCY=92
	LOG_ROOT="/home/pi/pifm/logs"
	SHUFFLE="true" # true | false
	WHITELIST="3gp|aac|flac|m4a|m4p|mmf|mp3|ogg|vox|wav|wma"

	#####################

	LOG="$LOG_ROOT/pifm.log"
	mkdir -p "$LOG_ROOT"

	TEMP_FILES_PREFIX='pifm.radio.tempfile'
	TEMP_FILES_PATTERN="$TEMP_FILES_PREFIX.XXXXX"

	{
	    echo; echo -n "script $0 starting at"; date
	    echo "MUSIC_ROOT: $MUSIC_ROOT"

	    iteration=0
	    while [ 1 ] # run forever...
	    do
	        iteration=$(( $iteration + 1 ))
	        echo -n "start with iteration $iteration of playing all files in $MUSIC_ROOT at "; date
	        rm -vf "/tmp/$TEMP_FILES_PREFIX."*

	        # Collecting the songs in the specified dir

	        songListFile="$( mktemp "$TEMP_FILES_PATTERN" )"

	        find "$MUSIC_ROOT" -type f -follow \
	        | grep -iE ".*\.($WHITELIST)$" \
	        | sort \
	        > "$songListFile"

	        songCount="$( wc -l "$songListFile" | grep -Eo '^[0-9]*' )"

	        if [ "x" = "x$songCount" ]; then echo "FATAL: no songs could be found in $MUSIC_ROOT"; exit 2; fi
	        if [ $songCount -lt 1 ];    then echo "FATAL: no songs could be found in $MUSIC_ROOT"; exit 2; fi

	        # Generate a playlist from the results

	        playlist="$( mktemp "$TEMP_FILES_PATTERN" )"
	        if [ $SHUFFLE = "true" ]; then
	            # prefix each line with random number, sort numerically and cut of leading number ;-)
	            cat "$songListFile" \
	            | while read song; do echo "${RANDOM} $song"; done \
	            | sort -n \
	            | cut -d " " -f 2- \
	            | while read song; do echo "${RANDOM} $song"; done \
	            | sort -n \
	            | cut -d " " -f 2- \
	            > "$playlist"
	        else
	            cp "$songListFile" "$playlist"
	        fi

	        # Play each song from the playlist

	        echo "will now air $songCount songs of $playlist on frequency $PIFM_FREQUENCY, enjoy!"
	        cat "$playlist" \
	        | while read song
	        do
	            # simple version: take 1st audio channel:
	            #
	            # command="avconv -v fatal -i '$song' -ac 1 -ar 22050 -b 352k -f wav - | '$PIFM_BINARY' - $PIFM_FREQUENCY"

	            # extended version: merge audio channels:
	            #
	            # merge the channesl of the song to one mono channel, write to stdout:
	            #     sox '$song' -t wav - channels 1
	            #
	            # read mono audio from stdin, convert into pifm format and write to stdout:
	            #     avconv -v fatal -i pipe:0 -ac 1 -ar 22050 -b 352k -f wav -
	            #
	            # read compatible audio data from stdin and play with pifm at specified frequency:
	            #     '$PIFM_BINARY' - $PIFM_FREQUENCY
	            command="sox '$song' -t mp3 - channels 1 | avconv -v fatal -i pipe:0 -ac 1 -ar 22050 -b 352k -f wav - | '$PIFM_BINARY' - $PIFM_FREQUENCY"



	            echo "$command # $( date )"
	            bash -c "$command"

	        done # with playlist
	        echo -n "done with iteration $iteration at "; date

	    done # with endless loop :-)

	} 2>&1 | tee -a "$LOG"

Make the script executable:

	sudo chmod +x radio.sh

Upload some files to the PirateBox and start the script with:

	sudo ./radio.sh

## Tweaks

### Start the radio script automatically on boot

	sudo crontab -e

Add this line:

	@reboot     /home/pi/radio.sh

### Logging in to the Pi via SSH

* Connect to the Pi's wifi network.
* ssh pi@192.168.77.1

### Changing the wifi network name (SSID)

* sudo nano /opt/piratebox/conf/hostapd.conf
* sudo reboot

### Deleting files

* sudo rm /opt/piratebox/share/Shared/example.mp3

### Making the shutdown button work

On Pi:

	sudo visudo

Add:

	nobody ALL=NOPASSWD:/sbin/halt


# Optional: updating website design

## Preparation on Pi
Make sure the git repository is on the Pi. You should have this folder:

	/home/pi/PirateBoxScripts_Webserver

If not, run

	cd ~
	git clone https://github.com/SchoolofArtsGent/PirateBoxScripts_Webserver

Make the update script executable. Only needed once:

	chmod +x /home/pi/PirateBoxScripts_Webserver/update_website.sh

## Making changes to the design on your computer
Clone the repo on your computer:

	git clone https://github.com/SchoolofArtsGent/PirateBoxScripts_Webserver

Files for design can be found in:

	piratebox/piratebox/www

We use Foundation 5. Changes to CSS should made in

	piratebox/piratebox/www/css/radius.css

Changes to the HTML of the home page should be made in

	piratebox/piratebox/www/css/index.html

Changes to the HTML of the file listing should be made in

	piratebox/piratebox/www/cgi-bin/shared_files.py

To link to file listing: use /cgi-bin/shared_files.py, not /Shared as on default PirateBox installation.

Make sure not remove the chat (div id shoutbox) and file upload (div id upload). You also want to keep the scripts.js in there.

To get an idea of how the design will look like: open index.html in your browser. Chat and file upload won't work while your testing on your computer.

## Updating Pi with changes
Make sure the Pi has an internet connection.

Each time you want to update the design on the Pi, do this:

	cd /home/pi/PirateBoxScripts_Webserver
	./update_website.sh
