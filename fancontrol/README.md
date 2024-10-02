# FAN CONTROL 

Login via ssh or use Web shell.

Download and uncompress the fancontrol package:

    sudo mkdir -p /var/db/system/custom/fan
   
    DEBIANVER=$(lsb_release -cs 2>/dev/null)
    PACKAGEINFO=$(wget  -qO- "https://packages.debian.org/$DEBIANVER/all/fancontrol/download")
    URI=$(echo "$PACKAGEINFO" | grep -oP 'href="\K[^"]*' | grep -E "\.deb$" |  sed -E 's|https?://[^/]+||' | grep ^/debian/pool/main|head -n1)
    sudo wget "http://ftp.debian.org$URI" -O /var/db/system/custom/fan/fancontrol.deb
    sudo /usr/bin/dpkg-deb -x fancontrol.deb /var/db/system/custom/fan/fancontrol

\
Create a script to start the service at boot

    sudo tee /var/db/system/custom/fan/fancontrol.init <<EOF
    #!/bin/bash
    
    modprobe f71882fg || exit 5
    [[ -r /etc/fancontrol ]] || ln -s /var/db/system/custom/fan/fancontrol.config /etc/fancontrol
    
    if systemctl is-active --quiet fancontrol; then
      systemctl stop fancontrol
    fi
    
    systemd-run --unit=fancontrol --description="fancontrol" \\
     --property=ConditionFileNotEmpty=/etc/fancontrol \\
     --property=Restart=on-failure \\
     --property=PIDFile=/run/fancontrol.pid \\
     /var/db/system/custom/fan/fancontrol/usr/sbin/fancontrol
    EOF

    #make it executable
    chmod +x /var/db/system/custom/fan/fancontrol.init

\
Prepare the configuration by running - save it under /var/db/system/custom/fan/fancontrol.config

    sudo /var/db/system/custom/fan/fancontrol/usr/sbin/pwmconfig

or use this pre-made one:

    sudo tee /var/db/system/custom/fan/fancontrol.config <<EOF
    INTERVAL=8
    DEVPATH=hwmon1=devices/platform/coretemp.0 hwmon2=devices/platform/f71882fg.2592
    DEVNAME=hwmon1=coretemp hwmon2=f71869a
    FCTEMPS=hwmon2/device/pwm2=hwmon1/temp2_input
    FCFANS= hwmon2/device/pwm2=hwmon2/device/fan2_input
    MINTEMP=hwmon2/device/pwm2=30
    MAXTEMP=hwmon2/device/pwm2=60
    MINSTART=hwmon2/device/pwm2=150
    MINSTOP=hwmon2/device/pwm2=0
    MAXPWM=250
    EOF

\
Add SystemD Sleep hook [not tested]

    sudo mkdir /etc/systemd/system-sleep
    sudo ln -s /var/db/system/custom/fan/fancontrol/lib/systemd/system-sleep/fancontrol /etc/systemd/system-sleep/
    sudo systemctl daemon-reload

\
In TrueNAS, goto System -> Advanced Setings -> Init/Shutdown Scripts
\
Add a Pre-Init script with path `/var/db/system/custom/fan/fancontrol.init`

