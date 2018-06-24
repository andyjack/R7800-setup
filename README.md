# Setting up Netgear R7800 (Nighthawk X4S) with OpenWRT

## Connections

1. Connect to laptop with ethernet - use internal lan port
1. Power up the router
1. Reconfigure ethernet network on the laptop to use DHCP - should get a
   192.168.1.x address
1. Factory Netgear firmware is set up to listen on http://192.168.1.1

The factory firmware will require you to set up a password and some security
questions. You just need to be able to get to the firmware update page so a
secure/complex password isn't really necessary.

## Firmware Install

1. Following the [Factory
   Install](https://openwrt.org/docs/guide-quick-start/factory_installation)
   guide.
1. Pages for R7800 on OpenWRT
  * [Hardware data](https://openwrt.org/toh/hwdata/netgear/netgear_r7800)
  * [Table of Hardware](https://openwrt.org/toh/hwdata/netgear/netgear_r7800)
    page for looking up supported releases

If you are looking for newer releases of the firmware to upgrade, the target
arch is `ipq806x/generic`.  The release I used was:

http://downloads.lede-project.org/releases/17.01.4/targets/ipq806x/generic/

And I downloaded the `R7800-squashfs-factory.img` file.  Look at the
Supplementary Files section at the bottom for sha256 sums.

## Setup

I have a different box connected to the internet, so I just want a "dumb AP"
setup for the R7800:

* no firewall
* no dhcp

### Password setup

When the R7800 first boots after installing the OpenWRT firmware it will have
a login page - log in with the user `root` and no password.

Then you can go to the password setup page and add a password for `root`.  Log
out and back in again to confirm that the password works.

### Network Interface

[This page](https://openwrt.org/docs/guide-user/network/wifi/dumbap) is a good
starting point.  In the 17.01.4 release for the R7800 the LAN and the wireless
radios are already set up in a bridge, so no additional work is needed for
that part of the article.

1. Edit the LAN interface of the R7800. "Network -> Interfaces" menu; then
   click the "Edit" button of the LAN section.
1. Assign a static IP to the LAN interface. Choose one outside of the DHCP
   range set up on the main router.
  * set "IPv4 gateway" to the main router's IP
  * set "Use custom DNS Servers" to the main router's IP too - this is so the
    R7800 can access the internet for NTP
1. At the bottom of the edit page, check off "Disable DHCP for this
   interface".
  * Then select the IPv6 Settings tab, and set all of the following to
    `disabled`:
    * Router Advertisement-Service
    * DHCPv6-Service
    * NDP-Proxy

After saving these changes, the R7800 is now on a different IP and network.
You'll need to reconfigure your laptop's network to be able to connect to the
R7800.

The Dumb AP document specifies that you have to disable multicast snooping,
but in the 17.01.4 firmware out-of-the-box already has this set to `0`.

```
root@LEDE:~# cat /sys/devices/virtual/net/br-lan/bridge/multicast_snooping
0
```

### Firewall

Disable the firewall so it doesn't run.

1. "System -> Startup"
1. Click the "Enabled" button so it shows "Disabled".
1. Click "Submit" at the bottom of the page.

### Wireless setup

These can be found under "Network -> Wireless".

For both radios:

* Under "Device Configuration -> Advanced Settings", change country code to
  "CA"
* "Device Configuration -> General Setup"
  * 802.11nac radio: Mode AC, Channel 36, Width 80 MHz
  * 802.11bgn radio: Mode N, Channel 7 , Width 20 MHz
* "Interface Configuration -> General Setup" tab:
  * Set up the ESSID; Mode is "Access Point"
* "Interface Configuration -> Wireless Security" tab:
  * Encryption: WPA2-PSK; Cipher is "auto"; key is a secret!

## Securing LuCI

The web UI for managing the R7800 runs via http only, so it's rather insecure.
You should set up https instead - for the purpose of a home router a
self-signed certificate for https better than no encryption.

Use `ssh` to connect to the router using `root` as the username, and the
password you set up initially.  The `opkg` command is used for managing
package installation.

```
opkg update
opkg install luci-ssl-openssl
```

The `luci-ssl-openssl` package pulls in other dependencies for the webserver
(`uhttpd`) to support TLS. [This forum
posting](https://forum.lede-project.org/t/luci-over-https-luci-ssl-vs-luci-ssl-openssl/12828/4)
gave a quick overview of the setup.

After installing the package, it generates a private key and certificate for
use by `uhttpd`.  Restarting the service afterward Just Worked, because
`uhttpd` was configured out-of-the-box for TLS.  Now requests to the http port
are redirected to the https port.

```
/etc/init.d/uhttpd restart
```

More information on configuring `uhttpd` is
[here](https://openwrt.org/docs/guide-user/services/webserver/uhttpd).

## Backing up configuration

"System -> Backup / Flash Firmware" allows you to download an archive of the
configuration you worked so hard on.  Once you're done setting up the R7800
creating a backup of this configuration would be helpful for future upgrades.

## Final Setup

Replace the old wireless router with the R7800.  Before disconnecting the old
router, change the SSID and password to some nonsense values.

<!--
 vim:tw=78
-->
