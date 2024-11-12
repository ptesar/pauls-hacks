# luks-clevis-initramfs-tpm2-multipartition-unlock
Simplified guide to enable auto unlock multiple luks partitions with initramfs and clevis on boot on debian 12 systems.

## Who is this for

Anyone who needs to unlock multiple LUKS partitions on Debian 12 systems without switching from initramfs to dracut.

## Background

The workaround addresses the following limitations of buggy implementations for LUKS volume unlocking with TPM on boot:

1. Clevis unlocks only the root partition on boot with TPM, secondary partitions require passphrase prompts
2. Interestingly systemd-cryptenroll unlocks only secondary partions due to a bug that ignores `tpm2-device=` option in `/etc/crypttab` for root partition

## Workaround

If you are in a situation like me where you cannot move away from default initramfs in favor of dracut, you likely found yourself stuck with clevis or systemd-cryptenroll. Clevis works great for setups with only one LUKS encrypted volume. 

### 1. Use systemd-cryptenroll for secondary partitions

Enroll all secondary volumes with `systemd-cryptenroll`, replace `/dev/sdaX` with your specific device identifier.

```
sudo systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=1+7 /dev/sdaX
```

Adjust the PCR banks to your needs. You will be propted for your current LUKS key.

### 2. Add TPM device parameter to /etc/crypttab volume entries

Assuming your `/etc/crypttab` is already populated, add `tpm2-device=auto` for each for each secondary LUKS volume. For example:

```
nvme1n1p1_crypt UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx none luks,tpm2-device=auto,discard
```

The name and UUID should match your specific volumes. Skip the primary LUKS volume, we will deal with that one later.

### 3. Rebuild the initramfs

Once you enroll all secondary LUKS volumes with `systemd-cryptenroll` and update the `/etc/crypttab` file, build the initramfs:

```
sudo update-initramfs -u
```

Once done you can condirm that the auto unlock of secondary LUKS volumes works by rebboting your system - you should be prompted only for one passphrase for the primary LUKS volume, all remaining volumes should unlock automatically.

### 4. Install clevis

The magic trick to get the primary LUKS volume unlocked on boot with TPM is to use clevis as fallback. First, we install the needed libraries:

```
sudo apt-get -y install clevis clevis-tpm2 clevis-luks clevis-initramfs initramfs-tools tss2
```

What's important here is that we are not downloading and using dracut that would replace the initramfs, instead we rely on `initramfs-tools` and `clevis-initramfs`. 

### 5. Use clevis to bind the primary LUKS volume

We do this step only for the primary LUKS volume, not already covered by `systemd-cryptenroll`:

```
sudo clevis luks bind -d /dev/nvme2n1p3 tpm2 '{"key":"rsa", "pcr_bank":"sha256", "pcr_ids":"1,7"}' <<< "your-secret-phrase-here"
```

The device should match your specific primary LUKS volume, `/dev/nvme2n1p3` refects general setup of nowadays on systems with nvme storage. The `"key":"rsa"` parameter is optional on most systems.
