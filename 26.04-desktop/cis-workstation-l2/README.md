# Ubuntu 26.04 Desktop – CIS Workstation (L2 layout)

[`autoinstall.yml`](autoinstall.yml) does an unattended Ubuntu 26.04 Desktop install with only:

1. the **CIS Level 2 partition layout** on LUKS2-encrypted LVM, with CIS mount options
2. **two accounts**: `sysadmin` (sudo) and `user` (no sudo)

Nothing else is configured. All hardening (CIS L1 or L2) and all software is applied afterwards with Ansible.

> ⚠️ This **wipes the largest disk in the machine without asking.**

## Install settings

| Setting | Value |
|---|---|
| Source | `ubuntu-desktop-minimal` |
| Locale / keyboard / timezone | `en_GB.UTF-8` / `gb` / `Europe/London` |
| Updates during install | security only |
| Codecs, drivers, OEM, SSH server | not installed |
| Interactive sections | none (fully unattended), reboots when done |

## Replace the placeholders

| Placeholder | What to put there |
|---|---|
| `<SYSADMIN_PASSWORD_HASH>` | SHA-512 hash, see [generating password hashes](../../README.md#generating-password-hashes) |
| `<USER_PASSWORD_HASH>` | same |
| `<LUKS_PASSPHRASE>` | plain text, typed at every boot |
| `cis-ws01` | your hostname |

## Accounts

| Account | Purpose | sudo |
|---|---|---|
| `sysadmin` | Installing packages, admin config, running Ansible | yes |
| `user` | Daily use | no (member of `users` only) |

Both accounts appear on the login screen. Hiding `sysadmin`, restricting `su` and similar settings are for Ansible to do.

## Disk layout

| Volume | Size | Mount | Options |
|---|---|---|---|
| ESP | 1G | /boot/efi | `umask=0077` |
| p2 ext4 | 2G | /boot | `nodev,nosuid` (unencrypted, because GRUB can't read LUKS2) |
| p3 **LUKS2** → LVM `vg0` | rest | | |
| lv_root | 40G | / | |
| lv_var | 30G | /var | `nodev,nosuid` |
| lv_var_tmp | 5G | /var/tmp | `nodev,nosuid,noexec` |
| lv_var_log | 10G | /var/log | `nodev,nosuid,noexec` |
| lv_var_log_audit | 10G | /var/log/audit | `nodev,nosuid,noexec` |
| lv_tmp | 8G | /tmp | `nodev,nosuid,noexec` |
| lv_swap | 8G | swap | (encrypted, inside LUKS) |
| lv_home | **remaining** | /home | `nodev,nosuid` |
| tmpfs | – | /dev/shm | `nodev,nosuid,noexec` (fstab line) |

- This layout satisfies the partitioning requirements of both CIS L1 and L2, so the same install works for either Ansible profile.
- `lv_home` takes all remaining space. Give it a `size:` if you want free space left in the VG for growing other volumes later.
- If the installer rejects `lv_home` without a `size:`, set one explicitly (e.g. `150G`).
- Minimum disk size is roughly 114 GB (fixed volumes + boot partitions) plus whatever you want for `/home`.

## Putting it on install media

The desktop installer reads `autoinstall.yaml` from the root of the install media. To add it, rebuild the ISO, e.g. with `livefs-edit` (`pip install livefs-edit`):

```bash
sudo livefs-edit ubuntu-26.04-desktop-amd64.iso cis-ws.iso --cp "$PWD/autoinstall.yml" new/iso/autoinstall.yaml
```

## Test in a VM first

Use UEFI firmware (OVMF) and a 256 GB sparse disk. After install:

```bash
lsblk -f
findmnt -o TARGET,SOURCE,OPTIONS | grep -E '/(tmp|var|home|boot|dev/shm)'
sudo cryptsetup luksDump /dev/vda3 | head -3   # Version: 2
sudo -l -U user                                # "not allowed to run sudo"
```

## After install: first things for Ansible

- **Install Ansible itself:** log in as `sysadmin` and run `sudo apt install ansible git`, then `ansible-playbook -i localhost, -c local site.yml -K`.
- **Secrets on disk:** the installer copies its config, including the LUKS key and password hashes, into `/var/log/installer/`. Redact or delete those files, and change the LUKS passphrase: `cryptsetup luksChangeKey /dev/<disk>p3`.
- **`noexec /tmp` vs apt:** if package installs fail with debconf errors, set `APT::ExtractTemplates::TempDir` to a directory outside `/tmp`.
- **Install failures:** installer logs are copied to `/var/log/installer-error/` on the target disk if it got that far.
