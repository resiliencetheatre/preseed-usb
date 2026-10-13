# Debian installation DVD with preseed

`build-preseed-dvd.sh` builds a bootable Debian 13 (trixie) amd64 installer
image with an embedded `/preseed.cfg`. It keeps the original DVD package tree
and replays its boot configuration, adding automatic installation for GRUB
(UEFI) and ISOLINUX (BIOS). The installer reads `file=/cdrom/preseed.cfg`;
no server is needed to serve the configuration.

Use the **DVD-1 installer ISO** for installations with limited or no network
access: it carries packages locally. The script requires an input filename
ending in `-DVD-1.iso`; use the Debian Installer image, not a live image.

1. Install the build dependencies on a Debian build machine:

   ```sh
   sudo apt install python3 xorriso
   ```

   Place the original DVD ISO and template in this directory. Allow space for
   another DVD-sized ISO. Once these inputs and tools are available, building
   the image does not require network access.

2. Review `preseed.cfg`. It selects a Finnish keyboard, English locale,
   Luxembourg timezone, GNOME desktop, SSH server and extra packages. It uses
   encrypted LVM, creates user `tech` with sudo access, disables root password
   login, and powers off after installation. `preseed-ax2.cfg` and
   `preseed-dell.cfg` are existing machine-specific configurations.

   **For offline installation**, copy the template to `offline.cfg` and replace
   the corresponding settings with:

   ```text
   d-i netcfg/enable boolean false
   d-i clock-setup/ntp boolean false
   d-i apt-setup/use_mirror boolean false
   d-i apt-setup/services-select multiselect
   d-i pkgsel/upgrade select none
   ```

   Add `netcfg/enable` if absent. The supplied configurations enable mirrors,
   security/updates services, NTP and upgrades; the script preserves those
   settings unless you edit the template. Review `tasksel/first` and
   `pkgsel/include`: all selected tasks and packages must be available on
   DVD-1 for a fully offline install. Test with networking disconnected in a
   disposable VM; the full supplied package list is not verified for offline
   availability. The template disables CD-ROM APT entries in the installed
   system; this is separate from using the DVD during installation.

3. Run the builder as your normal user, without `sudo`:

   ```text
   ./build-preseed-dvd.sh TECH_PASSWORD DRIVE LVM_PASSPHRASE HOSTNAME NAME [DVD_ISO] [TEMPLATE]
   ```

   Example (replace both placeholder passwords and confirm the target disk):

   ```sh
   ./build-preseed-dvd.sh 'replace-user-password' /dev/nvme0n1 \
     'replace-disk-passphrase' workstation offline \
     debian-13.7.0-amd64-DVD-1.iso offline.cfg
   ```

   `DRIVE` is the whole disk on the **installation target**, not the build
   machine or installation USB. `HOSTNAME` names the installed machine;
   `NAME` labels the output files. The optional arguments default to
   `debian-13.7.0-amd64-DVD-1.iso` and `preseed.cfg`.

   This example creates `preseed-offline.cfg` and
   `debian-13.7.0-amd64-DVD-offline.iso` in the current directory. The script
   sets the user password, target disk, encryption passphrase and hostname,
   updates checksums, and verifies the embedded configuration. Existing output
   files are never overwritten; choose a new `NAME` for another build.

4. Burn the resulting ISO as a DVD image, write it to a USB drive with an image
   writer, or attach it as a VM's virtual DVD. Boot the target from that media.
   The builder itself only creates files; it does not write installation media.

**Booting the image starts automatic installation after five seconds and
replaces all partitions on the selected disk.** Check the target disk name
before building. After installation powers off, remove the media before
starting the machine again, then enter the encryption passphrase to boot.

The generated configuration and ISO contain plaintext passwords. Keep them
private; passwords passed to the script can also appear in shell history and
process arguments. Output files are created with owner-only permissions.
