# sudo with a FIDO2 key (pam_u2f) on RHEL 10

With the key plugged in, `sudo` is authorized by touching it, no password needed. Without the key, it asks for the usual password.

## Requirements

- RHEL 10 and a FIDO2 USB security key
- A user with sudo rights
- A root terminal kept open for the whole process (`sudo -i`) as a safety net

## 1. Install packages

```bash
sudo dnf install pam-u2f pamu2fcfg fido2-tools
```

If they are not found, enable EPEL.

## 2. Check that the key is detected

```bash
fido2-token -L
sudo fido2-token -I /dev/hidrawX
```

`FIDO_2_0` should appear under `version strings`.

## 3. Register the key

As a regular user, without sudo:

```bash
mkdir -p ~/.config/Yubico
pamu2fcfg > ~/.config/Yubico/u2f_keys
```

Touch the key when prompted. Then check:

```bash
cat ~/.config/Yubico/u2f_keys
```

There should be a single line `user:<long string>`.

## 4. Configure PAM for sudo only

Back up and edit:

```bash
sudo cp /etc/pam.d/sudo /etc/pam.d/sudo.bak
sudo vi /etc/pam.d/sudo
```

Add this line before any other `auth` line:

```
auth       sufficient   pam_u2f.so cue
```

Result:

```
#%PAM-1.0
auth       sufficient   pam_u2f.so cue
auth       include      system-auth
account    include      system-auth
password   include      system-auth
session    include      system-auth
```

- `sufficient`: if the key validates, authentication succeeds without a password; otherwise it falls through to `system-auth`.
- `cue`: prints a prompt telling you to touch the key.

## 5. Test (in a new terminal)

With the key:

```bash
sudo -k
sudo whoami   # touch the key -> root
```

Without the key (unplugged):

```bash
sudo -k
sudo whoami   # asks for the password
```

## Security notes

- Without a PIN, anyone holding the key with access to your session can run sudo just by touching it. The touch protects against remote processes and malware, not against physical access.
- To require a PIN: `fido2-token -S /dev/hidrawX`, register with `pamu2fcfg -N`, and add `pinverification=1` to the PAM line.
- To require the key **and** the password: change `sufficient` to `required`. If the key is lost, recover with `rd.break` (see below).

## Troubleshooting

- **SELinux blocks reading `u2f_keys`**: use a central file, e.g. `authfile=/etc/u2f_mappings` on the PAM line.
- **Permission denied accessing the key**: check permissions on `/dev/hidraw*` and udev rules.
- **Stops working after an update**: check that the line is still present in `/etc/pam.d/sudo`.

## Revert

```bash
sudo cp /etc/pam.d/sudo.bak /etc/pam.d/sudo
```

Or just delete the `pam_u2f` line.

## Lost key

- With `sufficient`: use your normal password and delete `~/.config/Yubico/u2f_keys`.
- With `required`, or if sudo is broken: in GRUB press `e`, append `rd.break` to the `linux` line, press `Ctrl+X`, then:

```bash
mount -o remount,rw /sysroot
chroot /sysroot
# edit /etc/pam.d/sudo or run passwd root
touch /.autorelabel
exit
exit
```

This does not work if the disk is LUKS-encrypted and you lack the passphrase, or if GRUB is password-protected.

## Second key (recommended)

```bash
pamu2fcfg -n >> ~/.config/Yubico/u2f_keys
```

> Do not publish the contents of `u2f_keys`, hostnames, or credentials in the wiki.
