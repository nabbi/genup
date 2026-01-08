# genup

A tool to update the **Portage**(5) tree, all installed packages, and the kernel on Gentoo Linux.

---

## Background and Fork History

This project is a maintained fork of
[sakaki-/genup](https://github.com/sakaki-/genup).

The original upstream project received an
[EOL notice](https://forums.gentoo.org/viewtopic-p-8522963.html#8522963), and over time no longer aligned with my operational needs. While evaluating alternatives, I also reviewed
[phosphorcube/gentoo-update](https://github.com/phosphorcube/gentoo-update), another rewrite fork that is now unmaintained. Some ideas and approaches from that project were incorporated as well.

This fork exists to **preserve the spirit and workflow of the original genup**, while extending it with additional features required for modern Gentoo deployments.

---

## Notable Changes from Upstream

Compared to the original sakaki-/genup, this fork adds or improves support for:

* Email notifications via `sendmail`

  * Relies on a system mailer (for example `nullmailer`) being configured
* Support for `@live-rebuild`
* Detection of read-only Portage trees and overlays

  * Useful for NFS-mounted Portage or overlay setups
* Legacy `layman` overlay support

  * For systems not yet migrated to newer overlay tooling
* Automatic `git pull` of `/etc/portage/patches`

  * For users maintaining local patch repositories
* Automatic mounting and unmounting of `/boot`
* Optional use of binary packages via `usepkg`

  * Useful for NFS-mounted bindirs
* Kernel build support via `genkernel`

  * Used if installed and `buildkernel` is not available

The script has been deliberately decoupled from a hard dependency on the ebuilds that configure `emtee` and `buildkernel`. If these optional components are present on the system, genup will automatically detect and use them.

---

## Description

**genup** is a utility intended to simplify the process of keeping a Gentoo system fully up to date. When invoked, it performs the following steps in order:

* Updates the Portage tree and active overlays, and syncs **eix**(1)

  * Using `emaint sync` and `eix-sync`
* Removes any prior **emerge**(1) resume state

  * Using `emaint --fix cleanresume`
* On `aarch64`, attempts to apply pending fixups

  * By running `/etc/cron.weekly/fixup` if present (errors are non-fatal)
* Ensures **Portage**(5) itself is up to date

  * Using `emerge --oneshot --update portage`
* Ensures **genup** itself is up to date

  * Using `emerge --oneshot --update genup` (restarting if updated)
* Updates all packages in the `@world` set

  * First using **emtee**(1), if enabled via USE flags
  * Then using `emerge --deep --with-bdeps=y --changed-use --update @world`
* Removes unreferenced packages

  * Using `emerge --depclean`
* Rebuilds external kernel modules (for example VirtualBox)

  * Using `emerge @module-rebuild --exclude '*-bin'`
* Rebuilds packages linked against stale libraries

  * Using `emerge @preserved-rebuild`
* Updates outdated **perl**(1) modules

  * Using `perl-cleaner --all`
* Resolves configuration file updates in interactive mode

  * Using `dispatch-conf`
* Builds a new kernel into staging, if supported

  * Using `buildkernel --stage-only`
* Removes unreferenced packages (again)

  * Using `emerge --depclean`
* Fixes missing shared library dependencies

  * Using `revdep-rebuild`
* Rebuilds preserved libraries (again)

  * Using `emerge @preserved-rebuild`
* Optionally removes unused distfiles

  * Using `eclean --deep distfiles`
* Deploys a staged kernel, if available and requested

  * Using `buildkernel --copy-from-staging`
* Updates environment settings as a precaution

  * Using `env-update`
* Updates **eix** package metadata

  * Using `eix-sync -0`
* Runs any custom updater scripts found in `/etc/genup/updaters.d`

The utility can be run in non-interactive mode (the default) or interactive mode using the **--ask** option. Non-interactive mode is suitable for scripted execution, such as nightly **cron**(8) jobs.

---

## Installation

On Gentoo systems, **genup** is best installed via its ebuild. A maintained fork of the original sakaki- ebuild is available in the
[oubliette-overlay](https://github.com/nabbi/oubliette-overlay).

Additional background and installation context can be found in
[**Sakaki's EFI Install Guide**](https://wiki.gentoo.org/wiki/Sakaki's_EFI_Install_Guide) on the Gentoo Wiki.
