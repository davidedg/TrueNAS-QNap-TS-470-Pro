Follow instructions on repo [QNap-TS-470-Pro-USB-Copy-Button-and-Leds](https://github.com/davidedg/QNap-TS-470-Pro-USB-Copy-Button-and-Leds)
to generate the qts-hal archive.


    mkdir -p /var/db/system/custom/qts-hal
    QTSROOT=/var/db/system/custom/qts-hal && mkdir -p $QTSROOT && cp qts-hal.tar.gz $QTSROOT/ && cd $QTSROOT && tar xzf qts-hal.tar.gz && rm qts-hal.tar.gz

Clone the repo and prepare the scripts:

    sudo git clone https://github.com/davidedg/QNap-TS-470-Pro-USB-Copy-Button-and-Leds.git  /var/db/system/custom/copybutton
    sudo chmod +x /var/db/system/custom/copybutton/poll-copybutton
    sudo mv /var/db/system/custom/copybutton/qts-init-hal.bash /var/db/system/custom/qts-hal/qts-hal-init
    sudo chmod +x /var/db/system/custom/qts-hal/qts-hal-init

From the 

