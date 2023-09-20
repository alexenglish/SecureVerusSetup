# Setting up ssh unlock

## Before entering any commands, please read the entire text. There are some caveats with respect to DHCP/non-DHCP settings and SSH keys support, which are mentioned near the end.
## It is strongly advised to install this while you have console access to the system. Either through your VMs graphical manager, or through command line console access. If the configuration is not correct, you will not be able to unlock your drives without it.

log in as root on the Linux system and do:

```shell
apt update && apt upgrade
```
(answer `yes` if asked, to install updated packages)
Install the required binaries, to allow unlocking through an ssh session:
```shell
apt-get install -yy dropbear-initramfs cryptsetup-initramfs lvm2
```
Configure the dropbear configuration:
```shell
echo 'DROPBEAR_OPTIONS="-RFEsjk -c /bin/cryptroot-unlock -p 2222"' > /etc/dropbear-initramfs/config
```
(change `2222` to your desired port. It is advisable to use a non-standard port for security and avoiding *Man-In-The-Middle* warnings because the server keys are different in Dropbear than the default OpenSSL)
Now insert your (and only your) SSH key (obviously edit this command before pressing enter):
```shell
echo '<YOUR_PUBLIC_KEY>' > /etc/dropbear-initramfs/authorized_keys
```
Now update your initram and grub:
```shell
update-initramfs -k all -u
update-grub
```

Now reboot your system and test.


## Caveat 1: non-DNS IP (aka static IP)
The default kernel’s behavior is getting the IP address via dhcp (ip=dhcp). If your network lacks a DHCP server, special kernel boot IP parameter is needed. This would usually be in the form of:
ip=<client-ip>::<gw-ip>:<netmask>5

Append that to the GRUB_CMDLINE_LINUX_DEFAULT parameter in `/etc/default/grub` and regenerate GRUB config file:
`update-grub`

## Caveat 2: SSH keys
Older operating systems may not support ed25519 keys. You will need to use an inferior encryption, like rsa.
It is advisable to use different SSH keys for unlocking and general system access. It simply makes it harder to gain access if somebody manages to get their hands on your hardware.

## Caveat 3: Location, location, location
The latest version of Debian (such as 12+), Ubuntu (22.04 LTS+), Linux Mint, and Pop!_OS uses the following new version config files and directories:

New Directory: `/etc/dropbear/initramfs/`
New config file: `/etc/dropbear/initramfs/dropbear.conf`
New files containing public keys for public key authentication: `/etc/dropbear/initramfs/authorized_keys`
