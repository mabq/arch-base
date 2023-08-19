# Basic Archlinux installation with Ansible

I like Archlinux a lot but I don't want to manually redo everything time and again. To fully automate my setup I created 2 ansible scripts:

1. [ansible-archlinux](https://github.com/mabq/ansible-archlinux) (this repo) - fully automates a [basic Archlinux installation](https://wiki.archlinux.org/title/Installation_guide), leaving the host ready to run the second script.
2. [ansible-setup](https://github.com/mabq/ansible-setup) installs and configures all the tools I need.

#### Why two different scrips?

[Ansible](https://archlinux.org/packages/extra/any/ansible/) is not included in the Archlinux [installation image](https://archlinux.org/download/), so this playbook (`local.yml`) must be executed from a [controller node](https://docs.ansible.com/ansible/latest/getting_started/index.html#getting-started-with-ansible).

The playbook in [ansible-setup](https://github.com/mabq/ansible-setup) can be executed locally with `ansible-pull` (more details on that repo).


## About this script

   - Executes only if the managed node was booted from the Arch installation image (super important to avoid running it by accident and possibly lose all data on inventory hosts).
   - Detects firmware type and creates partitions accordingly.
   - Detects disk type and enables TRIM if supported.
   - Detects processor and installs microcode updates.
   - Sets up [LVM](https://wiki.archlinux.org/title/LVM) on top of [LUKS](https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#LVM_on_LUKS) for full disk encryption. Automatically generates a key file so you won't need to type the encryption password at boot).
   - Installs the latest and lts versions of linux kernels (in case you ever face any issues with the latest version)
   - Installs and enables opehssh and NetworkManager.
   - Installs tmux, neovim and ansible.
   - Performs basic configurations like time zone, locales, hostname, etc. (edit timezone and locales manually on `local.yml` if needed)
   - Sets up swap of the desired size (if desired, see `group_vars`)


## Before running the script

- Change the password of the root user in the live environment (managed node):

    ```bash
    passwd
    ```

    Use a simple password, its only temporary.

- Annotate the IP address of the live environment (managed node):

    ```bash
    ip a
    ```

- Clone this repo on the controller node:

   ```bash
   git clone git@github.com:mabq/ansible-archlinux.git
   ``` 

- Review `/hosts`:

    Make sure the name of the host you intend to affect is listed on the `/hosts` file.

    Make sure the variable `ansible-host` in `/host_vars/{hostname}.yml` is pointing to the right ip address.

- Change the `password` in `/group_vars/all.yml` (if needed):

    That password wil be used for disk encryption and for the root user --- make sure you use a long random password, you will almost never need to type it manually and it must be secure.

    Because this repo is public you must encrypt the value of the password with an encryption-password --- also make sure you use a long, random password. To avoid mistyping the password store it on a file with the following command:

    ```bash
    echo "{password}" > ~/.vault_key; chmod 600 ~/.vault_key
    ```

    Produce the encrypted value for the `password` variable:

    ```bash
    ansible-vault encrypt_string '{password}' --vault-password-file ~/.vault_key --name 'password'`
    ```

    Copy the output and replace the `password` variable value in `/group_vars/all.yml`.

- Check default variables in `/group_vars/all.yml`:

    Specially check `target_disk`, you may be using a nvme disk. Use `fdisk -l` to identify the target disk.

    Do not change the default values on `/group_vars/all.yml`, overwrite them by including them on `/host_vars/{hostname}.yml`.


## Run the script

Change directory into the cloned repo and run:

```bash
ansible-playbook local.yml -k --vault-password-file ~/.vault_key
```

You will be prompted for the root password you just changed.

If no erros occur the managed node will shutdown automatically after install. Remove install media and turn it back on.


## Connect to internet

Once the script completes the computer will automatically reboot.

Use the `nmtui` command to connect to a wireless network.

