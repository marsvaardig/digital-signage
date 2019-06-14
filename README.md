#   Digital Signage with Raspberry Pi

Run Raspberry 3 in digital signage mode with Vivaldi Web browser on a TV screen


##  Install Raspbian

- [Download Raspbian Stretch Lite](https://www.raspberrypi.org/downloads/raspbian/)
- [Follow installation instructions](https://www.raspberrypi.org/documentation/installation/installing-images/README.md)


## Enable SSH Access

Create an empty file in the root of the SD card named ssh (without dot or extension).

    $ touch /Volumes/boot/ssh


##  Install Digital Signage

Login via SSH: `ssh pi@ip-address`. Password is `raspberry`.

    sudo apt-get update
    sudo apt-get dist-upgrade -y
    sudo apt-get install -y matchbox chromium-browser cec-utils xinit x11-xserver-utils ttf-mscorefonts-installer xwit sqlite3 libnss3 libgtk-3-0 xserver-xorg-legacy
    sudo systemctl reboot
   
### Troubleshoot

If a newly flashed Raspberry shares the IP address of an old Pi then remove it from `know_hosts`: `ssh-keygen -R [IP]`


## Install Vivaldi until Chromium in Raspbian is at v74

    wget https://downloads.vivaldi.com/stable/vivaldi-stable_2.5.1525.48-1_armhf.deb

    sudo dpkg -i vivaldi-stable_2.5.1525.48-1_armhf.deb


##  Tweak Pi's config

Edit Pi's own configuration file `sudo nano /boot/config.txt`. This works for me on a LG Full HD TV screen:

~~~
disable_overscan=1
framebuffer_width=1920
framebuffer_height=1080
hdmi_force_hotplug=1
hdmi_group=1
hdmi_mode=16
~~~

Check the best mode by running `/opt/vc/bin/tvservice -m CEA`. The TV screen has activated CEC support (called _simplink_). I use this feature to automatically turn on the TV while Pi is booting.


##  Prepare X server

Because there is currently (January 2017) a [bug in X/xinit](https://bugs.launchpad.net/ubuntu/+source/xinit/+bug/1562219) we need more privileges on `/dev/tty*`. Edit `sudo nano /etc/rc.local`. Add this:

~~~ {.bash}
if [ -f /boot/xinitrc ]; then
    chmod 0660 /dev/tty*
    ln -fs /boot/xinitrc /home/pi/.xinitrc;
    su - pi -c 'startx' &
fi
~~~

Add before `exit 0`

## Pi user

Allow user `pi` to start the X server:

    sudo dpkg-reconfigure xserver-xorg-legacy # Set to”anybody”

GUI which changes setting in `/etc/X11/Xwrapper.config`

## Boot browser

Create file `sudo nano /boot/xinitrc`:

~~~ {.bash}
#!/bin/sh

while true; do
    # Turn on TV
    echo "on 0" | cec-client -s

    # Clean up previously running apps, gracefully at first then harshly
    killall -TERM chromium-browser 2>/dev/null;
    killall -TERM matchbox-window-manager 2>/dev/null;
    sleep 2;
    killall -9 chromium-browser 2>/dev/null;
    killall -9 matchbox-window-manager 2>/dev/null;

    # Clean out existing profile information
    rm -rf /home/pi/.cache;
    rm -rf /home/pi/.config;
    rm -rf /home/pi/.pki;

    # Generate the bare minimum to keep Chromium happy!
    mkdir -p /home/pi/.config/chromium/Default
    sqlite3 /home/pi/.config/chromium/Default/Web\ Data "CREATE TABLE meta(key LONGVARCHAR NOT NULL UNIQUE PRIMARY KEY, value LONGVARCHAR); INSERT INTO meta VALUES('version','46'); CREATE TABLE keywords (foo INTEGER);";

    # Disable DPMS / Screen blanking
    xset -dpms
    xset s off

    # Reset the framebuffer's colour-depth
    fbset -depth $( cat /sys/module/*fb*/parameters/fbdepth );

    # Hide the cursor (move it to the bottom-right, comment out if you want mouse interaction)
    xwit -root -warp $( cat /sys/module/*fb*/parameters/fbwidth ) $( cat /sys/module/*fb*/parameters/fbheight )

    # Start the window manager (remove "-use_cursor no" if you actually want mouse interaction)
    matchbox-window-manager -use_titlebar no -use_cursor no &

    vivaldi-stable --disable-features=TranslateUI --app="https://marsvaardig.staging.marsvaardig.eu?option=com_signage&view=pages&tmpl=signage"
done;
~~~


## Reboot

    sudo systemctl reboot


## Performance (optional)

For smooth running of Vivaldi on Raspberry Pi, we recommend to increase swap space. Open a “Terminal” and use the following command to change the SWAP from 100MB to change it to 2048MB:

    echo CONF_SWAPSIZE=2048 | sudo tee -a /etc/dphys-swapfile
    sudo /etc/init.d/dphys-swapfile restart


##  Enjoy the show

Reboot your Pi and you are hopefully done.

Credits to [https://gist.github.com/dampfklon/990005451ad7f7d0c57374df95a902ed](https://gist.github.com/dampfklon/990005451ad7f7d0c57374df95a902ed)
