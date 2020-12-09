# ansible-role-raspi-setup

An Ansible role to perform basic setup on a group of Raspberry Pi hosts, intended for use
with headless machines.

When run on a fresh installation of Raspberry Pi OS with only ssh enabled (add an empty file
called `ssh` to the `boot` partition on the SD card before first boot to do this!), this
role can be used to:

  * Set the `hostname` to match the hostname given in the Ansible inventory or from a variable called `local_hostname`
  * Set various options in different Pis' `config.txt` from a variable called `config_settings`
  * Add authorized ssh keys to the `pi` user for passwordless login from a variable called `authorized_keys`
  * Change the default password on the `pi` user from a variable called `pi_password`

To skip any of the steps, simply leave the relevant variable unset.

I recommend combining this with the `dev-sec.ssh-hardening` which will lock down ssh access
to disallow password login for security as well as making a range of other security improvements
to sshd's default setup.

Additionally, you can use the `geerlingguy.swap` to set up a swap file if needs be!

# Example playbook

## Main playbook `playbook.yml`

    - hosts: raspis
      roles:
        # This role will automatically set up passwordless SSH for the given keys
        # and customize any config.txt settings passed in via the host inventory.
        - role: fpiesche.raspi_setup
          vars:
            # These values will be the defaults for all of the Pis, but can be
            # overridden on a per-host basis.            
            authorized_keys: ["{{ lookup('file', lookup('env', 'HOME') + '/.ssh/id_rsa.pub') }}",
                              "/home/otheruser/.ssh/id_rsa.pub",
                              "ssh-rsa ..."]
            # The `pi_password` variable must be a hashed password.
            pi_password: "{{ raspberry | password_hash('sha512') }}"

        # RECOMMENDED: This role will lock down ssh access to disallow password login
        # and make other security improvements to the default ssh setup.
        # https://github.com/dev-sec/ansible-ssh-hardening
        - role: dev-sec.ssh-hardening
          become: yes
        
        # OPTIONAL: Set up a swap file using the `swap_megabytes` host variable
        # https://github.com/geerlingguy/ansible-role-swap
        - role: geerlingguy.swap
          become: yes

## Inventory `hosts.yml`

Here's a slice of the hosts.yml I use for my Docker cluster of various versions of Pi:

    ---
    all:
      children:
        raspis:
          vars:
            ansible_python_interpreter: /usr/bin/python3
            ansible_user: pi
            ansible_password: raspberry
          hosts:
            pi-1-a.local:
              # Set a different password for the `pi` user on this Pi
              pi_password: "{{ pi1-password | password_hash('sha512') }}"
              # Anything in the config_settings section will be automatically set
              # in this Pi's config.txt and the system rebooted afterwards.
              config_settings:
                - name: "gpu_mem"
                  value: 16
                # Raspberry Pi 1 overclocking settings taken from
                # https://haydenjames.io/raspberry-pi-safe-overclocking-settings/
                - name: "arm_freq"
                  value: 1000
                - name: "sdram_freq"
                  value: 500
                - name: "core_freq"
                  value: 500
                - name: "over_voltage"
                  value: 6
                - name: "temp_limit"
                  value: 75
              # This is used by Jeff Geerling's `geerlingguy.swap` role to set up a 1GB swap file.
              # https://github.com/geerlingguy/ansible-role-swap
              swap_file_size_mb: 1024

            pi-2.local:
              pi_password: "{{ pi2-password | password_hash('sha512') }}"
              local_hostname: that-other-one
              config_settings:
                - name: "gpu_mem"
                  value: 16
                # Raspberry Pi 2 overclocking settings taken from
                # https://wiki.debian.org/RaspberryPi#Overclocking_Pi_2
                - name: "arm_freq"
                  value: 1000
                - name: "core_freq"
                  value: 500
                - name: "sdram_freq"
                  value: 400
                - name: "over_voltage"
                  value: 0
                - name: "over_voltage_sdram_p"
                  value: 0
                - name: "over_voltage_sdram_i"
                  value: 0
                - name: "over_voltage_sdram_c"
                  value: 0
                - name: "temp_limit"
                  value: 75
              swap_file_size_mb: 1024
