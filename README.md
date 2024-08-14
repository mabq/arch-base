# Archlinux base installation with Ansible

To automate my Archlinux setup I created 2 ansible playbooks:

1. [Arch-base](https://github.com/mabq/arch-base) (this repo) - fully automates a [basic Archlinux installation](https://wiki.archlinux.org/title/Installation_guide).
2. [Arch-setup](https://github.com/mabq/arch-setup) installs and configures everything.


## Why two different scrips?

This playbook is meant to be used only once, when installing Arch, from a controller machine.

The `arch-setup` playbook is meant to be executed all the time from the same machine via `ansible-pull`.


## What do you need to run this playbook?

  - A second computer with [Ansible](https://archlinux.org/packages/extra/any/ansible/) installed -- Ansible is not included in the Archlinux [live environment](https://wiki.archlinux.org/title/Installation_guide#Boot_the_live_environment), so this playbook must be executed from a [controller](https://docs.ansible.com/ansible/latest/getting_started/index.html#getting-started-with-ansible) node.

  - The encryption password.


## Why not use the `archinstall` script instead?

It fails with disk encryption enabled, but more importantly with Ansible you are not limited to the predefined set of options offered by the script, you can literally do whatever you want, like setting up LVM, adjust the swap size, etc.


## About this playbook

> I decided not to use LVM. Logical volume data remained even after deleting the partitions, forcing me to securely wipe the disk before running this playbook. For my workstations I really don't need LVM anyway. Check the `local_lvm` playbook if you ever need to implement that again.

  - Only runs on systems booted from the Archlinux ISO.
  - Detects the firmware type and creates disk partitions accordingly.
      - Encrypts the main partition with LUKS.
      - Enables TRIM support if supported by disk.
  - Installs only basic packages plus a few required packages to run the `arch-setup` playbook later on.
  - Does not configure the system, configuration is done on the `arch-setup` playbook.
  - Creates the user account, giving it sudo privileges.
      - Generates the `.vault_key` file on the user's home directory. This file will be used by the `arch-setup` script.
  - Enables/disables the root account, as instructed.
  - Sets up the initial ram filesystem.
  - Configures GRUB as the bootloader.
  - Enables SWAP, as instructed.


## How to run this playbook?

1. Download the Arch ISO.

   - Head to [Arch Linux's official website](https://www.archlinux.org/download/) and download the Arch ISO image.

   - Execute `cp -v {path/to/ISO} /dev/{disk}` to copy it to your USB drive.

2. On the **managed node** (the computer where Archlinux will be installed):

   - Connect the USB drive and boot from the Arch Linux ISO.

   - Connect to the internet. A wired connection is preferred since it's easier to connect. [More info](https://wiki.archlinux.org/index.php/Installation_guide#Connect_to_the_internet).

   - Execute `passwd` to change the root password -- use `1`.

   - Execute `ip a` and annotate the IP address.

   - Execute `lsblk` or `fdisk --list` to identify the target disk.

   - Optionally, [securely wipe disk](https://wiki.archlinux.org/title/Securely_wipe_disk).


3. On the **controller node**:

   - Clone this repository (if already cloned, make sure to commit all changes):

     ```bash
     git clone git@github.com:mabq/arch-base.git
     ```

   - Review the `hosts.ini` file. Make sure the hostname of the managed node you intend to affect is listed there.

   - Default options for all hosts are defined in `group_vars/all.yml`. You can overwrite any options for each host on `host_vars/{HOSTNAME}.yml` file. Make sure the following variables are correct for each host:

     - `ansible_host` must be pointing to the correct IP address.
     - `installation_block_device_name` must be pointing to the correct [block device name](https://wiki.archlinux.org/title/Device_file#Block_devices).

   - Change directory into the cloned repository and run the playbook:

     > If you don't yet have the `~/.vault_key` file on the controller machine, create one with the instructions below.

     ```bash
     # replace {USERNAME} with the username you want
     ansible-playbook --extra-vars "username={USERNAME}" --vault-password-file ~/.vault_key --ask-pass local.yml
     ```

     When prompted, enter the password for the root user of the managed node (the one you just setup).

   - Wait for the playbook to execute, if no errors occur the managed node will shutdown automatically after a successful installation.

   - Remove install media and turn it back on.

   - Use the `nmtui` command to connect to a wireless network.


## How to encrypt a variable value with Ansible

First, create the `~/.vault_key` file with the encryption password to avoid any typos:

   ```bash
   echo "{encryption-password-here}" > ~/.vault_key; chmod 600 ~/.vault_key
   ```

Then, you can create encrypted the variables with the following command:

   ```bash
   ansible-vault encrypt_string '{TEXT_TO_ENCRYPT}' --vault-password-file ~/.vault_key --name '{VARIABLE_NAME}'`
   ```

Copy the encrypted output and paste it in `/group_vars/all.yml` or in the corresponding `host_vars/{HOSTNAME}.yml`.
