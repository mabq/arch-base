# Basic Archlinux installation with Ansible

I like Archlinux a lot but I don't like to manually install it every time. To automate my setup I created 2 Ansible scripts:

1. [ansible-arch-installation](https://github.com/mabq/ansible-arch-installation) (this repo) - fully automates a [basic Archlinux installation](https://wiki.archlinux.org/title/Installation_guide), leaving the host ready to run the second script.
2. [ansible-post-installation](https://github.com/mabq/ansible-post-installation) installs and configures all the tools I need.


## Why two different scrips?

[Ansible](https://archlinux.org/packages/extra/any/ansible/) is not included in the Archlinux [installation image](https://archlinux.org/download/), so this playbook (`local.yml`) must be executed from a [controller](https://docs.ansible.com/ansible/latest/getting_started/index.html#getting-started-with-ansible) machine. The second script can be executed locally with `ansible-pull`.


## Why not the `archinstall` script 

Most of the times I tried to run the script with disk encryption enabled, it fails. More importantly, with Ansible you are not limited to the predefined set of options offered by the script, you can literally do whatever you want, like setting up LVM, adjust the swap size, etc.

The second script should work regardless of the installation method.


## About this script

   - Executes only if the managed node was booted from the Arch installation image (avoid accidental executions).
   - Securely erase the disk before installation (`false` by default, if enabled the playbook will take several hours to complete even on SSDs).
   - Detects the firmware type and creates partitions accordingly.
   - Encrypts the root partition using [LUKS](https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#LVM_on_LUKS) and configures two [LVM](https://wiki.archlinux.org/title/LVM) logical volumes (on top of it), one for root `/` (32gb by default) and one for `/home` (remaining disk space).
   - Installs only basic packages and Ansible.
   - Performs basic configurations:
     - Detects disk type and enables TRIM if supported.
     - Enables `sshd` and `NetworkManager` services.
     - Sets the console keymap.
     - Configures the locales.
     - Sets the hostname.
     - Generates the initial RAM file system for both kernels (`linux` and `linux-lts`). Automatically adds a key file so you won't need to type the disk encryption password during the boot process.
     - Configures GRUB as the boot loader.
     - Optionally sets up a swap file.
     - Creates the user account (the username must be passed as a command-line argument)
     - Disables root login (by default. When set to false, the encryption password will be used as root password)

> Some actions in this script use the `ansible.builtin.command` module instead of the specialized module for that type of action, this is because the specialized module would affect the Live Environment, not the actual installation. For example, the `ansible.builtin.user` module would create the user in the Live Environment instead of the actual installation. 


## Before running the script

1. On the **managed node** (where you want to install Archlinux):

   - Boot the live environment.
   
   - Change the password of the root user:
   
     ```bash
     # use a simple password, its only temporary
     passwd
     ```
   
   - Check the IP address:
   
     ```bash
     ip a
     ```
   
     Use `iwctl` if you need to connect to a wireless network.

2. On the **controller node**:

   - Clone this repository:
   
     ```bash
     git clone git@github.com:mabq/ansible-arch-installation.git
     ``` 
   
   - Review the `hosts` file --- make sure the name of the host you intend to affect is listed.
   
   - Review the `host_vars/{HOSTNAME}.yml` file --- make sure the variable `ansible_host` is pointing to the correct IP address.

   - Check the variables defined in the `group_vars/all.yml` file, specially:

     - `target_disk` --- use `fdisk -l` on the managed node to identify the target disk block device name

     - `password` --- is the password for the user account (the username must be passed as a command-line argument), see below how to encrypt a variable value

     - `encryption_password` --- is the password used to encrypt the disk, make sure it is long and random (this password will also be used to automatically decrypt the contents of the second script)

### Encrypt a variable value with Ansible

To encrypt a variable with Ansible, first store the encryption password in a file to avoid any typos:

   ```bash
   echo "{encryption-password-here}" > ~/.vault_key; chmod 600 ~/.vault_key
   ```

Encrypt the variable value:

   ```bash
   ansible-vault encrypt_string '{TEXT_TO_ENCRYPT}' --vault-password-file ~/.vault_key --name '{VARIABLE_NAME}'`
   ```

Copy the output and replace the corresponding variable in `/group_vars/all.yml` or in the corresponding `host_vars/{HOSTNAME}.yml`.


## Run the script

Change directory into the cloned repository and run:

> the `username` variable is passed as a command-line argument for security reasons

   ```bash
   ansible-playbook --extra-vars "username={USERNAME}" local.yml -k --vault-password-file ~/.vault_key
   ```

You will be prompted for the root password of the managed node (the one you just changed).

If no errors occur the managed node will shutdown automatically after a successful installation.

Remove install media and turn it back on.

Use the `nmtui` command to connect to a wireless network.

