# Routerify: turns a Linux system into a WiFi router

## Introduction

This playbook configures a supported Linux system to work as a wireless
router.

For LAN connectivity, it can behave as a 2.4Ghz AP, 5GHz AP, and wired
switch at the same time, each element being optional. Each
wired/wireless LAN client can communicate with other clients. IPAM is
handled via `dnsmasq`, providing DHCP and DNS to all clients. The LAN IP
range and initial reservations are customizable.

WAN connectivity is expected to be provided via an authenticated PPPoE
link. `iptables` rules control forwarding between the LAN and the
Internet, applying IP masquerading to outgoing traffic. All incoming
traffic is blocked by default, with the exception of responses to
LAN-initiated traffic.

The following image depicts relationships between network elements. The
scenario involves a separate ONT for FTTH connectivity, which handles
the server-side part of PPPoE.

![Network diagram][network-diagram]

Once configured, the target system should be able to dial a PPP
connection, assign addresses to WiFi/wired clients, and handle outgoing
NAT'd communications, just like a residential router would.

_WARNING: since this playbook configures radio devices, you should read
the full documentation before trying to run it. Misconfiguring the
WiFi regulatory domain may result in generating interferences with other
equipment, and is illegal in many countries._

## Limitations

At the moment, some of the limitations of the playbook are:

* It can only configure a Debian 10 system; it _may_ work with other
  Debian versions or derivatives, but this has not been tested;
* No UI for configuration; tweaks must be applied via the command line
  (e.g., forwarding ports);
* It doesn't handle more than one wired NIC (although multiple wired
  clients can be connected simply by attaching an L2 switch to the NIC
  configured for wired connectivity);
* Tweaking AP settings requires editing `hostapd` configuration files
  within the playbook itself;
* Fixed structure for WAN connectivity;
* IPv6 is completely disabled.

## Structure

`ansible/routerify.yaml` contains the plays, which simply call on roles
from `ansible/roles/`. Almost all configuration is done via Ansible
variables. Most of them are kept within the `defaults` of the role to
which they pertain logically.  A few variables that did not fit nicely
in any roles were placed in `ansible/group_vars/all/00-defaults.yaml`.
Thus, the simplest way to configure your setup is to add an extra file
under `ansible/group_vars/all` that comes _lexicographically after_ the
defaults, and use it to override both globals and role defaults.

Currently, some configuration facets require editing role templates. An
example of this is the `hostapd` configuration. Changing the SSID
or the AP channel can be done via variables, but lower-level settings
can only be changed by editing the files.

## How to use

* Install a base Debian 10 system. There are no particular constraints,
  so you can partition the disks as you like, and install any additional
  tools that you want. If you plan to run the playbook on the system
  itself, Ansible must be installed. If a remote controller is used,
  `ssh` access must be granted. Also, `sudo` must be installed, as it's
  the default privilege escalation method used by Ansible;
* Configure the playbook according to your setup, via Ansible variables
  and template files when needed;
* Run the playbook;
* Reboot the system to apply the new configuration.

A minimal configuration file would look like this:

```yaml
# SSID used for both 2.4GHz and 5GHz networks, 2.4GHz
# will get a "-24" suffix
ap_common_ssid: MY_SSID

# WARNING: set this according to your local regulations
ap_common_country: IT

# WPA password used for both 2.4GHz and 5GHz networks
ap_common_wifi_password: I_@m_s3cuR3

# Replace all MAC's with the real addresses of the
# devices you want to use
ap24_interface_mac: 01:02:03:04:05:06
ap5_interface_mac: 01:02:03:04:05:07
wired_lan_interface_mac: 01:02:03:04:05:08
wan_interface_mac: 01:02:03:04:05:09

# Channel selection
ap24_channel: 6
ap5_channel: 36

# IP assignment
lan_bridge_ip: 10.0.1.1/24
lan_bridge_dhcp_range_start: 10.0.1.100
lan_bridge_dhcp_range_end:   10.0.1.199
# Add Google DNS server to the resolver configuration
lan_bridge_extra_nameservers:
  - 8.8.8.8

# PPP authentication
wan_chap_username: 'my_username'
wan_chap_password: 'my_password'
```

## Configuration

### Global variables

The following variables control global execution:

