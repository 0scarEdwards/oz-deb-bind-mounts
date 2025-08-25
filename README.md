# Oz Debian Profile Migration Wizard

The **Oz Debian Profile Migration Wizard** is a Bash utility for Linux workstations that are moving between Active Directory domains.  When a computer leaves an old domain and joins a new one, the existing user home directories under `/home` remain on disk, but the new domain accounts cannot see them by default.  This script discovers those legacy domain accounts, grants the corresponding new domain users access to their old data, and transparently mounts the old home directories into the new profiles.  Think of it as a *secret door* between your old house and your new one: you can still access your Documents and Downloads while fully inhabiting the new domain.

## Overview

After joining a new domain and logging in once as the new domain user, you can run this helper to migrate profiles.  The wizard performs three main tasks:

1. **Discovery of legacy accounts** – it scans `/home` for directories of the form `/home/OLD.DOMAIN/user` and attempts to locate the matching user on the new domain.  The script uses system tools to resolve usernames, supporting both `user@domain` and `DOMAIN\user` formats.
2. **Permission granting** – it applies POSIX Access Control Lists (ACLs) on the old home directory so that the new domain user can read and modify every file.  ACLs are applied recursively and set as defaults on directories so that newly created files inherit the correct permissions.
3. **Bind mounting** – it mounts the old home directory onto the new one using `mount --bind` and appends a corresponding entry to `/etc/fstab` so the mount persists across reboots.  Applications see the old Documents and Downloads folders as if they lived in the new profile, and changes are reflected both ways.

The script does **not** delete or modify the original homes.  Instead, it overlays them in the new location and leaves the old data intact.  If anything goes wrong, you can remove the entry from `/etc/fstab` and unmount the bind point to return to the previous state.

### A Quick Primer on the Linux Filesystem

