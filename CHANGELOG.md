# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

For versions that have a major of 0, a convention is followed so that
the minor number is incremented when backward-incompatible changes are
made, while the third number is incremented for backward compatible
changes. For example, versions `0.2.x` are not compatible with `0.1.x`.

## [Unreleased]

## [0.3.5] - 2026-02-04

### Added

* DNS interception: the router can intercept all DNS packets _over UDP_
  coming from the LAN and redirect them to a target (such as a PiHole
  instance) that can do any extra processing before returning an answer.
  This is designed to allow easy integration with a DNS sink.

### Fixed

* Fix an issue that caused `netplan` configuration to be applied too
  early during provisioning, even when `reconfigure_now` was false. This
  could cause connectivity loss as the new settings could interfere with
  the current addressing plan.
* Bind `hostapd` services for APs to the physical inetrfaces they
  manage, ensuring they do not try to start and fail if the
  corresponding radio device is not present.

## [0.3.4] - 2026-01-11

### Fixed

* Install `linux-headers-$(dpkg --print-architecture)` to ensure that
  out-of-tree drivers can be rebuilt when the kernel is upgraded.

## [0.3.3] - 2026-01-03

### Fixed

* Blacklist the conflicting module `rtw88_8822bu` when installing the
  `88x2bu` driver.

## [0.3.2] - 2025-12-28

### Added

* Support Debian 13.
* The new `pppoe-server` playbook can be used to provision a
  Debian-based PPPoE server to test uplink connectivity for the main
  playbook. Only tested on Debian 13.

### Fixed

* PPP peer files should belong to the `dip` group.

## [0.3.1] - 2023-09-24

### Fixed

* Implement renaming the PPP interface using `wan_ppp_interface_name`.

## [0.3.0] - 2023-08-12

### Added

* Tested against Debian 10, 11 and 12.
* Reconfigure a router via the playbook on the fly.
* Handle daily unattended upgrades.
* LAN hosts can be given DNS aliases (ex. `print.lan`).
* Control the bus speed (High/Super) of the RTL8812BU.
* TKIP can be disabled for increased security.
* Block Internet access for specific clients by MAC address.
* New role to install drivers for the RTL8821CE WiFi NIC.

### Changed

* PPPoE is handled using kernel-mode support.
* Allow for more than one wired LAN NIC.

### Fixed

* Ensure that kernel sources for the currently running kernel are always installed.
  Otherwise DKMS driver installation may fail.
* Do not update APT lists when reconfiguring, as we may not have Internet connectivity.

## [0.2.0] - 2020-12-18

### Fixed

* `iptables` rules have been reworked to better handle common
  patterns and avoid duplication. Most checks, which were only
  applied to input traffic, are now applied to forwarded traffic
  as well.
* The documentation has been reviewed for typos and stylistic issues.
  Contributed by [dgabriel123](https://github.com/dgabriel123).
* Ansible variable `enable_rtl8812u` has been renamed to `enable_rtl8812bu`,
  to reflect the full, correct chip name.
* Documented some AP-related variables that were left behind.
* It is now possible to disable VHT when using the RTL8812BU.
* Fixed disabling APs via `ap_spec.rb`.

### Changed

* All arguments passed to role invocations are now passed as parameters
  rather than as variables. This way, each parameter applies to that
  specific invocation only.

## [0.1.0] - 2020-12-13

### Added

* Initial release of the playbook.

<!-- vi: set tw=72 et sw=2 fo=tcroqan autoindent: -->
