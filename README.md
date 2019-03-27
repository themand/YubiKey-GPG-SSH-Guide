# YubiKey GPG (with SSH) Guide

This is a guide on how to setup YubiKey for GPG and SSH. It's based on [drduh/YubiKey-Guide](https://github.com/drduh/YubiKey-Guide) with some tiny changes:

* It's much less descriptive - just a series of steps/commands to execute. It assumes you've done it before and/or understand what you are doing. Use it if you want to follow a recipe for a quick execution. If it's your first time, read the [original guide](https://github.com/drduh/YubiKey-Guide) to understand the process.
* It configures YubiKey to require touch on every GPG action (additionally, not instead of PIN) using [a-dma/yubitouch](https://github.com/a-dma/yubitouch) script
* It includes instructions to split backup copy of the keys with Shamir's Secret Sharing Scheme, using [themand/shamir-sharing](https://github.com/themand/shamir-sharing)

## Table of Contents

* [Table of Contents](#table-of-contents)
* [Requirements](#requirements)
* [Verify YubiKey](#verify-yubikey)
* [Reset and erase YubiKey GPG](#reset-and-erase-yubikey-gpg)
* [Create keys and setup YubiKey](#create-keys-and-setup-yubikey)
    + [Download required software and config files](#download-required-software-and-config-files)
    + [Go offline](#go-offline)
    + [Prepare temporary directories](#prepare-temporary-directories)
    + [Create a master key](#create-a-master-key)
    + [Create subkeys](#create-subkeys)
    + [Verify keys](#verify-keys)
    + [Export keys](#export-keys)
        - [Export private keys](#export-private-keys)
        - [Export public keys](#export-public-keys)
        - [Archive GPG homedir](#archive-gpg-homedir)
        - [Split exported secrets using shamir-split](#split-exported-secrets-using-shamir-split)
    + [Backup exported keys into a secure location](#backup-exported-keys-into-a-secure-location)
    + [Configure YubiKey GPG Smartcard](#configure-yubikey-gpg-smartcard)
        - [Require touch for GPG operations](#require-touch-for-gpg-operations)
        - [Change PIN and Admin PIN](#change-pin-and-admin-pin)
    + [Transfer keys to YubiKey](#transfer-keys-to-yubikey)
    + [Verify card keys](#verify-card-keys)
    + [Cleanup](#cleanup)
* [Using keys](#using-keys)

## Requirements

* The guide assumes you run [Tails Linux](https://tails.boum.org/). It will work on other systems (if all software requirements from the original guide are installed), but I use and recommend using Tails for this task.
* The guide assumes your internet connection is via TOR (as it's using Tails). If not, remove `--socks5-host` argument from `curl` options.
* You need to have a working networking to download software. It's recommended to disable networking after downloading software and before generating any keys. If using USB WiFi dongle, you can simply unplug it to make sure you're offline.  

## Verify YubiKey

Use Yubico's [Verify YubiKey](https://www.yubico.com/genuine/) to confirm that you have a genuine YubiKey.

## Reset and erase YubiKey GPG

If for some reason (e.g. locked PIN) you'd like to erase YubiKey GPG application and restore it to a factory state, [follow this instruction](https://support.yubico.com/support/solutions/articles/15000006421-resetting-the-openpgp-applet-on-the-yubikey). Warning: this will delete your private keys stored on the YubiKey. 

## Create keys and setup YubiKey

### Download required software and config files

Download [themand/shamir-sharing](https://github.com/themand/shamir-sharing) binaries

```bash
export BINDIR=$(mktemp -d) && \
mkdir -p "$BINDIR" && \
curl --socks5-host localhost:9050 -Lo "$BINDIR"/shamir-split https://github.com/themand/shamir-sharing/releases/download/v1.0.0/linux-shamir-split && \
curl --socks5-host localhost:9050 -Lo "$BINDIR"/shamir-combine https://github.com/themand/shamir-sharing/releases/download/v1.0.0/linux-shamir-combine  && \
curl --socks5-host localhost:9050 -Lo "$BINDIR"/macos-shamir-split https://github.com/themand/shamir-sharing/releases/download/v1.0.0/macos-shamir-split  && \
curl --socks5-host localhost:9050 -Lo "$BINDIR"/macos-shamir-combine https://github.com/themand/shamir-sharing/releases/download/v1.0.0/macos-shamir-combine  && \
curl --socks5-host localhost:9050 -Lo "$BINDIR"/win-shamir-split.exe https://github.com/themand/shamir-sharing/releases/download/v1.0.0/win-shamir-split.exe  && \
curl --socks5-host localhost:9050 -Lo "$BINDIR"/win-shamir-combine.exe https://github.com/themand/shamir-sharing/releases/download/v1.0.0/win-shamir-combine.exe
```

Verify checksums

```bash
shasum -a 256 "$BINDIR"/shamir-split | grep 78cc9330c0431399798769a3a01d92eb963b8ba5a1374ecb5e81b0153bfa0100 && chmod +x "$BINDIR"/shamir-split && echo "===== OK" || echo "===== linux-shamir-split checksum DOES NOT MATCH. ABORT! ====="; \
shasum -a 256 "$BINDIR"/shamir-combine | grep 1c62344049f5bca1c6aeeaa2dbaed8ba2f888bf32950797b7f87cf100a5b76fa && chmod +x "$BINDIR"/shamir-combine && echo "===== OK" || echo "===== linux-shamir-combine checksum DOES NOT MATCH. ABORT! ====="; \
shasum -a 256 "$BINDIR"/macos-shamir-split | grep 14e888964e176040047173debbd60b272a028759e19cd59218671ef7c2c85b24 && chmod +x "$BINDIR"/macos-shamir-split && echo "===== OK" || echo "===== macos-shamir-split checksum DOES NOT MATCH. ABORT! ====="; \
shasum -a 256 "$BINDIR"/macos-shamir-combine | grep bf84bd196099b3733d36c30b27940d7da255f7c81487bbed7c67b497135dc05c && chmod +x "$BINDIR"/macos-shamir-combine && echo "===== OK" || echo "===== macos-shamir-combine checksum DOES NOT MATCH. ABORT! ====="; \
shasum -a 256 "$BINDIR"/win-shamir-split.exe | grep 8e3a4c0dffcaf14d772ae1ca341bdf8df6ed0adc694a9295f412ca24ebc048d0 && echo "===== OK" || echo "===== win-shamir-split.exe checksum DOES NOT MATCH. ABORT! ====="; \
shasum -a 256 "$BINDIR"/win-shamir-combine.exe | grep 22a1db4b7969bc480ce9d056ed01fc5d4a8c55cff025c8220e4e0aa10d5c2a62 && echo "===== OK" || echo "===== win-shamir-combine.exe checksum DOES NOT MATCH. ABORT! ====="
```

Download [themand/yubitouch](https://github.com/themand/yubitouch)

```bash
curl --socks5-host localhost:9050 -Lo "$BINDIR"/yubitouch https://raw.githubusercontent.com/themand/yubitouch/master/yubitouch.sh
```

Verify checksum

```bash
shasum -a 256 "$BINDIR"/yubitouch | grep 20d0b3aee42948cd3d7aaf9ef9ebdedbb91edbc4a38bc4a65d8e10d064194adc && chmod +x "$BINDIR"/yubitouch && echo OK || echo "===== yubitouch checksum DOES NOT MATCH. ABORT! ====="
```

Download [themand/macos-bootstrap/dotfiles/gpg.conf](https://raw.githubusercontent.com/themand/macos-bootstrap/master/bootstrap/assets/dotfiles/.gnupg/gpg.conf) with reasonable and secure GPG defaults.

```bash
export GPGCONF=$(mktemp) && \
curl --socks5-host localhost:9050 -Lo "$GPGCONF" https://raw.githubusercontent.com/themand/macos-bootstrap/master/bootstrap/assets/dotfiles/.gnupg/gpg.conf
```

### Go offline

Terminate networking, it will no longer be needed.

### Prepare temporary directories

```bash
export GNUPGHOME=$(mktemp -d) && \
mv "$GPGCONF" "$GNUPGHOME"/gpg.conf && \
unset GPGCONF && \
export EXPORTDIR=$(mktemp -d) && \
mkdir "$EXPORTDIR"/secret
```

### Create a master key

Optional: if you don't have your own, strong passhprase (preferably generated using diceware), you can generate a random password:

```bash
gpg --gen-random -a 2 24
```

Generate a new key, using the following choices for interactive options:

```
gpg --expert --full-generate-key
- 8         RSA (set your own capabilities)
- E         Disable encryption
- S         Disable signing
- Q         Finish if allowed action is only: Certify
- 4096
- 0         Don't expire master key
- Y         Really don't expire
- ...name
- ...email
- ...comment (can be left blank)
- O         Okay
```

Export the key ID as a variable for further use (copy and paste it from generated output):

```bash
export KEYID=0xXXXXXXXXXXXXXXXX
```

### Create subkeys

Create subkeys for signing, encryption and auth. Change expiration time if you wish (e.g. to 1y), the following example uses no expiration.

```
gpg --expert --edit-key "$KEYID"

addkey
- 4         RSA (sign only)
- 4096
- 0         Never expire (change it to 1y or whatever you need)
- Y         This is correct
- Y         Really create

addkey
- 6         RSA (encrypt only)
- 4096
- 0         Never expire (change it to 1y or whatever you need)
- Y         This is correct
- Y         Really create

addkey
- 8         RSA (set your own capabilities)
- S         Disable sign
- E         Disable encrypt
- A         Enable authenticate
- Q         Finish if allowed action is only: Authenticate
- 4096
- 0         Never expire (change it to 1y or whatever you need)
- Y         This is correct
- Y         Really create

save
``` 

### Verify keys

```bash
gpg --list-secret-keys
```

You should see a master key (sec) and three subkeys (ssb).

### Export keys

#### Export private keys

```bash
export KEYNAME=$(gpg --list-keys "$KEYID" | grep uid | sed 's/.*<\(.*\)>.*/\1/g') && \
gpg --armor --export-secret-keys "$KEYID" > "$EXPORTDIR"/secret/"$KEYNAME".gpg.master.key && \
gpg --armor --export-secret-subkeys "$KEYID" > "$EXPORTDIR"/secret/"$KEYNAME".gpg.sub.key
```

Verify:

```bash
ls -l "$EXPORTDIR"/secret/"$KEYNAME".gpg.master.key "$EXPORTDIR"/secret/"$KEYNAME".gpg.sub.key
```

#### Export public keys

```bash
gpg --armor --export "$KEYID" > "$EXPORTDIR"/"$KEYNAME".gpg.pubkey.txt && \
gpg --armor --export-ssh-key "$KEYID" > "$EXPORTDIR"/"$KEYNAME".gpg.pubkey.ssh.txt
```

Verify:

```bash
ls -l "$EXPORTDIR"/"$KEYNAME".gpg.pubkey.txt "$EXPORTDIR"/"$KEYNAME".gpg.pubkey.ssh.txt
```

#### Archive GPG homedir

It includes master key revocation certificate which you will need in the future.

```bash
cd "$GNUPGHOME" && \
tar cf "$EXPORTDIR"/secret/"$KEYNAME".gpg-homedir.tar * && \
gzip "$EXPORTDIR"/secret/"$KEYNAME".gpg-homedir.tar && \
cd -
```

Verify:

```bash
ls -l "$EXPORTDIR"/secret/"$KEYNAME".gpg-homedir.tar.gz
```

#### Split exported secrets using shamir-split

Change desired threshold and number of shares.

```bash
export SHAMIR_THRESHOLD=2 && \
export SHAMIR_SHARES=3 && \
cd "$EXPORTDIR"/secret && \
"$BINDIR"/shamir-split "$SHAMIR_THRESHOLD" "$SHAMIR_SHARES" "$EXPORTDIR"/secret/"$KEYNAME".gpg-homedir.tar.gz -out "$EXPORTDIR"/secret/"$KEYNAME".gpg-homedir.tar.gz && \
"$BINDIR"/shamir-split "$SHAMIR_THRESHOLD" "$SHAMIR_SHARES" "$EXPORTDIR"/secret/"$KEYNAME".gpg.master.key -out "$EXPORTDIR"/secret/"$KEYNAME".gpg.master.key && \
"$BINDIR"/shamir-split "$SHAMIR_THRESHOLD" "$SHAMIR_SHARES" "$EXPORTDIR"/secret/"$KEYNAME".gpg.sub.key -out "$EXPORTDIR"/secret/"$KEYNAME".gpg.sub.key && \
unset SHAMIR_THRESHOLD && \
unset SHAMIR_SHARES
```

### Backup exported keys into a secure location

Preferably move shares into individual locations. Add destination location, change numbers.

```bash
mv "$EXPORTDIR"/secret/*-1 
```

### Recommended: include shamir-combine binaries in backup

It's also a good idea to store [shamir-sharing binaries](https://github.com/themand/shamir-sharing/releases) with actual shares (e.g. on the same USB drives) to ensure you will always be able to combine secrets.

Fill in `DESTDIR` with your destination.

```bash
DESTDIR=

mkdir -p "$DESTDIR"/bin/linux "$DESTDIR"/bin/macos "$DESTDIR"/bin/win && \
cp "$BINDIR"/shamir-split "$DESTDIR"/bin/linux/shamir-split && \
cp "$BINDIR"/shamir-combine "$DESTDIR"/bin/linux/shamir-combine && \
cp "$BINDIR"/macos-shamir-split "$DESTDIR"/bin/macos/shamir-split && \
cp "$BINDIR"/macos-shamir-combine "$DESTDIR"/bin/macos/shamir-combine && \
cp "$BINDIR"/win-shamir-split "$DESTDIR"/bin/win/shamir-split && \
cp "$BINDIR"/win-shamir-combine "$DESTDIR"/bin/win/shamir-combine
```

### Optional: reset GPG and re-import keys

In order to make sure that your export is valid and you remember the correct passphrase, you might want to reset/erase GPG keyring and re-import exported keys.

Erase GPG homedir:

```bash
export GPGCONF=$(mktemp) && \
mv "$GNUPGHOME"/gpg.conf "$GPGCONF" && \ 
srm -r "$GNUPGHOME" && \
export GNUPGHOME=$(mktemp -d) && \
mv "$GPGCONF" "$GNUPGHOME"/gpg.conf
``` 

Verify there are no keys:

```bash
gpg --list-secret-keys
```

Import previously exported keys:

```bash
gpg --import "$EXPORTDIR"/secret/"$KEYNAME".gpg.master.key
```

Verify keys

```bash
gpg --list-secret-keys
```

You should see a master key (sec) and three subkeys (ssb).

### Configure YubiKey GPG Smartcard

#### Require touch for GPG operations

```bash
"$BINDIR"/yubitouch sig on && \
"$BINDIR"/yubitouch aut on && \
"$BINDIR"/yubitouch dec on
``` 

#### Change PIN and Admin PIN

```
gpg --card-edit

admin
- 3         change Admin PIN
- 1         change PIN
- q         quit

quit
```

### Transfer keys to YubiKey

**Warning:** Transferring keys to YubiKey is a destructive, one-way operation only. Make sure you've made a backup before proceeding.

```
gpg --edit-key "$KEYID"

key 1

keytocard
- 1         Signature key

key 1

key 2

keytocard
- 2         Encryption key

key 2

key 3

keytocard
- 3         Authentication key

save
```

### Verify card keys

```bash
gpg --list-secret-keys
```

You should see a master key (sec) and three subkeys (ssb). The subkeys should exist only on keycard, which is indicated by *>* after ssb: *ssb>*

### Cleanup

Ensure that you have:

* Saved the Encryption, Signing and Authentication subkeys to YubiKey.
* Saved the YubiKey PINs which you changed from defaults.
* Saved the password to the Master key.
* Saved a copy of the Master key, subkeys and revocation certificates (GPG homedir)
* Saved a copy of the public key somewhere easily accessible later.

Reboot or securely delete directories containing secrets:

```bash
srm -r "$GNUPGHOME" && \
srm -r "$EXPORTDIR" && \
unset GNUPGHOME && \
unset EXPORTDIR
```

Delete downloaded binaries if you no longer need shamir-sharing

```bash
rm -r "$BINDIR" && \
unset BINDIR
```

## Using keys

@todo