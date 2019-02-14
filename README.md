# Setting up Netgear R7800 (Nighthawk X4S) with OpenWRT

## Product Pages

* https://www.netgear.com/support/product/R7800
* https://kb.netgear.com/30180/R7800-FAQs

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

http://downloads.openwrt.org/releases/18.06.1/targets/ipq806x/generic/

And I downloaded the `R7800-squashfs-factory.img` file.  Look at the
Supplementary Files section at the bottom for sha256 sums.

## Sysupgrade.bin and TFTP recovery

After two months of running the OpenWRT firmware, I tried using the [upgrade
instructions](https://openwrt.org/docs/guide-quick-start/sysupgrade.luci) to
upgrade from 17.04 to 18.06. However I ran into the power led blinking after
the reboot. 192.168.1.1 was pingable but the LEDE web UI would not load. It
was like that for several minutes. It turned out this was a recovery state of
the router due to a [failed firmware
upgrade](https://kb.netgear.com/000059633/How-to-upload-firmware-to-a-NETGEAR-router-using-TFTP-client).
By running a [TFTP
upload](http://www.i-helpdesk.com.au/index.php?/Knowledgebase/Article/View/559/0/uploading-firmware-using-mac-osxunix-via-tftp)
with the `-squashfs-factory.img` file, it was possible to get the router
functional again.

```
$ tftp 192.168.1.1
tftp> binary
tftp> put openwrt.img
Sent 8257665 bytes in 1.6 seconds
```

The router immediately rebooted after the TFTP upload was done.

### Upgrading from 18.06.1 to 18.06.2

Attempting to use the `-squashfs-sysupgrade.bin` here:

http://downloads.openwrt.org/releases/18.06.2/targets/ipq806x/generic/

Verified the integrity of the downloaded firmware with gpg.  First downloaded
a backup of the settings on the **System** -> **Backup / Flash Firmware**
page.

Then uploaded the upgrade firmware.  I checked off the "Keep settings" box,
assuming that since we're only doing a minor revision bump, the settings
should be compatible across versions.

During the process - the power LED went to solid orange, then white blinking,
then solid white.  Success!

Pinging the router during the upgrade process let me know it came back on the
same IP.  However I couldn't log in with https, I had to use
`http://<router_ip>` to use the web UI.  **BUT** logging in wouldn't work?

I could log in via `ssh` though - after doing so I went through the process
for installing openssl support of the web UI:

```
opkg update
opkg install luci-ssl-openssl
/etc/init.d/uhttpd restart
```

Then I was able to use `https://<router_ip>` to log in.

Because I was using the root password (unsuccessfully) on a web form thru an
insecure http connection, I changed the root pw to a new one with the `passwd`
command.

I did have to go over to "System -> Startup" and disable `dnsmasq` and
`firewall` again. Other customizations in this doc appeared to have survived
the upgrade.

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
However as of the 18.06 version it will try to confirm the changes after
waiting 30 seconds in the browser. This will fail since the UI is listening on
the new IP, so it will revert to the old IP and display a message to retry the
changes.  Use the "Apply Unchecked config" option to re-do the changes so they
don't revert.

When the interface IP is updated, you'll need to reconfigure your laptop's
network to be able to connect to the R7800.

The Dumb AP document specifies that you have to disable multicast snooping,
but in the 18.06.1 firmware out-of-the-box already has this set to `0`.

```
root@LEDE:~# cat /sys/devices/virtual/net/br-lan/bridge/multicast_snooping
0
```

### Firewall and dnsmasq

Disable these services so they don't run.

1. "System -> Startup"
1. For each of `firewall` and `dnsmasq` in the listing:
  * Click the "Enabled" button so it shows "Disabled"
  * Click the "Stop" button to stop the running instance of the service
  * Click "Submit" at the bottom of the page

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
  * Enable KRACK protection on session auto-renegotiation

See also [the OpenWRT
doc](https://openwrt.org/docs/guide-quick-start/walkthrough_wifi) on wifi
setup.

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

However the sysupgrade documentation says that it's probably a good idea to
start over again from defaults for new major releases, so saving the backups
might only be useful for reference.

## Final Setup

Replace the old wireless router with the R7800.  Before disconnecting the old
router, change the SSID and password to some nonsense values.

## System Info

| | |
|---|---|
| Hostname | OpenWrt |
| Model | Netgear Nighthawk X4S R7800 |
| Architecture | ARMv7 Processor rev 0 (v7l) |
| Firmware Version | OpenWrt 18.06.2 r7676-cddd7b4c77 / LuCI openwrt-18.06 branch (git-19.020.41695-6f6641d) |
| Kernel Version | 4.14.95 |

<!--
 vim:tw=78
-->
