Unboxed - SSH Server Configs
============================

Lets you configure an SSH server with reasonable configs.

Usage
=====

In your primary yaml file:

    - name: Server name
      roles:
        - ssh-server

In your `host_vars/servername.example.com.yml` file:

    sshd_config_password_authentication: false
    sshd_config_x11_forwarding: false
    sshd_config_permit_root_login: false
    sshd_config_allow_agent_forwarding: false

Available Config Options
========================

* `sshd_config_password_authentication` - true or false - Allows ssh login without a private key. Default is false.
* `sshd_config_permit_root_login` - true or false - Permits root login if true. Default is false.
* `sshd_config_x11_forwarding` - true or false - Permits X11 forwarding if true. Default is false.
* `sshd_config_allow_agent_forwarding` - true or false - Permit or deny ssh agent forwarding. Default is false

Author Information
------------------

Unboxed Consulting - Oskar Pearson