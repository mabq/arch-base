# Archlinux base installation with Ansible

To automate my Archlinux setup I created 2 ansible playbooks:

1. [arch-base](https://github.com/mabq/arch-base) (this repo) - fully automates a [basic Archlinux installation](https://wiki.archlinux.org/title/Installation_guide)
2. [arch-setup](https://github.com/mabq/arch-setup) installs and configures all the tools I need


## Why two different scrips?

This playbook is meant to be used only once, when installing Arch --- since [ansible](https://archlinux.org/packages/extra/any/ansible/) is not included in the [live environment](https://wiki.archlinux.org/title/Installation_guide#Boot_the_live_environment) this playbook must be executed from a [controller](https://docs.ansible.com/ansible/latest/getting_started/index.html#getting-started-with-ansible) machine.

The second playbook is meant to be executed all the time from the same machine via `ansible-pull`.


## Why not use the `archinstall` script instead?

It fails with disk encryption enabled, but more importantly with Ansible you are not limited to the predefined set of options offered by the script, you can literally do whatever you want, like setting up LVM, adjust the swap size, etc.


## About this playbook

  - Only runs on systems booted from the Archlinux ISO.
  - Detects the firmware type and creates disk partitions accordingly.
      - Encrypts the main partition with LUKS.
      - Sets up LVM on the main partition with two logical volumes; one for `/` and one for `/home`.
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

   - Optionally, securely erase disk data:

     > This step is mandatory if the disk was previously setup with LVM.

     ```bash
     # For SSDs (super fast):
     hdparm --user-master u --security-set-pass p /dev/sd{X}
     hdparm --user-master u --security-erase p /dev/sd{X}

     # For spinning drives (super slow):
     sudo dd if=/dev/zero of=/dev/sd{X} bs=4M status=progress
     ```

     > Reboot and use `lsblk` to make sure disk has been wiped.

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

   - Review the `hosts.ini` file --- make sure the hostname of the managed node you intend to affect is listed there.

   - Make sure there is a configuration file for the managed node you intend to affect. Default options for all hosts are defined in `group_vars/all.yml`, specific host options are defined in `host_vars/{HOSTNAME}.yml`.

   - Make sure the following variables are correct on the host options file:

     - `ansible_host` must be pointing to the correct IP address.
     - `installation_block_device_name` must be pointing to the correct [block device name](https://wiki.archlinux.org/title/Device_file#Block_devices).

   - Change directory into the cloned repository and run the playbook:

     > If you don't yet have the `~/.vault_key` file on the controller machine, create one with the instructions below.

     ```bash
     # replace {USERNAME} with the username you want
     ansible-playbook --extra-vars "username={USERNAME}" --vault-password-file ~/.vault_key --ask-pass local.yml
     ```

     When prompted, enter the password for the root user of the managed node (the one you just setup).

     Wait for the playbook to execute, if no errors occur the managed node will shutdown automatically after a successful installation.

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
