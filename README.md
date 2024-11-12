# luks-clevis-initramfs-tpm2-multipartition-unlock
Simplified guide to enable auto unlock multiple luks partitions with initramfs and clevis on boot on debian 12 systems.

## Who is this for

For anypne who needs to unlock multiple LUKS partitions on Debian 12 systems without switching initramfs to dracut.

## Backround

The workaround addresses the following limitations of buggy implementations:

1. Clevis unlocks only the root partition on boot with TPM, secondary partitions require passphrase prompt
2. Interestingly systemd-cryptenroll unlocks only secondary partions due to a bug that ignores `tpm2-device=` option in `/etc/crypttab` for root partition.

## Workaround

If you are in a situation like me where you cannot move away from default initramfs in favor of dracut, you likely found yourself stuck with clevis or systemd-cryptenroll. Clevis works great for setups with only one LUKS encrypted volume. 
