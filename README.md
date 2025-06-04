This guide describe how set up system with detached LUKS header, Unified Kernel Image (UKI), Secure Boot with own keys and tpm2-totp in Devuan 5 (Debian 12) without systemd!

**Now until disk partition setup system via install media**

## Partitioning:
Disks after setup will be similar this:
```
/dev/sda - main storage
/dev/sdb - usb stick with ESP (/boot/efi), LUKS header partition
```

Make *ESP (/boot/efi)* and *LUKS header* partitions in **/dev/sdb**
```
parted -s /dev/sdb mklabel gpt
parted -s -a optimal /dev/sdb mkpart "EFI System Partition" "fat32" "0%" "1024MiB"
parted -s -a optimal /dev/sdb mkpart "primary" "ext4" "1024MiB" "1040MiB"
parted -s /dev/sdb set 1 esp on
```

Make */ (root)* encrypted partition in **/dev/sda**
```
parted -s /dev/sda mklabel gpt
parted -s -a optimal /dev/sda mkpart "primary" "ext4" "0%" "50%"
parted -s /dev/sda set 1 lvm on
```
Setup encryption in main storage with detached LUKS header:
*(Ensure that is optimal config for selected algorithms: `cryptsetup benchmark`)*
```
dd if=/dev/zero of=/tmp/header.img bs=16M count=1
cryptsetup luksFormat --offset 32768 --header /tmp/data.img --type luks2 --cipher twofish-xts-plain64 --key-size 512 --hash sha512 --pbkdf argon2id --iter-time 3000 --verify-passphrase /dev/sda1
cryptsetup open --header /tmp/data.img /dev/sda1 sda1_crypt
```

Setup LVM on LUKS
```
pvcreate /dev/mapper/sda1_crypt
vgcreate devuan-vg /dev/mapper/sda1_crypt

lvcreate -L 2G devuan-vg --name swap
lvcreate -l 100%FREE devuan-vg --name root
lvreduce -L -256M devuan-vg/root
```

Setup mount points, remove names of partiotions and install system from setup manager. *`(If ESP mounting error, try partitioning USB from setup manager)`*

