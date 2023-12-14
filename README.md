# Ansible Arch Installation Script

To automate my Archlinux setup I created 2 Ansible playbooks:

1. [arch-base](https://github.com/mabq/arch-base) (this repo) - fully automates a [basic Archlinux installation](https://wiki.archlinux.org/title/Installation_guide).
2. [arch-setup](https://github.com/mabq/arch-setup) installs and configures all the tools I need.


## Why two different scrips?

This playbook is meant to be used only once, when installing Arch --- since [ansible](https://archlinux.org/packages/extra/any/ansible/) is not included in the [live environment](https://wiki.archlinux.org/title/Installation_guide#Boot_the_live_environment) this playbook must be executed from a [controller](https://docs.ansible.com/ansible/latest/getting_started/index.html#getting-started-with-ansible) machine.

The second playbook is meant to be executed all the time from the same machine via `ansible-pull`.


## Why not use the `archinstall` script instead?

It fails with disk encryption enabled, but more importantly with Ansible you are not limited to the predefined set of options offered by the script, you can literally do whatever you want, like setting up LVM, adjust the swap size, etc.


## About this playbook

> To customize options see notes on `group_vars/all.yml`.

> Some tasks in this playbook use the `ansible.builtin.command` module instead of the specialized module for the given task, this is because the specialized module would affect the Live Environment, not the actual installation. For example, the `ansible.builtin.user` module would create the user in the Live Environment instead of the actual installation. 

This playbook does the following:

- Only runs on systems booted from the Archlinux ISO.
- Detects the firmware type and creates disk partitions accordingly. 
    - Encrypts the main partition with LUKS.
    - Creates an LVM volume group with two logical volumes, one for `/` and one for `/home`.
    - Enables TRIM support if supported by the disk.
- Installs only basic packages plus a few required to run the `arch-setup` playbook later on.
- Does not configure the system, configuration is done on the `arch-setup` playbook.
- Creates the user account giving it sudo privileges.
    - Stores the encryption key in a file called `.vault_key` in the user's home directory.
- Enables or disables the root account, as instructed.
- Enables SWAP, as instructed.


## Before running the script

1. On the **managed node** (where you want to install Archlinux):

   - Boot from the [installation image](https://archlinux.org/download/).
   
   - Change the password of the root user:
   
     ```bash
     # Use a simple password, its only temporary.
     passwd
     ```
   
   - Annotate the IP address:
   
     ```bash
     # Use `iwctl` if you need to connect to a wireless network.
     ip a
     ```
   
2. On the **controller node**:

   - Clone this repository:
   
     ```bash
     git clone git@github.com:mabq/arch-base.git
     ``` 
   
   - Review the `hosts.ini` file --- make sure the hostname you intend to assign to the managed node is listed there.
   
   - Make sure there is a file matching that hostname in `host_vars/{HOSTNAME}.yml` --- the variable `ansible_host` must be pointing to the correct IP address (the one you just checked a couple of steps above).

   - Read the `group_vars/all.yml` file, you may need to overwrite one or more of the default variables, pay special attention to the following variables: 

     - `target_disk` --- use `fdisk -l` on the managed node to identify the [block device name](https://wiki.archlinux.org/title/Device_file#Block_devices).

     - `password` --- is the password for the user account (see below how to encrypt a variable value).

     - `encryption_password` --- is the password used to encrypt the disk, make sure it is long and random.

   - Run the script:

     Change directory into the cloned repository and run:

     ```bash
     ansible-playbook --extra-vars "username={USERNAME}" local.yml -k --vault-password-file ~/.vault_key
     ```

     You will be prompted for the root password of the managed node (the one you changed recently). If no errors occur the managed node will shutdown automatically after a successful installation.

     Remove install media and turn it back on.

     Use the `nmtui` command to connect to a wireless network.


## Encrypt a variable value with Ansible

To encrypt a variable with Ansible, first store the encryption password in a file to avoid any typos:

   ```bash
   echo "{encryption-password-here}" > ~/.vault_key; chmod 600 ~/.vault_key
   ```

Encrypt the variable value:

   ```bash
   ansible-vault encrypt_string '{TEXT_TO_ENCRYPT}' --vault-password-file ~/.vault_key --name '{VARIABLE_NAME}'`
   ```

Copy the encrypted output and paste it in `/group_vars/all.yml` or in the corresponding `host_vars/{HOSTNAME}.yml`.


