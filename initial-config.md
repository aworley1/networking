# Initial RouterOS Configuration

## First Login

I first logged into the hEX with Winbox and was given the option to start with a blank configuration, or some typical home router defaults. I chose to clear the configuration, set a static IP on ether2, and a DHCP client on ether1, so I could connect to my current LAN and e.g. upgrade the OS.

## Upgrading the OS

RouterOS can be upgraded from within the OS. The hardware's firmware can also be upgraded in this way, and new firmware is released with new RouterOS versions. Log into the router via SSH and run the following:
```
/system package update check-for-updates
/system package update download
/system reboot
/system routerboard upgrade
/system reboot
```
