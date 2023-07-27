# Ansible - Arch base install

Ansible script to perform a basic installation of Archlinux on BIOS or UEFI.

The `ansible` package is not part of the *Live Environment*, so you will need a second machine with `ansible` installed to act as the control node.

1. Change root password on the managed node:

    ```bash
    passwd
    ```

    This is only a provisional password that will allow us to run the Ansible script on the managed node.

2. Connect via ssh at least once to add the fingerprint to the `~/.ssh/known_hosts` file.

3. Run the script:

    Before running the script, check for any erros:

    ```bash
    ansible-playbook local.yml --check
    ```

    If everything is ok, run the script:

    ```bash
    ansible-playbook local.yml -k
    ```

    You will be prompted to enter the password of the `root` user (the one you setup in the previous step).


