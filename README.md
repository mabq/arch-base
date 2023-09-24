# Basic Archlinux installation with Ansible

I like Archlinux a lot but I don't want to manually redo everything time and again. To fully automate my setup I created 2 ansible scripts:

1. [ansible-archlinux](https://github.com/mabq/ansible-archlinux) (this repo) - fully automates a [basic Archlinux installation](https://wiki.archlinux.org/title/Installation_guide), leaving the host ready to run the second script.
2. [ansible-setup](https://github.com/mabq/ansible-setup) installs and configures all the tools I need.

##### Why two different scrips?

[Ansible](https://archlinux.org/packages/extra/any/ansible/) is not included in the Archlinux [installation image](https://archlinux.org/download/), so this playbook (`local.yml`) must be executed from a [controller node](https://docs.ansible.com/ansible/latest/getting_started/index.html#getting-started-with-ansible), on the other hand the playbook in [ansible-setup](https://github.com/mabq/ansible-setup) can be executed locally with `ansible-pull` (more details on that repo).


## About this script

   - Executes only if the managed node was booted from the Arch installation image (avoid accidental executions).
   - Securly erase the disk before installation (`false` by default, if enabled the playbook will take several hours to complete even on SSDs).
   - Detects firmware type and creates partitions accordingly.
   - Encrypts the root partition using [LUKS](https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#LVM_on_LUKS) and configures two [LVM](https://wiki.archlinux.org/title/LVM) logical volumes (on top of it), one for `/` (32gb by default) and one for `/home` (remaining space).
   - Installs the following packages:
     - System packages: `base`, `linux`, `linux-lts`, `linux-firmware`, `lvm2`, `grub` and `efibootmgr` (UEFI only).
     - Basic utilities: `networkmanager`, `openssh`, `neovim`, `tmux` and `ansible` (leaving the host ready to run [ansible-setup](https://github.com/mabq/ansible-setup)).
     - Microcode updates: `amd-ucode` or `intel-ucode`.
   - Performs basic configurations:
     - Detects disk type and enables TRIM if supported.
     - Enables `sshd` and `NetworkManager` services.
     - Sets the console keymap (`us` by default).
     - Generates locales (`en_US` and `es_EC` by default).
     - Sets the `hostname` (with the name assigned in `/hosts`).
     - Generate the initial RAM filesystem for both kernels (`linux` and `linux-lts`). Automatically adds a key file so you won't need to type the disk encryption password during the boot process.
     - Configures grub as the boot loader.
     - Optionally sets up a swap file (4096mb by default).


## Before running the script

- Change the password of the root user in the managed node:

    ```bash
    # use a simple password, its only temporary
    passwd
    ```

- Annotate the IP address of the managed node:

    ```bash
    ip a
    ```

    Use `iwctl` if you need to connect to a wireless network.

- Clone this repo on the controller node:

   ```bash
   git clone git@github.com:mabq/ansible-archlinux.git
   ``` 

- Review `/hosts`:

    Make sure the name of the host you intend to affect is listed on the `/hosts` file.

    Make sure the variable `ansible-host` in `/host_vars/{hostname}.yml` is pointing to the right ip address.

- Change the value of the `password` variable in `/group_vars/all.yml` (if needed):

    That password will be used for disk encryption and for the root user --- make sure you use a long random password, you will almost never need to type it manually and it must be secure.

    Because this repo is public you must encrypt the value of the password with an encryption password --- also make sure you use a long, random encryption password. To avoid mistyping the password store it on a file with the following command:

    ```bash
    echo "{encryption-password}" > ~/.vault_key; chmod 600 ~/.vault_key
    ```

    Encrypt the password:

    ```bash
    ansible-vault encrypt_string '{password}' --vault-password-file ~/.vault_key --name 'password'`
    ```

    Copy the output and replace the `password` variable value in `/group_vars/all.yml`.

- Check default variables in `/group_vars/all.yml`:

    > Do not change the default values in the `/group_vars/all.yml` file, overwrite them by including them on each `/host_vars/{hostname}.yml` file.

    Specially check `target_disk`, you may be using a nvme disk. Use `fdisk -l` to identify the target disk.


## Run the script

Change directory into the cloned repo and run:

```bash
ansible-playbook local.yml -k --vault-password-file ~/.vault_key
```

You will be prompted for the root password of the managed node (the one you just changed).

If no erros occur the managed node will shutdown automatically after install. Remove install media and turn it back on.

Use the `nmtui` command to connect to a wireless network.

**Note**: You won't be able to login via SSH yet because password login for the root user is disabled by default.