* `run_handlers`: will force any Ansible handler to run again. This is
  mainly useful in case an error occurs and the playbook must be
  re-executed. It ensures that handlers are not skipped because their
  triggering tasks have already been executed;
* `keep_going`: does not stop on errors. Useful for debugging.

A set of variables enable or disable optional features:

* `enable_rtl8812u`: if set to `true`, the playbook will install the
  out-of-tree driver for the Realtek RTL8812BU wireless chipset via
  DMKS;
* `enable_ap5`: if set to `true`, support for a single 5GHz AP is
  configured;
* `enable_ap24`: if set to `true`, support for a single 2.4GHz AP is
  configured;
* `enable_wired_lan`: if set to `true`, support for wired LAN
  connectivity is configured;
* `enable_powersave`: if set to `true`, power-saving configurations
  will be performed.

By default, _all_ of the feature variables above are enabled.

## Feature-specific configuration

### Realtek RTL8812BU driver support

Enabled or disabled via `enable_rtl8812u`. Performed by role
`rtl8812bu`.

Generally speaking, the playbook is agnostic to the devices used to
create wireless AP's or wired LANs; it simply expects the kernel to
support them. If this is not the case, drivers/firmware must be
installed before running the playbook.

However, `routerify` was written to automate a specific deployment
scenario, which involves a well-defined network adapter employing the
Realtek RTL8812BU chipset. Its Linux driver is not in-tree, so it must
be downloaded, added do DKMS, and compiled for each available kernel.

As a convenience, a role to deploy this specific driver was included, so
if you happen to have an adapter employing the same chip this will save
you the effort of installing it manually. It also includes some nice
module option tweaks (e.g., enabling 80MHz channels in 802.11ac mode).

Disable this feature if you are using any other WiFi chip.

Fine-tuning of this feature can be obtained via the variables define in
`roles/rtl8812bu/defaults/main.yaml`. In particular, they define the URL
used to download the driver, and the expected naming of the extracted
folder.

### 2.4GHz AP

Enabled or disabled via `enable_ap24`. Performed by role `ap24`.

If enabled, it will configure an instance of `hostapd` to start at boot
and to use a WiFi card supporting AP mode as a 2.4GHz AP.

The following variables can be used to configure the AP:

* `ap24_interface_name`: defines how the interface used for the 2.4GHz
  AP will be renamed;
* `ap24_interface_mac`: the AP interface is selected by its MAC address
  in colon format (e.g., `00:11:22:33:44:55`);
* `ap24_channel`: WiFi channel for the AP; default value is `6`;
* `ap24_country`: country code used to set the wireless regulatory
  domain for the AP. If not explicitly overridden, it will use the value
  of `ap_common_country`;
* `ap24_ssid`: SSID of the 2.4GHz network. If not overridden, it will
  use the value of `ap_common_ssid`, with `-24` appended;
* `ap24_password`: ASCII WPA/WPA2 passphrase used for authentication. If
  not overridden, it will use the value of `ap_common_wifi_password`.

For each new setup, the only variable which requires tweaking is
`ap24_interface_mac`, as every adapter will have its own MAC. Changing
the channel may or may not be useful in your environment.

Country code, SSID, and WPA password can be either set to unique values
or shared with the 5GHz AP. In that case, you should override variables
from the `ap_common` role instead.

The template file `hostapd.conf.j2` may need to be customized if you
need to change low-level device settings.

### 5GHz AP

Enabled or disabled via `enable_ap5`. Performed by role `ap5`.

If enabled, it will configure an instance of `hostapd` to start at boot
and to use a WiFi card supporting AP mode as a 5GHz AP.

The following variables can be used to configure the AP:

* `ap5_interface_name`: defines how the interface used for the 5GHz AP
  will be renamed;
* `ap5_interface_mac`: the AP interface is selected by its MAC address
  in colon format (e.g., `00:11:22:33:44:55`);
* `ap5_channel`: WiFi channel for the AP, default value is `36`;
* `ap5_country`: country code used to set the wireless regulatory domain
  for the AP. If not explicitly overridden, it will use the value of
  `ap_common_country`;
* `ap5_country3`: extra country code byte that controls indoor/outdoor
  usage of the AP. The default is `0x49`, for indoor-only use;
* `ap5_ssid`: SSID of the 5GHz network. If not overridden, it will use
  the value of `ap_common_ssid`;
