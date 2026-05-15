Measured Boot Overview
======================

# Table of Contents

# Purpose

This document is to give an overview of measured boot and what it does. It additionally describes one approach of measured boot on Linux.

# Background

Measured boot, simply put, records the hash of some firmware/binary/key/string representing some "event" being loaded to the system into some cryptographically-resistant chip. This chip will then include that hash into the current hash. That new hash reflects the hashes of all events that were submitted into this chip. With this, each event is recorded and the final hash can be checked later for comparison against some known good value(s), based on implementation/policy. More specifically, the hash cannot be directly set to a specific value by an application; instead, it can only be extended with an additional value.

For recording events, a TPM 2.0 chip is frequently used for this purpose. A TPM is a secure cryptoprocessor that, among other things, has 24 different banks (known as PCRs, or platform configuration registers) available for recording and using hashes. It also keeps an event log with the description of each event recorded in each PCR; this will be useful in remote attestation later. The UAPI Group has a [page listing what each PCR is used for](https://uapi-group.org/specifications/specs/linux_tpm_pcr_registry/) on Linux systems. The [TCG PC Client Platform Specific Profile](https://trustedcomputinggroup.org/resource/pc-client-specific-platform-firmware-profile-specification/) specification designates PCR 0-7 to be owned by the firmware (which will generally be UEFI) along with what each PCR is used for, 8-15 to be usable by the OS however it wants to use it, PCR 16 to be meant for debugging purposes only, and PCR 23 to be usable by any application (but not necessarily preserved if the system were to go to sleep). PCRs 17-22 are used by the dynamic CRTM (Core Root of Trust for Measurement), or DRTM.

There are different kinds of TPM implementations available today. The most common one is fTPM, or firmware TPM, which is typically supported via the firmware (such as UEFI) and uses a trusted execution environment within the CPU. Most if not all recent Intel and AMD chips support this. The rest of the document will assume the use of a fTPM, although there should be no changes required for using a different TPM implementation besides making sure that it is available on the target environment.

It's worth noting that nothing about measured boot (by itself) is meant to block anything from happening; the purpose of measured boot is simply to measure important events.

# Measurements during OS bootup and runtime (Linux-specific)

PCRs 0-7 are populated by the firmware; on most systems, this will be UEFI. Next, the bootloader may record some entries into the TPM. In the case of GRUB, if the tpm module is loaded, then GRUB will record all commands run (including ones that are read from the configuration files) and the kernel commandline into PCR 8, and all files read by GRUB into PCR 9. If systemd-boot is used, then it will record the `loader.conf` that it read into PCR 5. If systemd-stub is used as well, then if a UKI is used, then each component of the UKI is recorded into PCR 11 (described in more detail in the next section); otherwise, if a regular kernel and initrd is being loaded, then the kernel command line and the initrd is recorded into PCR 11.

## Measurements made via UKI and systemd

When booting via an unified kernel image (UKI) with systemd-stub embedded into the UKI, and if the initrd environment has systemd tools included (which will likely be the case if using something like dracut to generate the initrd), then there are measurements made into PCR 11. Specifically, when the UKI is loaded by UEFI (which is possible because of systemd-stub), systemd-stub will measure the different sections of the UKI into PCR 11, then start booting up the kernel. Then, systemd in the initrd environment extends PCR 11 when entering the initrd, leaving initrd, after system initialization (after basic services have started up), and when the system reaches a ready state (when user sessions are ready to be started). This is getting into trusted boot a little bit, but these extension points could be used to decrypt keys that depend on the state of this (and other) PCRs, such as TPM2-backed LUKS encryption of a partition (which is supported by systemd), such that these keys can only be decrypted in initrd, but not at any other point.

## Integrity Measurement Architecture

All of this covers everything up to the point that the kernel and initrd is loaded; however, here, the userspace binaries that are loaded are not covered here. For this to happen, there is an Integrity Measurement Architecture (IMA) available in the Linux kernel, which can record the hashes of files that are read or loaded into memory, as defined by some policies. If enabled, if a file is being read or loaded into memory (either to execute the file itself or as part of another program), it is first checked against the configured policy to determine whether it should be measured or not. If the policy decides that it should be measured, then the IMA will take a hash of the file contents, add it into the IMA log that the kernel maintains, and record the event into PCR 10. This particular mode of the IMA only measures the files, it does not block execution/read of any files; this can be used later with trusted boot. The IMA log is accessible at `/sys/kernel/security/integrity/ima/ascii_runtime_measurements` and `/sys/kernel/security/integrity/ima/binary_runtime_measurements`, to make it possible to audit the hashes of the files against PCR 10 (useful later with remote attestation). IMA can also validate the measurement made for a file against a hash recorded in the extended attributes for the file; this is IMA appraisal, and will be covered more in trusted boot.

While IMA _can_ measure every file that gets opened or executed, and doesn't remeasure files that haven't changed since the last measurement, doing so can cause a noticeable performance impact. This is because access to the TPM is serialized, and the TPM is slow. If every file needs to be measured, then the TPM access will become a bottleneck in how fast files can be opened/loaded into memory. For this reason, it makes sense to define a policy such that only specific files and/or files read by specific users are measured. This does mean that there will be a gap in what is measured.

# Possible implementation

## Initrd

One possible implementation of measured boot (keeping in mind trusted boot will be implemented following this or as part of this) is to use dracut to generate the initrd environment, and then generate a UKI using `ukify`, which will embed `systemd-stub`. There are two reasons why dracut and UKI should be used:

* Dracut (and its modules) has support for including systemd in the initrd, as well as including the TPM2 and (for trusted boot) cryptsetup tools/units necessary for the PCR 11 measurements to work. Initramfs-tools does not support this easily, as its architecture is different (it's probably not impossible, but very difficult and requires some manual implementation). This does mean that any custom logic in the initramfs scripts in SONiC will need to be ported to dracut or modified to use "normal" methods (for example, filesystem mounts will need to be specified in `/etc/fstab`).
* A UKI includes the kernel vmlinuz, initrd, the kernel command line to boot up with, and any additional files/metadata into a single `.efi` file that can be booted via UEFI (typically with the help of `systemd-stub`). Because it is a single executable file with any runtime changes being easily visible and auditable (i.e. someone booting with a custom cmdline besides the default cmdline in the UKI), it is easier to measure it. It is also possible to sign the UKI with the secure boot certs, which would mean someone building a new UKI will not be able to boot it on the system.

## Userspace

For critical userspace processes, the IMA infra in Linux can be used to measure files getting loaded. As mentioned earlier, since TPM access is serialized and slow, it may make sense to start with a very limited policy. The kernel does provide some IMA policy templates that could be used as a starting point that can be activated via cmdline, such as `ima_policy=tcb`. This policy will ignore files on special/virtual filesystems and measure all files being executed or being mapped with executable bit set, along with files that are being read by root or a process run as root, along with kernel modules that are loaded. However, this is going to include almost every file read since processes are run as root (except for a small set of processes like FRR, dbus, redis, chrony, etc that are run as a non-root user). On KVM, when booting up with this config, there was a very noticeable performance impact; on physical devices, this performance impact will very likely be more noticeable.

A policy that might be better as a starting point is the following:

```
dont_measure fsmagic=0x9fa0          # PROC_SUPER_MAGIC
dont_measure fsmagic=0x62656572      # SYSFS_MAGIC
dont_measure fsmagic=0x64626720      # DEBUGFS_MAGIC
dont_measure fsmagic=0x1021994       # TMPFS_MAGIC
dont_measure fsmagic=0x1cd1          # DEVPTS_SUPER_MAGIC
dont_measure fsmagic=0x42494e4d      # BINFMTFS_MAGIC
dont_measure fsmagic=0x73636673      # SECURITYFS_MAGIC
dont_measure fsmagic=0xf97cff8c      # SELINUX_MAGIC
dont_measure fsmagic=0x43415d53      # SMACK_MAGIC
dont_measure fsmagic=0x27e0eb        # CGROUP_SUPER_MAGIC
dont_measure fsmagic=0x63677270      # CGROUP2_SUPER_MAGIC
dont_measure fsmagic=0x6e736673      # NSFS_MAGIC
dont_measure fsmagic=0xde5e81e4      # EFIVARFS_MAGIC
measure func=MMAP_CHECK mask=MAY_EXEC
measure func=BPRM_CHECK mask=MAY_EXEC           # binary executed
measure func=MODULE_CHECK
measure func=FIRMWARE_CHECK
measure func=POLICY_CHECK
```

This is the same as `ima_policy=tcb`, but doesn't cover regular files that are opened for reading or reading/writing. This _should_ give slightly better performance.

If dracut is being used (as suggested), then the integrity module of dracut supports loading an IMA policy in the initrd phase if `/etc/sysconfig/ima` is present. It also supports loading X.509 certs for use by IMA (which will be covered in trusted boot).

# References

* https://www.gnu.org/software/grub/manual/grub/html_node/Measured-Boot.html
* https://ima-doc.readthedocs.io/en/latest/ima-policy.html
