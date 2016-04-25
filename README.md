# Storing your LUKS key in TPM NVRAM

First read BUILD, to make sure you have all the runtime pre-reqs installed,
including the upstream trousers and tpm-tools packages.

1. Required steps
2. If the LUKS volume is not your rootfs
3. If the LUKS volume is your rootfs
  1. RHEL6
  2. Fedora 17
4. Sealing your NVRAM area to PCR state
5. Backup

## Required steps

1. You can check that your TPM is available by looking for /dev/tpm0, which
   will exist if a kernel driver is loaded. If not you'll need to load the tpm_tis
   module (or other TPM 1.2 module depending on your platform). On RHEL 6, the tpm
   driver is built into the kernel -- on Fedora 17, you'll need to install the
   kernel-modules-extra package to get tpm_tis.
2. Install tpm-luks, tpm-tools >= 1.3.8, trousers >= 0.3.9. Available at
   sf.net/projects/trousers. Start the tcsd:
   ```
   $ tcsd
   ```
   trousers 0.3.9 is included with Fedora 17. On Ubuntu, /dev/tpm0 ownership might
   need to be changed to tss:tss: `chown tss:tss /dev/tpm0` if tcsd fails to start.

   You can test if trousers and tpm-tools are working ok by running tpm_nvinfo.
   If it errors out with missing library errors after a build, follow these steps:

   After the trousers build:

   ```
   $ echo "/usr/local/lib" >> /etc/ld.so.conf
   $ ldconfig
   ```

   or during the build:

   ```
   $ ./configure --prefix=/usr
   ```

3. Take ownership of your TPM if you haven't before:

   ```
   $ tpm_takeownership
   ```

4. Mount securityfs:

   ```
   $ mount -t securityfs securityfs /sys/kernel/security
   ```

   and add to /etc/fstab to remount it automatically:

   ```
      securityfs              /sys/kernel/security    securityfs defaults 0 0
   ```


## If the LUKS volume is not your rootfs

1. Determine your LUKS encrypted partions:

   ```
   $ blkid -t TYPE=crypto_LUKS
   /dev/sda2: UUID="4cb97e1f-b921-4f1a-bd86-032831b277af" TYPE="crypto_LUKS"
   ```

2. Add a new LUKS key to a key slot and the TPM:

   ```
   # tpm-luks -c -d /dev/sda2
   Enter a new TPM NV area password: 
   Re-enter the new TPM NV area password: 
   Enter your TPM owner password: 
   Successfully wrote 33 bytes at offset 0 to NVRAM index 0x2 (2).
   You will now be prompted to enter any valid LUKS passphrase in order to store
   the new TPM NVRAM secret in LUKS key slot 1:

   Enter any passphrase: 
   Using NV index 2 for device /dev/sda2
   ```

   tpm-luks creates a 32-byte binary key and writes it TPM NVRAM. An extra byte
   is prepended as a version check.

## If the LUKS volume is your rootfs

These setup steps for RHEL and Fedora are required to include your current kernel
and initramfs in the trust chain (if configured in D.) and to insert the code into
your initramfs to read the LUKS secret from the TPM.

### RHEL 6 (may work elsewhere but so far only tested on RHEL 6)

Run tpm-luks-init, or do these steps manually:

1. Determine your LUKS encrypted partions:

   ```
   $ blkid -t TYPE=crypto_LUKS
   /dev/sda2: UUID="4cb97e1f-b921-4f1a-bd86-032831b277af" TYPE="crypto_LUKS"
   ```

2. Add a new LUKS key to a key slot and the TPM:

   ```
   # tpm-luks -c -d /dev/sda2
   Enter a new TPM NV area password: 
   Re-enter the new TPM NV area password: 
   Enter your TPM owner password: 
   Successfully wrote 33 bytes at offset 0 to NVRAM index 0x2 (2).
   You will now be prompted to enter any valid LUKS passphrase in order to store
   the new TPM NVRAM secret in LUKS key slot 1:

   Enter any passphrase: 
   Using NV index 2 for device /dev/sda2
   ```