* `ap5_password`: ASCII WPA/WPA2 passphrase used for authentication. If
  not overridden, it will use the value of `ap_common_wifi_password`.

For each new setup, the only variable which requires tweaking is
`ap5_interface_mac`, as every adapter will have its own MAC. Changing
the channel may or may not be useful in your environment.

Country code, SSID and WPA password can be either set to unique values
or shared with the 2.4GHz AP. In that case, you should override
variables from the `ap_common` role instead.

The template file `hostapd.conf.j2` may need to be customized if you
need to change low-level device settings.

### Common AP settings

Performed by role `ap_common`. This role acts as a dependency, and
cannot be enabled or disabled on its own. It is called by other AP
roles.

```yaml
# Base SSID used for both 2.4GHz and 5GHz if not overridden
ap_common_ssid: MYSSID
# Global country code
ap_common_country: IT
# Default password shared between bands
ap_common_wifi_password: abcd$123
```

It defines the SSID, country code and password variables, which are
inherited by `ap24` and `ap5`, unless overridden. It is useful to set
the same configuration for both bands at once.

### Wired LAN

Enabled or disabled via `enable_wired_lan`. Performed by role
`wired_lan`.

If enabled, it will configure a single wired interface to join the LAN.
A single device can be attached to this interface directly, or a
physical L2 switch may be used to attach multiple devices. Every host
can see all other wired/wireless nodes and gets its address from the
local DHCP server.

The following variables can be used:

* `wired_lan_interface_name`: this is the final name the chosen LAN
  interface will be renamed to;
* `wired_lan_interface_mac`: the interface is selected by its MAC
  address in colon format (e.g., `00:11:22:33:44:55`).

For each new setup, the only variable which requires tweaking is
`wired_lan_interface_mac`, as every adapter will have its own MAC.

### LAN bridge

Performed by role `lan-bridge`. This role acts as a dependency, and
cannot be enabled or disabled on its own.

It creates a software switch, to which the wired LAN interface, the
2.4GHz AP, and the 5GHz AP will connect to allow for hosts to reach one
another at L2 level.

The following variables can be used:

* `lan_bridge_interface_name`: the name of the bridge. Must be a valid
  Linux interface name;
* `lan_bridge_ip`: IPv4 assigned to the bridge interface, used to
  address the router itself. Must be given in CIDR notation
  (e.g.,`10.0.1.1/24`);
* `lan_bridge_extra_nameservers`: a list of additional name servers that
  `dnsmasq` will use to resolve domain names;
* `lan_bridge_dhcp_range_start`: this is the first address of the DHCP
  pool, inclusive (e.g., `10.0.1.100`);
* `lan_bridge_dhcp_range_end:`: this is the last address of the DHCP
  pool, inclusive (e.g., `10.0.1.199`);
* `lan_bridge_dhcp_reservations`: this is the list of MAC reservations.
  Each reservation is an object with a `mac` and an `ip` fields:

  ```yaml
  lan_bridge_dhcp_reservations:
    - mac: "11:22:33:44:55:66"
      ip: "10.0.1.10"
  ```

Currently, it is not possible to disable DHCP; however, hosts can still
be configured to use static addresses, and the DHCP range can be very
small.

`lan_bridge_ip` must lay outside the DHCP range. Currently, this is not
checked nor is checked that they are on the same subnet. 

### Powersave

Enabled or disabled via `enable_powersave`. Performed by role
`powersave`.

It configures energy-related settings to ensure that:

* sleep, suspend, and hibernation are disabled;
* the screen shuts down after a timeout while displaying the console.

These settings are mainly useful when a laptop is used as a router. The
first bullet ensures that you cannot put the system to sleep by mistake
if the lid is closed. The second bullet cuts screen power so that
it does not remain on forever.

The following variables can be used:

* `powersave_tty_poweroff_minutes`: controls the inactivity timeout in
  minutes; the screen will power off after this set period of time.

### WAN

Performed by role `wan`. This role acts as a dependency, and
cannot be enabled or disabled on its own.

It performs all the required configuration steps to allow a PPP
connection to be established between the system and an upstream PPPoE
server, via which Internet access is obtained. It involves settings up
PPPoE, VLAN tagging, and identifying the physical NIC connected to the
ONT.

Currently, this is the only supported scheme for connecting to the
Internet. PPPoE must be authenticated with CHAP.

