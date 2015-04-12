yout: post
title: Yubikey NEO for GPG, SSH and Profit
description: "A setup guide to configure Yubikey Neo smart card"
modified: 
tags: [GPG, Information Security, Encryption, Yubikey, Cryptostick]
image:
  feature:
  credit:
  creditlink:
comments: true
share: true
---

## TLDR

I am updating my GPG keys

## Process

I am cleaning up my GPG keys along the way as part of my DR plan. At the moment I have a mess of keys everywhere! Most legacy keys have expired, been revoked or ultimately lost through mismanagement. My aim is to arrive at a point where I can carry with me my identity for both SSH and GPG.

I am generating a day-to-day GPG key to use via my [Yubikey NEO](https://www.yubico.com/products/yubikey-hardware/yubikey-neo/). I will also create a backup GPG smart card using a spare [German Privacy Foundation Cryptostick v1.2](http://www.privacyfoundation.de/projekte/crypto_stick/). Finally, the master copy remains on a physically secured USB key(s).

What follows below is notes on recreating this whole thing from scratch again.

### Trusted Live Environment

To generate new keys I first boot a ancient laptop using an [isostick](http://isostick.com/) and a copy of the Fedora 21 Workstation image. After verifying the sha256sum I copied the ISO to the isostick and can boot via the virtual CD-ROM.

Once booted and presented with the installers grub menu, select **Troubleshooting** then **Test this media & start Fedora Live**. At this stage I also press tab and remove the *quiet rhbg* arguments so that I get feedback on the boot process.

At the gnome prompt click **Try Fedora** to continue into the live OS.

#### Extending the Live Environment

The first time you boot you will need to connect your environment to the Internet and grab some packages for offline use later.

    sudo yum install yum-utils
    yumdownloader --resolve ykpers-devel libyubikey-devel libusb-devel autoconf gnupg gnupg2-smime pcsc-lite pcsc-lite-libs
    yumdownloader --resolve gnupg-agent libpth20 pinentry-curses libccid pcscd scdaemon libksba8

Full deps in the case of resolve not working, it seems that pre-installed packages are not pulled into the download directory leaving you with broken deps.

	autoconf-2.69-16.fc21.noarch.rpm
	elfutils-0.161-2.fc21.x86_64.rpm
	elfutils-libelf-0.161-2.fc21.x86_64.rpm
	elfutils-libs-0.161-2.fc21.x86_64.rpm
	glibc-2.20-7.fc21.x86_64.rpm
	glibc-common-2.20-7.fc21.x86_64.rpm
	gnupg-1.4.18-4.fc21.x86_64.rpm
	gnupg2-smime-2.0.25-2.fc21.x86_64.rpm
	libgudev1-216-17.fc21.x86_64.rpm
	libusb-devel-0.1.5-5.fc21.x86_64.rpm
	libusbx-devel-1.0.19-2.fc21.x86_64.rpm
	libyubikey-1.11-3.fc21.x86_64.rpm
	libyubikey-devel-1.11-3.fc21.x86_64.rpm
	m4-1.4.17-6.fc21.x86_64.rpm
	nss-softokn-freebl-3.17.4-1.fc21.x86_64.rpm
	pcre-8.35-8.fc21.x86_64.rpm
	pcsc-lite-1.8.13-1.fc21.x86_64.rpm
	pcsc-lite-ccid-1.4.18-1.fc21.x86_64.rpm
	pcsc-lite-libs-1.8.13-1.fc21.x86_64.rpm
	systemd-216-17.fc21.x86_64.rpm
	systemd-compat-libs-216-17.fc21.x86_64.rpm
	systemd-libs-216-17.fc21.x86_64.rpm
	systemd-python-216-17.fc21.x86_64.rpm
	systemd-python3-216-17.fc21.x86_64.rpm
	ykpers-1.16.1-1.fc21.x86_64.rpm
	ykpers-devel-1.16.1-1.fc21.x86_64.rpm


After yum has downloaded the packages to your working directory copy them to media you can attach to your offline machine.

Reboot to start a clean instance and mount your storage containing the downloaded packages.

    yum localinstall *

Now we can get started generating keys!

### RAID

While some people might trust their cold storage, since I am using what are actually very crappy USB keys I thought I might RAID then just in case one decides to flip a bit on me.

### Creating the array

First lets test both disks are actually OK

    dd if=/dev/zero of=/dev/disk/by-id/YOUR-DISK

Create the RAID partitions

    fdisk /dev/disk/by-id/YOUR-DISK
    n
    p
    1
    enter
    enter
    t
    fd
    w

Create the array

    mdadm --create md0 -n 2 -l 1 /dev/sdc1 /dev/sdd1

Lets format it!

    mkfs.ext4 /dev/md0
    mkdir /tmp/raid
    mount /dev/md0 /tmp/raid

### Further use

Once we are done we will stop the array to unplug the USB keys

    mdadm -S md0

To re-use this array next time we scan for existing RAID with both disks plugged in

    mdadm --assemble -s

### Generate the master key

After opening your terminal window, you need to update the environment variables for gnupg.

    export GNUPGHOME=/tmp/raid/gnupg
    mkdir $GNUPGHOME

**Note:** You cannot mount this on a vfat volume as gpg-agent will not be able to open a unix socket.

Now we update our config

    cat > $GNUPGHOME/gpg.conf
    personal-cipher-preferences AES256 AES192 AES
    personal-digest-preferences SHA512 SHA384 SHA256 SHA224
    cert-digest-algo SHA512
    default-preference-list SHA512 SHA384 SHA256 SHA224 AES256 AES192 AES ZLIB BZIP2 ZIP Uncompressed
    use-agent

**Note:** You can't opt out of 3DES and SHA with this configuration. gnupg will automatically add them to the trailing end of your preferences.

After the configuration is done we generate the master key to be used only for signing operations.

        gpg2 --gen-key
        Your Selection? 4
        4096
        2y
        y
        Sam Wilson
        cycloptivity@internode.on.net
        O

Now we can add our extra UID's

        gpg2 --edit-key <YOUR KEY ID>
        adduid
        Sam Wilson
        swilsonau@gmail.com
        O
        uid 1
        primary
        save

Generate a revocation certificate

        gpg2 --output $GNUPGHOME/../revocation-certificate.txt --gen-revoke <YOUR KEY ID>
        1
        Created during key creation, emergency use only.

Backup the private keys to ascii

    gpg2 -a --export-secret-keys <YOUR KEY ID> > $GNUPGHOME/../masterkey.txt

### Generate sub keys

First we generate separate 2048 bit RSA keys for signing, authentication and encryption.

        gpg2 --expert --edit-key <YOUR KEY ID>
        addkey
        4
        2048
        2y
        y
        y
        addkey
        6
        2048
        2y
        y
        y
        addkey
        8
        s
        e
        a
        q
        2048
        2y
        y
        y
        save

Lets backup our sub keys.

    gpg2 -a --export-secret-keys <YOUR KEY ID> > $GNUPGHOME/../mastersubkeys.txt
    gpg2 -a --export-secret-subkeys <YOUR KEY ID> > $GNUPGHOME/../subkeys.txt

Also backup the $GNUPGHOME binary content in case we need to roll back GPG during later steps.

You can print to hard copy the text files we are created now.

### Configure Yuibkey NEO

I found starting up ```gpg2 --card-edit``` as **liveuser** failed to open the smartcard. Running as root resolves the issue.

Lets configure the Yubikey NEO!

        gpg2 --card-edit
        admin
        passwd
        3
        12345678 # Default admin passwd
        <NEWPIN>
        <NEWPIN>
        1
        123456 # Default user passwd
        <NEWPIN>
        <NEWPIN>
        q
        name
        Wilson
        Sam
        lang
        en
        url
        http://www.cycloptivity.net/KEYID.txt
        sex
        m
        quit

Note: If your looking for a random pin generator try ```< /dev/urandom tr -cd 0-9 | head -c 6```.

Now we start to move our sub keys to hardware. This is a one way operation and will leave the backups we took earlier as the only copy of your sub keys (except for the smart card of course!).

        gpg2 --edit-key <YOUR KEY ID>
        toggle
        key 1
        keytocard
        1
        key 1
        key 2
        keytocard
        2
        key 2
        key 3
        keytocard
        3
        save

**Note:** At this point, I actually hit [this bug](http://forum.yubico.com/viewtopic.php?f=26&t=1692), after raising a case with Yubico to get a new unit I got started again.

## Public key

You need to generate a copy of your master public key to share with the world. Lets make that available quickly.

    gpg2 --export -a <KEY ID> >> $GNUPGHOME/../pub.asc

Take a copy of the **pub.asc** file to your daily laptop along with your Yuibkey.

## Setup day to day laptop

### Desktop Environments
While technically I should have been able to configure XFCE on my laptop to disable ssh-agent I found this had no effect on Fedora 21.

    xfconf-query -c xfce4-session -p /startup/ssh-agent/enabled -n -t bool -s false

Instead its much easier to just hack on this via trusty ~/.bashrc

    killall ssh-agent gpg-agent > /dev/null 2>&1
    eval $(gpg-agent --daemon --enable-ssh-support)
    
### GPG

[[TODO - Importing onto daily machine]]

## References

These posts helped me out a lot when writing this! YMMV

[https://help.riseup.net/en/security/message-security/openpgp/best-practices](https://help.riseup.net/en/security/message-security/openpgp/best-practices)

[http://blog.josefsson.org/2014/06/23/offline-gnupg-master-key-and-subkeys-on-yubikey-neo-smartcard/](http://blog.josefsson.org/2014/06/23/offline-gnupg-master-key-and-subkeys-on-yubikey-neo-smartcard/)

[https://github.com/herlo/ssh-gpg-smartcard-config/blob/master/YubiKey_NEO.rst](https://github.com/herlo/ssh-gpg-smartcard-config/blob/master/YubiKey_NEO.rst)

[http://www.bradfordembedded.com/2013/12/yubikey-smartcard/](http://www.bradfordembedded.com/2013/12/yubikey-smartcard/)

[http://ekaia.org/blog/2009/05/10/creating-new-gpgkey/](http://ekaia.org/blog/2009/05/10/creating-new-gpgkey/)

[https://www.debian-administration.org/users/dkg/weblog/97](https://www.debian-administration.org/users/dkg/weblog/97)

[https://developers.yubico.com/ykneo-openpgp/ResetApplet.html](https://developers.yubico.com/ykneo-openpgp/ResetApplet.html)

[https://josefsson.org/key-transition-2014-06-22.txt](https://josefsson.org/key-transition-2014-06-22.txt)

[http://www.bootc.net/archives/2013/06/09/my-perfect-gnupg-ssh-agent-setup/](http://www.bootc.net/archives/2013/06/09/my-perfect-gnupg-ssh-agent-setup/)

[http://dst.lbl.gov/ksblog/2011/05/xfce-without-gpg-agent/](http://dst.lbl.gov/ksblog/2011/05/xfce-without-gpg-agent/)

[http://docs.xfce.org/xfce/xfce4-session/advanced](http://docs.xfce.org/xfce/xfce4-session/advanced)
### 
