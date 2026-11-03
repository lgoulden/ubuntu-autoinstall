# Ubuntu autoinstall files

A collection of [Ubuntu autoinstall](https://canonical-subiquity.readthedocs-hosted.com/en/latest/intro-to-autoinstall.html) configs for unattended installs, organised by Ubuntu release.

> ⚠️ These configs wipe disks without asking. Read the README for a config before using it, and test in a VM first.

## Configs

| Release | Config | Description |
|---|---|---|
| 26.04 | [cis-workstation-l2](26.04-desktop/cis-workstation-l2/) | Desktop, CIS L2 partition layout on LUKS2 LVM, `sysadmin` + `user` accounts |

Each config folder has its own `README.md` covering what it does, the placeholders to replace, and how to put it on install media.

## Generating password hashes

Autoinstall expects password **hashes**, not plain text. Generate a SHA-512 hash with either:

```bash
mkpasswd -m sha-512
```

```bash
openssl passwd -6
```

(`mkpasswd` is in the `whois` package.) Paste the output into the config inside **single quotes** so the `$` characters are kept as-is, e.g. `password: '$6$...'`.
