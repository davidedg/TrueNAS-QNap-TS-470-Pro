Login via ssh or use Web shell.
\
Download LCD Control [repo](https://github.com/davidedg/Qnap-TS-470-Pro-Lcd-Scale.git):

    sudo mkdir -p /var/db/system/custom/ && cd /var/db/system/custom/
    sudo git clone https://github.com/davidedg/Qnap-TS-470-Pro-Lcd-Scale.git lcd

\
In TrueNAS, goto System -> Advanced Setings -> Init/Shutdown Scripts and add:
- shutdown command: /bin/env python3 /var/db/system/custom/lcd/shutdown.py
- pre-init script: /bin/env python3 /var/db/system/custom/lcd/preinit.py
- post-init command: (/bin/env python3 /var/db/system/custom/lcd/lcd-menu.py) >/dev/null 2>&1 &
