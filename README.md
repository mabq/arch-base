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
      - Creates an LVM volume group with two logical volumes, one for `/` and one for `/home`.
      - Enables TRIM support if supported by disk.
  - Installs only basic packages plus a few required to run the `arch-setup` playbook later on.
  - Does not configure the system, configuration is done on the `arch-setup` playbook.
  - Creates the user account, giving it sudo privileges.
      - Creates the file `~/.vault_key` with the encryption key on it. This key is used by the `arch-setup` script.
  - Enables or disables the root account, as instructed.
  - Enables SWAP, as instructed.


## Before running the script

1. On the **managed node** (where you want to install Archlinux):

   - Boot from the [installation image](https://archlinux.org/download/). See [installation guide](https://wiki.archlinux.org/title/Installation_guide) for more info.

   - Once booted, change the root password:

     ```bash
     passwd
     # Use a simple password, its only temporary.
     ```

   - Annotate the IP address:

     ```bash
     # Use `iwctl` if you need to connect to a wireless network.
     ip a
     ```

   - Optionally, completely erase disk data:

     ```bash
     # Make sure you select the correct disk, use the `lsblk` command
     sudo dd if=/dev/zero of=/dev/sd{X} bs=4M status=progress
     ```

2. On the **controller node**:

   - Clone this repository:

     ```bash
     git clone git@github.com:mabq/arch-base.git
     ```

   - Review the `hosts.ini` file --- make sure the hostname of the managed node you intend to affect is listed there.

   - Default options for all hosts are defined in `group_vars/all.yml`. You can overwrite any variable for a given host on `host_vars/{HOSTNAME}.yml`:

     - Make sure the variable `ansible_host` is pointing to the correct IP address.

     - `target_disk` --- use `fdisk -l` on the managed node to identify the [block device name](https://wiki.archlinux.org/title/Device_file#Block_devices).

     - `password` --- is the password for the user account (see below how to encrypt a variable value).

     - `encryption_password` --- is the password used to encrypt the disk, make sure it is long and random.

   - Run the script:

     Change directory into the cloned repository and run:

     ```bash
     ansible-playbook --extra-vars "username={USERNAME}" --vault-password-file ~/.vault_key --ask-pass local.yml
     ```

     > If you don't have the `~/.vault_key` file yet on the controller machine, create one following the instructions below.

     > If you get any weird errors related to previous state of the target installation disk try completely erasing disk data as instructed above.

     > You will be prompted for the root password of the managed node (the one you changed recently). If no errors occur the managed node will shutdown automatically after a successful installation.

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
