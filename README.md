# Basic Archlinux installation with Ansible

I love Archlinux but I don't want to manually redo everything time and again. To fully automate my setup I created 2 ansible scripts:

1. [ansible-archlinux](https://github.com/mabq/ansible-archlinux) (this repo) - fully automates a [basic Archlinux installation](https://wiki.archlinux.org/title/Installation_guide), leaving the host ready to run the second script.
2. [ansible-setup](https://github.com/mabq/ansible-setup) installs and configures all the tools I need.

#### Why two different scrips?

The [live environment](https://wiki.archlinux.org/title/Installation_guide#Boot_the_live_environment) you get with the Arch installation image does not include the [ansible](https://archlinux.org/packages/extra/any/ansible/) package, so this script **must** be executed from a [controller machine](https://docs.ansible.com/ansible/latest/getting_started/index.html#getting-started-with-ansible).

The *ansible-setup* script on the other hand, can be executed via `ansible-pull` in the localhost (more details on that repo).


## About this script

   - Executes only if the managed node was booted from the Arch installation image.
   - Automatically detects the firmware type and configures the system accordingly.
   - Automatically detects the disk type and enables TRIM (if supported).
   - Automatically detects the processor and installs microcode updates accordingly.
   - Automatically sets up disk encryption, [lvm](https://wiki.archlinux.org/title/LVM) and swap (see `group_vars`).


## Run the script

Before running the script:
    - Make sure the `hosts` file includes the hostname of the managed host/s you intend to affect.
    - Make sure there is a file for each of the hosts you intend to affect under the `host_vars` directory --- the only required variable is `ansible-host` (make sure the IP is correct).
    - Read the default varibles in `group_vars` --- overwrite any of those by including that variable in the `{hostname}` file in the `host_vars` directory.

Execute the script:

```bash
ansible-playbook local.yml -k --vault-password-file ~/.vault_key
```

Special thanks to [LearnLinuxTv](https://www.youtube.com/@LearnLinuxTV) and [ThePrimeagen](https://www.youtube.com/@ThePrimeagen) for their great content.

I use Ansible to configure all of my desktops, laptops, and servers. I use the Ansible Pull method, which I describe in both my Ansible Pull tutorial (already uploaded) and my Ansible desktop tutorial (will be published by December 11th 2020). It's fully automated, and scripts everything from system services to GNOME desktop settings.

Disclaimer
This repository contains a copy of the Ansible configuration that I use for laptops, desktops as well as servers. Please don't directly use this against your own machines, as it is something I developed for myself and may not translate to your use-case. It even configures OpenSSH, so if you run it you may get locked out. I've provided this as a HUGE example you can use to build your own, and compare syntax.



This is an Ansible script that automates the 

The goal of this script is to automate a basic Archlinux installation. Once the script completes succesfully, execute the [setup](https://github.com/mabq/ansible-setup) script.

> The `ansible` package is not part of the *Live Environment*, so you will need a second machine with `ansible` installed to act as the control node.

The script should work with any BIOS/UEFI system with an Intel/AMD processor and any type of disk.


## Before running the script

Review the script (`local.yml`) and the default variable values (`/group_vars/all.yml`). Overwrite any variable value in each host file (`/host_vars/{host}.yml`).

Change the values of the variables `encryption_password` and `root_password`. To encrypt a string using ansible use the following command:

```bash
ansible-vault encrypt_string --vault-password-file {PATH_TO_FILE_WITH_ENCRYPTION_PASSWORD} '{STRING_TO_ENCRYPT}' --name '{VARIABLE_NAME}' 
```


## Run the script

1. Change root password on the managed node:

    ```bash
    # this is just a temporary password
    passwd
    ```

2. Run the script:

    ```bash
    # use this command to enter the encryption password manually 
    ansible-playbook local.yml -k --ask-vault-pass

    # use this command to use a encryption password file
    ansible-playbook local.yml -k --vault-password-file {PATH_TO_ENCRYPTION_PASSWORD_FILE}
    ```

    Either way, you will be prompted for the `root` user password of the managed node (the one you set in the previous step).

3. Connect to internet:

    Once the script completes the computer will automatically reboot.

    Use the `nmtui` command to connect to a wireless network if needed.

