# GPG Masterkey management toolset #

These tools automate the creation of GPG masterkeys on macOS.  
The intended audience is people using a [YubiKey](https://www.yubico.com/products/yubikey-hardware/)
or other smartcard.

## Dependencies ##
The only dependency is the GPG Suite from [gpgtools.org](https://gpgtools.org/).  
Alternatively you can install "GnuPG modern", which is the 2.1 version
and includes major improvements (but no deep macOS integration) with
`brew install gnupg` (check out [the notes on how to get gpg 2.1 set up](SETUP-2.1.md)).

## Resources ##

[Offline GnuPG Master Key and Subkeys on YubiKey NEO Smartcard](https://blog.josefsson.org/2014/06/23/offline-gnupg-master-key-and-subkeys-on-yubikey-neo-smartcard/)  
[PGP and SSH keys on a Yubikey NEO](https://www.esev.com/blog/post/2015-01-pgp-ssh-key-on-yubikey-neo/)  
The two primary sources for this guide.

[Yubikey, GnuPG 2.1 Modern, and SSH on macOS](https://www.rempe.us/blog/yubikey-gnupg-2-1-and-ssh/)  
The source for [SETUP-2.1.md](SETUP-2.1.md).

[drduh/YubiKey-Guide](https://github.com/drduh/YubiKey-Guide)  
Extensive guide on how to get GnuPG working with your YubiKey

## Walkthrough ##
You will be creating a secure directory that is exempt from spotlight indexing
and general system access and only readable by you.  
This is the poor man's version of an airgapped machine.

Inside that secure directory you will be creating a masterkey and an encryption
key. You will need to back these keys up to some secure storage
(e.g. [VeraCrypt](https://www.veracrypt.fr/en/Home.html)), just remember to keep
backups of your backups, _USB keys are not a reliable storage medium_.

Only run the commands with the `$` in front. The rest is output, so you can see
where you went wrong if something doesn't work.
```sh
# Create a secure directory that GnuPG can work in
$ sudo ./create-secure-dir.sh
Creating ramdisk
Formatting ramdisk at /dev/disk2
Initialized /dev/rdisk2 as a 1024 KB case-insensitive HFS Plus volume
Creating mountpoint at .../secure
Mounting volume
Setting permissions on mountpoint
.../secure: 00 -> 0100000
Telling Spotlight to not index the volume
.../secure:
2016-10-09 10:39:35.684 mdutil[84268:21085066] mdutil disabling Spotlight: .../secure -> kMDConfigSearchLevelFSSearchOnly
	Indexing disabled.

Secure directory created at '.../secure'.
The directory can be destroyed with ./destroy-secure-dir.sh

# Generate a master & encryption key
$ ./generate-master.sh 'John Doe' 'jd@example.com'
Generating master key and encryption subkey for "John Doe" <jd@example.com>
Key ID is E22FE7692F473FA12F2BAB164046979C50C10E97

# Change the GnuPG home dir, so that you can interact with the keys you just created
$ export GNUPGHOME=$PWD/secure/gnupg-home

# Make sure gpg-agent uses the new home dir
$ killall gpg-agent

# Add a photo to your key, copy the encryption key to your smartcard
# and sign the authentication & signing keys with your master key
$ gpg2 --edit-key E22FE7692F473FA12F2BAB164046979C50C10E97

# Once all keys have been created, back them up
$ ./backup.sh E22FE7692F473FA12F2BAB164046979C50C10E97
Backing up private keys
Backing up public keys
Generating revocation certificate

# Store the backup in a safe place
$ cp -r secure/backup /Volumes/encrypted-storage

# Import all public keys into your regular GPG keychain
$ unset GNUPGHOME
$ gpg2 --import secure/backup/E22FE7692F473FA12F2BAB164046979C50C10E97.public.asc

# Done! Kill the secure directory
$ sudo ./destroy-secure-dir.sh
Unmounting the volume
Volume secure on disk2 unmounted
Deleting the mountpoint
Ejecting the disk
Disk /dev/disk2 ejected
```

## Other tools ##

### SSH auth ###

Check out [SETUP-2.1.md](SETUP-2.1.md) on how to get the gpg-agent running on
macOS when you are using GnuPG 2.1.

Get your public SSH key with:
```sh
# With GPGTools
gpgkey2ssh E22FE7692F473FA12F2BAB164046979C50C10E97
# With gpg 2.1
gpg2 --export-ssh-key E22FE7692F473FA12F2BAB164046979C50C10E97
```
Add it to wherever you want to authenticate with your GPG authentication key.

### GitHub commit signing ###

Export your public key with:
```sh
gpg2 --armor --export E22FE7692F473FA12F2BAB164046979C50C10E97
```

And paste it into into the GPG field on https://github.com/settings/keys

### git config ###

* `commit.gpgSign = true`: Always sign commits
* `push.gpgSign = if-asked`: Enable signing of pushes
* `gpg.program = gpg2`: ... instead of `gpg`

### Fallback private key ###

For DB editors that do not support ssh agents, add a restricted normal
private key auth.  
`$HOME/.ssh/authorized_keys`
```
no-pty,no-X11-forwarding,permitopen="127.0.0.1:5432",command="/bin/echo do-not-send-commands" ssh-rsa private_key jd+postgres-only@example.com
```