Setup mounting LUKS when USB is inserted
 - Write /tmp/header.img to /dev/sdb2 (because it's probably unpossible mount with LUKS header as file, need mount with LUKS header as partition)
   ```
   dd if=/tmp/data.img of=/dev/sdb2
   ```
 - Write to /etc/crypttab
   ```
   sda1_crypt    /dev/disk/by-id/[part which symbol link to /dev/sda1]    none    luks,header=/dev/disk/by-uuid/[part which symbol link to /dev/sdb2]
   ```

Follow install steps until ending setup, without GRUB install!

## UKI (without Secure Boot integration):
Plug-in and mount another drive/USB stick with scripts from this repo for fast configuring and mount it to /mnt
```
mount /dev/sdc /mnt
```

Update APT preferences
```
./mnt/linux-secureboot-guide/system/update-apt
```

Add initramfs scripts
```
./mnt/linux-secureboot-guide/hardening/uki/install-initramfs-scripts
```

Unmount drive
```
umount /mnt
```

Install packages to target system for generating EFI stub
```
in-target sh -c "apt update"
apt-install systemd-ukify systemd-boot-efi efibootmgr
```

Update initramfs via
```
in-target sh -c "update-initramfs -u"
```

**Check if all right and reboot in new system:**
`Now press end setup in installation media and reboot in installed system`

## Setup Secure Boot:
At this step you already booted in instaled system. If all OK now configure Secure Boot, first you should enable 'Setup Mode' in UEFI settings.

>[!CAUTION]
>Perform all actions in this section at your own peril and risk, because incorrect execution can lead to a **breakdown of the device**!
>
>All manipulations with Secure Boot keys were made on a device with a video chip built into the processor, on other (and even similar) devices, the inclusion of Secure Boot with a discriminated video card can lead to a **black screen**!
>
>Look for more information on the network.

Install packages for generating keys and work with UEFI vars
```
sudo apt install efitools sbsigntool uuid-runtime
sudo rc-update del uuidd
sudo rc-service uuidd stop
```
Generate keys and store them encrypted on USB drive and in dir
```
sudo /mnt/linux-secureboot-guide/hardening/secureboot/efi-mkkeys -e -s "My OWN keys!" -o /tmp/keys/
```
Now in /tmp/keys appears _*.key (private keys)_, _*.crt (public keys)_, _*.cer (public keys in DER format)_, _*.esl (UEFI certs for enrolling in 'Setup Mode')_, _*.auth (singed UEFI certs for enrolling in 'User Mode')_

In theory you can just enrolling generated keys without cat with existings, but for reliability, it is better to combine them with existing ones. For example, here is an example of combining your DB keys with Microsoft keys:

In system has this keys
  - Trust Certificate				        - PK key, used to sign KEK keys
  - Microsoft Corporation KEK CA 2011	    - KEK key, used to sign db, dbx updates from Windows OS
  - Microsoft Corporation KEK 2K CA 2023	- KEK key, see above
  - Microsoft Windows Production PCA 2011	- db key, used to load Windows OS
  - Windows UEFI CA 2023					- db key, used to load Windows OS (?) but not only (?)

There is no keys, that may be used for load UEFI/other firmware, but for first time maybe usefull cat db with only cert 'Windows UEFI CA 2023'

Cat generated db.esl with 'Windows UEFI CA 2023'
```
openssl x509 -in /mnt/MicWinUEFICA2023.der -inform DER -out /tmp/MicWinUEFICA2023.pem -outform PEM
cert-to-efi-sig-list -g "$(cat /tmp/keys/guid.txt)" /tmp/MicWinUEFICA2023.pem /tmp/MicWinUEFICA2023.esl
cat /tmp/keys/db.esl /tmp/MicWinUEFICA2023.esl > /tmp/keys/dbMicWinUEFICA2023.esl
sign-efi-sig-list -t "$(cat /tmp/keys/timestamp.txt)" -a -k /tmp/keys/KEK.key -c /tmp/keys/KEK.crt db /tmp/keys/dbMicWinUEFICA2023.esl /tmp/keys/dbMicWinUEFICA2023.auth
```

Enroll keys to UEFI via efi-updatevar command in 'Setup Mode' (i.e. *.esl certs)
**`(PK key ALWAYS MUST BE ADDED LAST!!)`**
```
sudo efi-updatevar -a -e -f db.esl db
sudo efi-updatevar -a -e -f KEK.esl KEK
sudo efi-updatevar -f PK.auth PK
```	

If all OK, now we can delete all files in /tmp/keys, except *.key, *.crt, *.txt
```
rm -iv /tmp/keys/{*.cer,*.esl,*.auth}
```

Encrypt private keys via openssl
```
cd /tmp/keys
read -r -s pass
for name in PK KEK db; do
    printf '%s\n' "$pass" \
        | openssl rsa -aes256 -passout stdin -in "$name.key" -out "$name.key.enc"
    mv "$name.key.enc" "$name.key"
done
unset pass
```

Make correct rights to private/public keys and move them to another place
```
sudo mkdir -p /usr/share/keyrings/efi/
sudo mv /tmp/keys/* /usr/share/keyrings/efi/
sudo chown -R root /usr/share/keyrings/efi/*
sudo chgrp -R root /usr/share/keyrings/efi/*
sudo chmod 0600 /usr/share/keyrings/efi/*
sudo chmod 0600 /usr/share/keyrings/efi/
```

Now you can edit */usr/local/sbin/generate-uki* and enable generating singed UKI images

## tpm2-totp:
Install depends
```
sudo apt install git build-essential autoconf autoconf-archive automake m4 libtool pkg-config libqrencode-dev tpm2-tss-engine-dev --no-install-recommends
```

Now download and build tpm2-totp
```
git clone https://github.com/tpm2-software/tpm2-totp
cd tpm2-totp

# Apply patch that add support for initramfs-tools systems without plymouth
patch -p1 < /mnt/linux-secureboot-guide/hardening/tpm2/tpm2-totp_add-custom-secret_33e1986.patch
patch -p1 < /mnt/linux-secureboot-guide/hardening/tpm2/tpm2-totp_add-initramfs-support_33e1986.patch

./bootstrap
./configure --sysconfdir=/etc
make check
sudo make install
sudo ldconfig
```

Add script for correct resealing secrets
```
sudo /mnt/linux-secureboot-guide/hardening/tpm2/install-tpm2-totp-reseal
```

Update initramfs to integrate tpm2-totp in initrd
```
sudo update-initramfs -u
```

Setup tpm2-totp and generate TOTP secret
>[!NOTE]
>After every updating UEFI/UEFI settings (including boot entries)/GPT partition/updating initramfs need reseal secret via
>```
>tpm2-totp -P - reseal
>```
>If occurs error like `'0x90006 - mu:A buffer is not enough large'` after reseal command, that you should clean and init tpm2-totp again after every NEW initramfs LOADED (e.g. update kernel -> update-initramfs -> reboot system -> clean/init new totp code)
>
>Or simply use this for generating/resealing secret
>```
>tpm2-totp-reseal
>```

This config should allowing load in normal/recovery mode without altering about changed TPM PCR's state
```
sudo tpm2-totp -P - -b SHA256 -p 0,1,2,3,4,5,7,9,11,13,14 init
```

Done! On boot it's show TOTP code, which should be equvivalent to code on phone

## Sources:
1. Partitioning
	- https://wiki.artixlinux.org/Main/InstallationWithFullDiskEncryption
	- https://wiki.archlinux.org/title/LVM

2. LUKS
	- https://wiki.archlinux.org/title/Dm-crypt/Specialties#Encrypted_system_using_a_detached_LUKS_header

3. UKI
	- https://wiki.debian.org/EFIStub#Setting_up_a_Unified_Kernel_Image
	- https://wiki.archlinux.org/title/Unified_kernel_image

4. Secure Boot
	- https://www.rodsbooks.com/efi-bootloaders/controlling-sb.html
    - https://github.com/jirutka/efi-mkkeys/

	- https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot#Microsoft_Windows
	- https://github.com/microsoft/secureboot_objects
	- https://habr.com/ru/articles/273497/

	- https://wiki.gentoo.org/wiki/Secure_Boot#Installing_the_keys

5. TPM
	- https://github.com/tpm2-software/tpm2-totp
	- https://wiki.archlinux.org/title/Trusted_Platform_Module#Accessing_PCR_registers
	- https://github.com/uapi-group/specifications/blob/main/specs/linux_tpm_pcr_registry.md

6. Guides
   1. Secure boot
        - https://habr.com/ru/articles/308032/

   2. Complete (UKI + Secure boot + LUKS)
		 - https://habr.com/ru/companies/flant/articles/835778/
		 - https://forum.manjaro.org/t/howto-using-secure-boot-and-tpm2-to-unlock-luks-partition-on-boot/101626