Linux organizes data in a hierarchical tree.  According to the [Filesystem Hierarchy Standard](https://en.wikipedia.org/wiki/Filesystem_Hierarchy_Standard), all files and directories appear beneath the root directory `/`【270118462581305†L168-L181】.  Some important subdirectories include:

| Directory | Purpose |
|----------|---------|
|`/etc`|Host‑specific configuration files.|
|`/bin` and `/usr/bin`|Essential user commands and utilities.|
|`/home`|User home directories that store personal files and settings.|
|`/root`|The superuser’s home directory.|

Regular users spend their time in `/home/<username>`, which is represented by the `~` shorthand.  Each home directory is a personal space containing the user’s files, configuration and environment.  The wizard works by temporarily giving a new domain user access to their old home and then binding the old home into the new path.

### What Is a Bind Mount?

A bind mount is a Linux kernel feature that allows you to mount one directory onto another location in the same filesystem.  When you run `mount --bind /old/path /new/path`, both paths refer to the same directory; creating, removing or modifying files in either location immediately affects the other.  The Baeldung guide explains that a bind mount acts like an alias: when we bind mount the directory `/tmp/foo` on `/tmp/bar`, *both will reference the same content* and you can access the files in `/tmp/foo` from `/tmp/bar`【950781406456951†L88-L91】.  This mechanism is what allows the wizard to make your old Documents and Downloads directories appear seamlessly inside your new domain profile.

### Why Not Just Use Symlinks?

Symbolic links (`ln -s`) point to another path at the filesystem level.  They are fine for many scenarios, but there are important reasons to prefer bind mounts in enterprise environments:

- **Chroot and container boundaries** – inside a chroot or container environment, a symlink whose target lies outside the environment becomes dead.  A bind mount remains accessible.
- **Program behaviour** – many utilities treat symbolic links and real directories differently; they may copy the link rather than the target or follow a symlink chain incorrectly.  Few programs can distinguish a bind‑mounted directory from a regular directory.
- **Network shares and security policies** – file servers such as Samba disable following symlinks outside a share by default.  The Samba documentation notes that the `wide links` option, which allows the server to follow symlinks outside a share, is automatically disabled when UNIX extensions are enabled “to prevent UNIX clients creating symlinks to areas of the server file system that the administrator does not wish to export”【75777632841461†L1173-L1177】.  Enabling wide links is considered insecure and is often prohibited in corporate policy.
- **Privilege requirements on Windows clients** – in mixed environments with Windows clients, creating symlinks requires the `SeCreateSymbolicLinkPrivilege` and is restricted to administrators.  Microsoft warns that only trusted users should receive this right because symbolic links can expose vulnerabilities and the default configuration grants this privilege exclusively to administrators【52874551376956†L153-L186】.

Bind mounts avoid these pitfalls because they are implemented in the kernel and behave exactly like the original directory.  Once configured in `/etc/fstab`, they persist across reboots and do not depend on client permissions.

## Features

- **Auto‑discovery of domain accounts** – scans `/home` for `DOMAIN/user` directories and matches them with new domain users.
- **Granular permission grants** – uses `setfacl` to grant full access to the old home directory without disturbing existing permissions.
- **Persistent bind mounts** – configures a bind mount for each old home so the old Documents and Downloads appear in the new profile.  Entries are added to `/etc/fstab` for persistence.
- **Transparent overlay** – applications see the old data inside the new profile, and any changes are reflected both ways.
- **Comprehensive logging** – all actions are logged with timestamps to `/var/log/oz-domain-home-migrate.log`.  You can follow the log for troubleshooting or audit purposes.
- **Safety first** – the script checks for root privileges, backs up `/etc/fstab` before modifying it, and stops on errors to avoid damaging your system.
- **Self‑contained** – no external dependencies beyond the standard `acl` package.  The script will attempt to install the package for you if it isn’t present.

## Requirements

- **Debian‑based system** – the wizard is tested on Debian but may work on Ubuntu or other derivatives.
- **Root privileges (mandatory)** – many operations such as editing ACLs and modifying `/etc/fstab` require administrative rights.  Run the script with `sudo`.
- **Joined to the new domain** – the machine must already be joined to the new Active Directory domain, and the new domain users must have logged in once (to create their home directories).
- **`acl` package** – required for `setfacl`.  The script will prompt to install it if missing.

## Installation

1. Download the script to your system.
2. Make it executable:
   ```bash
   chmod +x oz-deb-profile-migrate.sh
   ```
3. **Run as root (required)**:
   ```bash
   sudo ./oz-deb-profile-migrate.sh
   ```
   The script will automatically check for root privileges and exit if not run with `sudo`.

## Usage

### Profile Migration

Running the wizard without arguments performs a standard profile migration:

```bash
sudo ./oz-deb-profile-migrate.sh
```

The script will:

1. Scan `/home` for legacy domain accounts.
2. Prompt you to confirm each matched user before proceeding.
3. Apply ACLs to grant the new user access to their old files.
4. Create a bind mount from the old home to the new one and update `/etc/fstab`.

After completion, log out and log back in as the new user to see the old data seamlessly integrated into your profile.

### Audit Mode

```bash
sudo ./oz-deb-profile-migrate.sh --audit
```

Runs through the discovery process and reports what would be changed without modifying anything.  Use this to preview migrations before committing.

### Help

```bash
sudo ./oz-deb-profile-migrate.sh --help
```

Displays usage information and available options.

## Project Structure

```
Oz-Profile-Migrate/
├── oz-deb-profile-migrate.sh  # Main migration script
├── README.md                  # This documentation file
├── EXAMPLE-DEMO.md            # Example demonstration and troubleshooting guide
└── LICENSE                    # MIT License
```

## Code Analysis

### File Breakdown

- **Main Script**: `oz-deb-profile-migrate.sh` – responsible for discovery, permission adjustment and bind mounting.  It contains approximately 500 lines of code and comments.
- **Documentation**: `README.md`, `EXAMPLE-DEMO.md`, `LICENSE` – supporting documents and licensing information.

### Function Breakdown

Below are some of the key functions in the script:

- `main()`: Entry point and overall script flow control.
- `check_root()`: Verifies that the script is run with root privileges.
- `discover_legacy_accounts()`: Scans `/home` for directories matching the old domain format and resolves them to new domain users.
- `grant_permissions()`: Uses `setfacl` to grant the new user full access to the old home directory.
- `create_bind_mount()`: Performs the `mount --bind` operation and appends a persistent entry to `/etc/fstab`.
- `install_acl_package()`: Installs the `acl` package if it is missing.
- `log_message()`: Handles structured logging to `/var/log/oz-domain-home-migrate.log`.
- `handle_failure()`: Provides error handling and rollback in case of issues.

### Key Features Explained

#### Automated Account Discovery

The script identifies legacy domain user home directories by scanning `/home` for subdirectories that match the pattern `DOMAIN/username`.  It then queries the system’s name service switch to resolve the corresponding new domain user.  Both `user@domain` and `DOMAIN\user` naming conventions are supported.

#### Permission Granting with ACLs

POSIX ACLs allow fine‑grained permissions beyond the standard Unix owner/group/other model.  The wizard uses `setfacl -R -m` to apply read, write and execute permissions to every file and directory in the old home for the new user.  Default ACLs are applied on directories so that newly created files inherit the correct permissions.

#### Bind Mount Mechanism

Mounting the old home directory onto the new one with `mount --bind` makes both paths reference the same underlying files【950781406456951†L88-L91】.  Adding the mount to `/etc/fstab` ensures that the binding persists across reboots.  Because the underlying files are still located in the old domain path, no data is moved or copied; you can remove the mount at any time without data loss.

#### Safety and Logging

The script checks for root privileges and exits if they are not available.  Before modifying `/etc/fstab`, it creates a timestamped backup in `/etc/fstab.backup-<date>`.  If any command fails, the wizard logs the error, stops further actions and offers guidance.  All actions are logged with timestamps to `/var/log/oz-domain-home-migrate.log` for auditing and troubleshooting.

## Safety Features

- **Root Privilege Verification** – the script refuses to run unless executed as root.
- **Backup of Critical Files** – `/etc/fstab` is backed up before modification.
- **Comprehensive Logging** – all operations and potential errors are written to a log file.
- **Non-Destructive Operation** – data is never deleted; mount points can be removed to revert to the original state.
- **Audit Mode** – run the script with `--audit` to preview changes without modifying anything.

## Troubleshooting

If you encounter issues:

1. Check the log file in `/var/log/oz-domain-home-migrate.log` for detailed output.
2. Verify that the machine is joined to the correct domain and that the new user accounts have logged in at least once.
3. Ensure the `acl` package is installed; the script will prompt to install it if missing.
4. If a bind mount fails to mount on reboot, review `/etc/fstab` for correctness and consult the system journal (`journalctl -xe`).
5. To revert, remove the relevant entry from `/etc/fstab` and run `sudo umount /home/NEWDOMAIN/user`.

## License

This project is licensed under the MIT License – see the `LICENSE` file for details.

## Support

For issues and questions:
1. Check the log file at `/var/log/oz-domain-home-migrate.log`.
2. Review `EXAMPLE-DEMO.md` for usage examples and troubleshooting guidance.
3. Ensure all requirements are met before running the script.

The script includes comprehensive error handling and will provide specific guidance based on the type of failure encountered.

---

<!--
⢀⣠⣾⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⠀⠀⠀⠀⣠⣤⣶⣶
⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⠀⠀⠀⢰⣿⣿⣿⣿
⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣧⣀⣀⣾⣿⣿⣿⣿
⣿⣿⣿⣿⣿⡏⠉⠛⢿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⡿⣿
⣿⣿⣿⣿⣿⣿⠀⠀⠀⠈⠛⢿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⠿⠛⠉⠁⠀⣿
⣿⣿⣿⣿⣿⣿⣧⡀⠀⠀⠀⠀⠙⠿⠿⠿⠻⠿⠿⠟⠿⠛⠉⠀⠀⠀⠀⠀⣸⣿
⣿⣿⣿⣿⣿⣿⣿⣷⣄⠀⡀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢀⣴⣿⣿
⣿⣿⣿⣿⣿⣿⣿⣿⣿⠏⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠠⣴⣿⣿⣿⣿
⣿⣿⣿⣿⣿⣿⣿⣿⡟⠀⠀⢰⣹⡆⠀⠀⠀⠀⠀⠀⣭⣷⠀⠀⠀⠸⣿⣿⣿⣿
⣿⣿⣿⣿⣿⣿⣿⣿⠃⠀⠀⠈⠉⠀⠀⠤⠄⠀⠀⠀⠉⠁⠀⠀⠀⠀⢿⣿⣿⣿
⣿⣿⣿⣿⣿⣿⣿⣿⢾⣿⣷⠀⠀⠀⠀⡠⠤⢄⠀⠀⠀⠠⣿⣿⣷⠀⢸⣿⣿⣿
⣿⣿⣿⣿⣿⣿⣿⣿⡀⠉⠀⠀⠀⠀⠀⢄⠀⢀⠀⠀⠀⠀⠉⠉⠁⠀⠀⣿⣿⣿
⣿⣿⣿⣿⣿⣿⣿⣿⣧⠀⠀⠀⠀⠀⠀⠀⠈⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢹⣿⣿
⣿⣿⣿⣿⣿⣿⣿⣿⣿⠃⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢸⣿⣿
Coded By Oscar
-->