PPPoE connections react to carrier changes of the physical NIC.  When
the carrier goes away, the connection is terminated. When the carrier is
detected, it is brought up.

The following variables can be used:

* `wan_interface_name`: the final name of the physical NIC connected to
  the ONT;
* `wan_interface_mac`: MAC of the physical NIC, in `:`-separated form;
* `wan_chap_username`: CHAP username;
* `wan_chap_password`: CHAP password;
* `wan_vlan_id`: ID of the VLAN on which PPPoE traffic is sent. This is
  usually an ISP-provided value;
* `wan_vlan_interface_name`: final name of the VLAN interface;
* `wan_ppp_interface_name`: final name of the PPP interface. _Currently
  renaming this interface does not work, so please do not override
  this variable_.

## Routing

After configuration, the system will route packets as most residential
routers would do:

* Outgoing traffic is allowed from both the router itself and from any
  LAN host, and is subject to masquerading;
* Inbound traffic that is related to already seen outgoing traffic is
  allowed;
* Establishing new connections from the WAN is not allowed (e.g., all
  ports are closed).

Before provisioning, the following variables can be used to define port
forwarding to specific services:

* `routing_input_rules` defines ports on the router itself that will be
  accessible from the Internet (e.g., to allow SSH connectivity from
  outside the network);
* `routing_portfwd_rules` defines port forwarding rules for services
  that are offered by hosts behind the gateway itself. You should state
  both the port and the target host address;
* `routing_other_rules` defines ports on the router itself that will be
  accessible from any interface which is not the PPP interface (this
  means that such ports are open for additional, non PPPoE traffic
  coming from the WAN interface). This is mainly useful if you need to
  connect to SSH from a local interface which is not part of the LAN.

Have a look at their definitions for the exact layout.

Port forwarding can be enabled by adding new rules/items to dedicated
chains/sets, depending on the location of the target service:

* For services that run on the router itself, simply add the port number
  you want to open to either `TCPLOCALPORTS` or `UDPLOCALPORTS` sets (if
  you want them to be accessible from the Internet). This is equivalent
  to a setup-time rule added to `routing_input_rules`.  Using
  `TCPOTHERPORTS` or `UDPOTHERPORTS` instead is equivalent to using
  `routing_other_rules`. For example:

  ```sh
  # For a TCP service
  ipset add TCPLOCALPORTS 5000
  # For a UDP service
  ipset add UDPLOCALPORTS 5000

  # Make the change persistent
  ipset save > /etc/iptables/ipsets
  ```
* For services running on a LAN node, define the appropriate DNAT rule,
  adding it to the `PORTFWD` chain in the NAT table:

  ```sh
  # DNAT connections for local port 2222/TCP to 192.168.1.100:4444
  iptables -t nat -A PORTFWD -p tcp --dport 2222 \
    -j DNAT --to-destination 192.168.1.100:4444

  # Make the change persistent
  iptables-save > /etc/iptables/rules.v4
  ```

## Vagrant debug box

The `vagrant` subfolder contains support files to provision a debug
machine that can be used for quick and dirty testing of the playbook. It
is designed to use VirtualBox and USB passthrough to drive real WiFi
adapters in AP mode, while the WAN and LAN interfaces are faked via
host-only networks.

At present, the `Vagrantfile` is incomplete. It will try to load ad
additional file called `ap_spec.rb` which shall provide information
about the AP devices to use:

```ruby
AP24_MAC = "11:22:33:44:55:66"
AP24_VID = "1111"
AP24_PID = "2222"
AP5_MAC  = "11:22:33:44:55:67"
AP5_VID  = "3333"
AP5_PID  = "4444"
AP24_PROVISIONER = "./ap24.yaml"
AP5_PROVISIONER = ""
```

For both the 2.4GHz and 5GHz APs, you should provide their MAC
addresses, as well as their USB vendor and product ID (which are needed
to automatically setup VirtualBox passthrough filters). Leaving a VID
blank will disable the corresponding AP from the playbook execution.

By defining `*_PROVISIONER` to point to a file, it will automatically be
invoked as an Ansible playbook prior to running `routerify`. This is
optional and useful if your AP device needs additional setup to work
(i.e. firmware files).

<!-- Links and resources -->
[network-diagram]: ./docs/architecture.png

<!-- vi: set tw=72 et sw=2 fo=tcroqan autoindent: -->
