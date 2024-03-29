##Kidddo Print Server Raspberry Pi
This guide will help you set up a [Kidddo](http://kidddo.com) Print Server on a Raspberry Pi. Portions of this guide are adapted from another [repository](https://github.com/churchio/checkin-printer).

If you are new to the Raspberry Pi, check out the following links:

* [What is a Raspberry Pi](http://www.youtube.com/watch?v=e0wkVVVLvR8#t=49)?
* [Where to Buy a Pi](https://www.google.com/shopping/product/16525736034140563056?q=raspberry+pi&client=safari&rls=en&bav=on.2,or.r_qf.&bvm=bv.59930103,d.cGU,pv.xjs.s.en_US.eYG7PyzNpLg.O&biw=1532&bih=1011&tch=1&ech=1&psi=CBvkUqGmBcmFogTU3oHgAQ.1390680841259.3&prds=hsec:online&ei=CRvkUtXXPIT9oATDxoHIBw&ved=0CIIEENkrMAA)

===
###Requirements
* A Raspberry Pi Model B with power adapter.
* An SD card with at least 1 GB of storage.
* A network cable.
* A DYMO LabelWriter 450 (or compatible label printer) connected via USB.

===
###Setup

####Get Raspbian
* Download Raspbian "Wheezy" from [here](http://www.raspberrypi.org/downloads) (Choose Direct download YYYY-MM-DD-wheezy-raspbian.zip).
* Copy the image to your SD card using your computer. ([Guide](http://elinux.org/RPi_Easy_SD_Card_Setup))
* Insert the SD card into your Raspberry Pi and plug it in. Make sure the network cable is plugged in as well.
* Locate your PI's IP address:
	* Use a network utility like [LanScan](https://itunes.apple.com/us/app/lanscan/id472226235?mt=12):![image](https://dl.dropboxusercontent.com/u/222645/Kidddo/github/screen_shot_2014-02-19_at_3.46.05_pm_7d9f.png)
	* OR log into your router and look at the DHCP lease table.
* SSH to your PI: `ssh pi@yourIPaddress` eg. `ssh pi@192.168.0.18`. The default password is `raspberry`.
* You can change your password by typing `passwd` immediately after login.

It's also recommended that you make a couple of configuration changes to your pi:

    sudo raspi-config

![image](https://dl.dropboxusercontent.com/u/222645/Kidddo/github/screen_shot_2014-02-25_at_5.44.54_pm_ec2b.png)

1. Expand Filesystem to fill up entire SD card
2. Advanced Options > Memory Split: Change to 16 (the minimum) since we're not using the graphic-intensive tasks

===
Let's make sure everything is up to date:

    sudo apt-get update

Then:

    sudo apt-get upgrade

===
*The following process takes about 20 minutes*

===
####Install Software
Create a directory for NodeJS - which is the platform that our Print Server will run on:

    sudo mkdir /opt/node

Download Node.js:

    wget http://nodejs.org/dist/v0.10.2/node-v0.10.2-linux-arm-pi.tar.gz

Unpack it:

    tar -xvzf node-v0.10.2-linux-arm-pi.tar.gz

Copy it to our directory:

    sudo cp -r node-v0.10.2-linux-arm-pi/* /opt/node

Open `/etc/profile` using "nano" (built-in editor) or use your editor of choice:

    sudo nano /etc/profile

And add the following before `export PATH`, exit & save:

    NODE_JS_HOME="/opt/node"
    PATH="$PATH:$NODE_JS_HOME/bin"

Logout:

    logout

Log back in via ssh. If node installed correctly, we can check the version:

    node -v

The response should be something like: `v0.10.2`. 

Next, we'll use NPM to install a few more modules needed for our PrintServer script (give it a few more minutes):
    
    npm install ipp pdfkit ibtrealtimesjnode


Ok, now let's make a new directory in your PI's home folder:

    mkdir kidddo

Change Directory to "kidddo":

    cd kidddo

Download Kidddo Print Server:

    git clone git://github.com/Kidddo/Raspberry-Pi-Print-Server

Change Directory to "Rasberry-Pi-Print-Server":

    cd Raspberry-Pi-Print-Server

Open PrintServer.js in Vim editor to add your organization's token:

    nano PrintServer.js

Replace `abc` on line 4 with your organization's token (found in Settings area of your Admin site: Settings>Set Printer>Server) and save the file.

####Automatically start print server on boot-up 

Create a start script for Node:

    nano print.sh

Put the following content in `print.sh` and exit/save:

    #!/bin/bash
    
    NODE=/opt/node/bin/node
    SERVER_JS_FILE=/home/pi/kidddo/Raspberry-Pi-Print-Server/PrintServer.js
    USER=pi
    OUT=/home/pi/kidddo/PrintServer.log
    
    case "$1" in
    
    start)
	    echo "starting node: $NODE $SERVER_JS_FILE"
	    sudo -u $USER $NODE $SERVER_JS_FILE > $OUT 2>$OUT &
	    ;;
    
    stop)
	    killall $NODE
	    ;;
    
    *)
	    echo "usage: $0 (start|stop)"
    esac
    
    exit 0

Make the script executable with 'chmod':

    chmod 755 print.sh

Copy it to '/etc/init.d':

    sudo cp print.sh /etc/init.d

Register the script as a service with 'update-rc.d':

    sudo update-rc.d print.sh defaults

####Configure Printers
Install CUPS (Common Unix Printing System):

    sudo aptitude install cups

Add your user (pi) to the to the lpadmin group (so we can manage printers):

    sudo usermod -aG lpadmin pi

Next, we'll make a few changes to the CUPS Configuration:

    sudo nano /etc/cups/cupsd.conf

Change `Listen localhost:631` to `Listen 0.0.0.0:631`:

    # Only listen for connections from the local machine.
    Listen 0.0.0.0:631
    Listen /var/run/cups/cups.sock
    
Add `Allow @LOCAL` to both the `<Location />` and `<Location /admin>` sections:

    # Restrict access to the server...
    <Location />
      Allow @LOCAL
      Order allow,deny
    </Location>
    
    # Restrict access to the admin pages...
    <Location /admin>
      Allow @LOCAL
      Order allow,deny
    </Location>

Save cupsd.conf

Restart CUPS:

    sudo service cups restart

Now we can leave the command line and open your web browser. Access the CUPS admin interface via the URL: `http://yourIPaddress:631/admin` eg. `http://192.168.0.18:631/admin`:

![image](https://dl.dropboxusercontent.com/u/222645/Kidddo/github/screen_shot_2014-02-21_at_9.53.34_am_c578.png)

Click the "Add Printer" button. If prompted, enter your user info (default "pi"/"raspberry").

Select Your label printer and click "Continue":

![image](https://dl.dropboxusercontent.com/u/222645/Kidddo/github/screen_shot_2014-02-21_at_9.57.56_am_ac13.png)

Enter a unique name for this printer and click "Continue". This is the name you will enter on all devices that print to this printer:

![image](https://dl.dropboxusercontent.com/u/222645/Kidddo/github/screen_shot_2014-02-21_at_9.59.29_am_9d61.png)

Click "Add Printer" on the Confirmation Screen.

In "Set Default Options" change the Media Size to "Shipping Address" and click "Set Default Options" (If you have modified the label template in PrintServer.js or are using a printer other than DYMO LabelWriter, adjust these options accordingly):

![image](https://dl.dropboxusercontent.com/u/222645/Kidddo/github/screen_shot_2014-02-21_at_10.04.57_am_b6a9.png)

Test your printer by selecting "Print Test Page" from the Maintenance Dropdown:

![image](https://dl.dropboxusercontent.com/u/222645/Kidddo/github/screen_shot_2014-02-21_at_10.11.16_am_636d.png)

Congrats! Repeat the process to add any additional printers. When finished, reboot your PI:

    sudo reboot

In your Kidddo Admin Settings tab, Enable "Use Label Printers" and click "Save Settings". Then click "Set Printer" and enter your selected printer name eg. `DYMO_1` and click Print Test Label:

![image](https://dl.dropboxusercontent.com/u/222645/Kidddo/github/screen_shot_2014-02-21_at_10.14.51_am_f18b.png)