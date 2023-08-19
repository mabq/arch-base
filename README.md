# Basic Archlinux installation with Ansible

I love Archlinux but I don't want to manually redo everything time and again. To fully automate my setup I created 2 ansible scripts:

1. [ansible-archlinux](https://github.com/mabq/ansible-archlinux) (this repo) - fully automates a [basic Archlinux installation](https://wiki.archlinux.org/title/Installation_guide), leaving the machine ready to run the second script.
2. [ansible-setup](https://github.com/mabq/ansible-setup) installs and configures all the tools I need.

#### Why two different scrips?

The [live environment](https://wiki.archlinux.org/title/Installation_guide#Boot_the_live_environment) you get with the [Archlinux installation image](https://archlinux.org/download/) does not include the [ansible](https://archlinux.org/packages/extra/any/ansible/) package, so this playbook (`local.yml`) must be executed from another machine (a [controller node](https://docs.ansible.com/ansible/latest/getting_started/index.html#getting-started-with-ansible), the machine where you are installing Archlinux becomed the managed node).

The [ansible-setup](https://github.com/mabq/ansible-setup) playbook on the other hand, can be executed with `ansible-pull` on that same machine (more details on that repo).


## About this script

   - Executes only if the managed node was booted from the Arch installation image (super important to avoid running it by accident and possibly lose all data on inventory hosts).
   - Detects firmware type and creates partitions accordingly.
   - Detects disk type and enables TRIM (if supported).
   - Detects processor and installs microcode updates.
   - Sets up [lvm](https://wiki.archlinux.org/title/LVM) on top of full disk encryption (generates a key file so you don't need to type the encryption password when booting).
   - Installs basic packages (leave the node ready for [ansible-setup](https://github.com/mabq/ansible-setup)).
   - Enables sshd and NetworkManages services.
   - Sets up time zone, locales, hostname, etc.
   - Sets up swap (if desired, see `group_vars`)


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

    Make sure the name of the host you intend to affect is listed. That name will become the `hostname` of the managed node.

- Change the `password`:

    The same password wil be used for disk encryption and for the root user --- make sure you use a long random password, you will almost never need to type it manually and it must be secure.

    Because this repo is public you must encrypt the value of the password with an encryption-password --- also make sure you use a long, random password. To avoid mistyping the password store it on a file with the following command:

    ```bash
    echo "{password}" > ~/.vault_key; chmod 600 ~/.vault_key
    ```

    Produce the encrypted value for the `password` variable:

    ```bash
    ansible-vault encrypt_string '{password}' --vault-password-file ~/.vault_key --name 'password'`
    ```

    Copy the output and replace the `password` variable value in `/group_vars/all.yml`.

- Check default variables:

    Do not change the values on `/group_vars/all.yml`, overwrite them on each `/host_vars/{host}.yml` file.

    Specially check `target_disk`, you may be using a nvme disk. Use `fdisk -l` to identify the target disk.


## Run the script

Change directory into the cloned repo and run:

```bash
ansible-playbook local.yml -k --vault-password-file ~/.vault_key
```

You will be prompted for the root password you just changed.


## Connect to internet

Once the script completes the computer will automatically reboot.

Use the `nmtui` command to connect to a wireless network.

