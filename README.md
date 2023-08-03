# Install Archlinux with Ansible

The goal of this repository is to have an ansible script that automates the installation of a base Archlinux system. Once the base system is in place the remaining configurations can be done with `ansible-pull` and other repos.

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

