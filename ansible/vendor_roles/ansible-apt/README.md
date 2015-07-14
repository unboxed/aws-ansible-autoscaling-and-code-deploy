[![Build Status](https://travis-ci.org/yetu/ansible-apt.svg?branch=master)](https://travis-ci.org/yetu/ansible-apt)

ansible-apt
==================

Ansible apt role which manages apt on a ubuntu 12.04 systems, it has two functionalities:

1. Keeps your apt cache updated, cleans dependencies, removes .deb file and configures your apt system 

2. Does automatic package updates on debian systems (tested on ubuntu 12.04) for more info check [automatic-updates](https://help.ubuntu.com/12.04/serverguide/automatic-updates.html)


##Prerequisite
* If you are using mail notification you must have configured mailx


##Variables 
  All the default variables are located **defaults/main.yml**. Mostly they are al sane but 

```yaml
---
## *********** Apt config ***********
apt_update_source_list        : "no"  # Deploy Apt source.list ['no', 'copy', 'template']
apt_update_source_list_mirror : "http://us.archive.ubuntu.com/ubuntu/" # apt mirror only works with apt_update_source_list='template'
apt_cache_valid_time          : 3600  # apt cache validity 
apt_autoremove                : yes   # remove left over dependencies of packages no longer have
apt_autoclean                 : yes   # clears out the local repository of retrieved package files
apt_default_key_urls          : []    # Default apt keys to install
apt_default_repositories      : []    # Default apt repos to install
apt_default_packages:                 # Default package to install 
  - python-apt
  - unattended-upgrades
apt_config                    :       
  "APT::Install-Recommends"   : "no"  # Install the "recommended" packages recommanded 'no'
  "APT::Install-Suggests"     : "no"  # Install the "suggested" packages recommanded 'no'
  "APT::Get::Show-Upgraded"   : "yes" # Print out a list of all packages that are to be upgraded

## *********** Unattended upgrade config ***********
unattended_install      : false       # Install unattended packages false or true (default not to install)
unattended_blacklist    : []          # Array of packages not to be updated (set [ ] for empty list or [ "vim", "libc6" ])  
unattended_origins      :             # What origins to update
   security  : true
   updates   : false
   proposed  : false
   backports : false
unattended_autofix          : true   # Control if on a unclean dpkg exit will automatically run dpkg --force-confold --configure -a
unattended_steps            : false  # Split the upgrade into the smallest possible chunks 
unattended_shutdown         : false  # Install upgrades when the machine is shuting down instead of doing it in the background
unattended_mail             : "root" # Send email to this address. (set false to disable emails)
unattended_mail_error       : false  # Get emails only on errors. Default is to always send a mail if unattended_mail is set
unattended_remove_unused    : false  # Automatic removal of new unused dependencies after the upgrade
unattended_automatic_reboot : false  # Automatically reboot *WITHOUT CONFIRMATION* if packages require that
unattended_download_limit   : false  # Set to Integer representing kb/sec limit else false


## *********** Update your system***********
upgrade_now                     : False  # Enable upgrade now feature
upgrade_now_force               : False  # Force upgrade and reboot if needed all checks will be ignored
upgrade_now_pause_after_reboot  : 10 # Sleep time after machine reboots
upgrade_now_first_boot_file     : "/var/run/ansible_apt_upgrade"
upgrade_now_apt_reboot_file     : "/var/local/reboot-required"
```

## TODO
- make better tests
- make ansible-galaxy role
- Support trusty
