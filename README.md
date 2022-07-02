<!--
SPDX-FileCopyrightText: 2022 Håvard Moen <post@haavard.name>

SPDX-License-Identifier: GPL-3.0-or-later
-->

# gnome-keyring-unlock

Script to unlock gnome keyring using password from stdin. This can be used for example to unlock gnome-keyring when using fingerprint to login.

## Usage

### Read password and unlock

```bash
read password
./unlock.py <<<$password
```

### Decrypt password using tpm chip

First you need to set up the encrypted password file. You will need to install [clevis](https://github.com/latchset/clevis).
I'm using [doas](https://github.com/slicer69/doas), but you can replace with sudo.

The required configurion for doas is (replace `USERNAME` with your user):
```
permit nopass USERNAME as tss cmd /usr/bin/clevis-encrypt-tpm2
permit nopass USERNAME as tss cmd /usr/bin/clevis-decrypt-tpm2
```

To setup the encrypted password file, run:

```bash
read password
doas -u tss /usr/bin/clevis-encrypt-tpm2 '{"pcr_ids":"7"}' <<<$password > ~/.config/gnome-keyring.tpm2
```

Then to unlock you can run:
```bash
doas -u tss /usr/bin/clevis-decrypt-tpm2 < .config/gnome-keyring.tpm2 | ./unlock.py
```

### Setting up automatic unlock during login

If you are using fingerprint and/or fido2 to log in instead of password,
gnome keyring will not be unlocked.
Copy `unlock.py` to `~/bin` and put the following in `~/.bash_profile`
if using bash or `~/.zprofile` if using zsh:

```bash
if [ -f ~/.config/gnome-keyring.tpm2 ]
then
    if ! [ -S /run/user/$UID/keyring/control ]
    then
      gnome-keyring-daemon --start --components=secrets
    fi
    doas -u tss /usr/bin/clevis-decrypt-tpm2 < .config/gnome-keyring.tpm2 | ~/bin/unlock.py
fi
```