3. Add code to query the TPM to the initramfs:

   ```
   $ dracut /boot/initramfs-2.6.32-XXX.el6.x86_64-tpm-luks.img
   ```

4. Create a new boot entry that uses the new initramfs:

   ```
   $ vi /boot/grub/menu.lst
   ```

   (The only change you need to make here is to copy the current boot entry
   for the RHEL kernel and change the initramfs path to
   /boot/initramfs-2.6.32-XXX.el6.x86_64-tpm-luks.img)

### Fedora 17

Do these steps manually:

1. Determine your LUKS encrypted partions:

   ```
   $ blkid -t TYPE=crypto_LUKS
   /dev/sda2: UUID="4cb97e1f-b921-4f1a-bd86-032831b277af" TYPE="crypto_LUKS"
   ```

2. Add a new LUKS key to a key slot and the TPM:

   ```
   $ tpm-luks -c -d /dev/sda2
   Enter a new TPM NV area password: 
   Re-enter the new TPM NV area password: 
   Enter your TPM owner password: 
   Successfully wrote 33 bytes at offset 0 to NVRAM index 0x2 (2).
   You will now be prompted to enter any valid LUKS passphrase in order to store
   the new TPM NVRAM secret in LUKS key slot 1:

   Enter any passphrase: 
   Using NV index 2 for device /dev/sda2
   ```

3. Add code to query the TPM to the initramfs:

   ```
   $ dracut /boot/initramfs-3.4.4-5.fc17.x86_64-tpm-luks.img
   ```

4. Create a new boot entry that uses the new initramfs:

   ```
   $ vim /boot/grub2/grub.cfg
   ```

   (The only change you need to make here is to copy the current boot entry
   for Fedora and change the initramfs path to 
   /boot/initramfs-3.X.X-X.fc17.x86_64-tpm-luks.img)

   From https://fedoraproject.org/wiki/GRUB_2:
   "It is safe to directly edit /boot/grub2/grub.cfg in Fedora."

8. Reboot

## Sealing your NVRAM area to PCR state

"Sealing" means binding the TPM NVRAM data to the state of your machine. Using
sealing, you can require any arbitrary software to have run and recorded its
state in the TPM before your LUKS secret would be released from the TPM chip.
The usual use case would be to boot using a TPM-aware bootloader which records
the kernel and initramfs you've booted. This would prevent your LUKS secret
from being retrieved from the TPM chip if the machine was booted from any other
media or configuration.

To get a full chain of trust up through your initramfs, you'll first need to
install TrustedGrUB, available from http://sourceforge.net/projects/trustedgrub/.
A vanilla install of TrustedGrUB doesn't appear to work with Fedora 17 -- if
you get TrustedGrUB working with recent fedora distros, please send a note
to trousers-users@lists.sf.net or shpedoikal@gmail.com.

Note that trustedgrub is supported 32bit only, so you'll need for example
the glibc-devel.i686 and libgcc.i686 packages to build it on x86_64.

Once you've installed TrustedGrub successfully, reboot, then continue
with these steps:

1. Edit /etc/tpm-luks.conf and set either the 'profile' or 'pcrs' option
   to tell tpm-luks to use the PCRs you choose. You'll want to take some time
   and make sure you really understand what you're doing here. If you remove
   your non-TPM keys from your LUKS header and then your system config
   changes, you could lose access to your LUKS partition. Make sure you backup
   your LUKS header before removing all the non-TPM keys!
   ATM, only the "srtm" profile or PCRs 0-15 are supported.

2. Complete the steps in C.I. or C.II. above

3. At yum update time:
  tpm-luks installs a yum post-transaction hook in
  /etc/yum/post-actions/tpm-luks.action. Whenever the kernel package is
  updated, the hook runs the tpm-luks-update script, which attempts to
  migrate your current TPM NVRAM secret to the new PCR values for the
  changed kernel+initramfs.

## Backup

1. Backup your current LUKS header
   ```
   $ cryptsetup luksHeaderBackup <device> --header-backup-file <file>
   ```

2. Remove the LUKS key slot with the non-TPM key, using a secret held
   in the TPM:
   ```
   $ tpm-luks -k -s <slot>
   ```
