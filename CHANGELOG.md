# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

For versions that have a major of 0, a convention is followed so that
the minor number is incremented when backward-incompatible changes are
made, while the third number is incremented for backward compatible
changes. For example, versions `0.2.x` are not compatible with `0.1.x`.

## [Unreleased]

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
