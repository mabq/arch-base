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

> I decided not to use LVM. A single encrypted partition is all I need for my workstation. Use the `local_lvm.yml` playbook to setup LVM.

  - Only runs on systems booted from the Archlinux ISO.
  - Detects the firmware type and creates disk partitions accordingly.
      - Encrypts the main partition with LUKS.
      <!-- - Sets up LVM on the main partition with two logical volumes; one for `/` and one for `/home`. -->
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

1. On the **managed node** (where you want to install Archlinux):

   - Boot from the [installation image](https://archlinux.org/download/). Visit the [installation guide](https://wiki.archlinux.org/title/Installation_guide) for more information.

   - Run the `lsblk` command to identify the target disk for the installation.

   - Optionally, securely erase all data on disk:

     > This is recommended to avoid any errors, on SSD if super fast, on normal hard drives it will take a while.

     ```bash
     # SSDs:
     hdparm --user-master u --security-set-pass p /dev/sd{X}
     hdparm --user-master u --security-erase p /dev/sd{X}

     # Normal HDD:
     sudo dd if=/dev/zero of=/dev/sd{X} bs=4M status=progress
     ```

   - Change the root password:

     ```bash
     passwd
     # Use a simple password, its only temporary.
     ```

   - Annotate the IP address:

     ```bash
     # Use `iwctl` if you need to connect to a wireless network.
     ip a
     ```

2. On the **controller node**:

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


## Correct MacBook flickering issue

For laptops with old nvidia cards do the following (more info [here](https://ubuntuforums.org/showthread.php?t=2498385)):

- Edit the GRUB config file:

  ```bash
  # open the file
  sudo nvim /etc/default/grub
  ```
 
  Make sure the following variable contains the following option (there will be other values):

  ```bash
  # grub will pass this parameter to the Linux kernel during boot, the noveau
  # driver is part of the linux kernel.
  GRUB_CMDLINE_LINUX_DEFAULT="nouveau.NvMXMDCB=0"
  ```

  Save and close the file.

- Re-build grub:

  ```bash
  sudo grub-mkconfig -o /boot/grub/grub.cfg
  ```

- Reboot

  You can check what driver is controlling each device with `lspci -k`.
