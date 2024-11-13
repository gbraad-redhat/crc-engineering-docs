Windows installation process
============================

On Windows, setting up the system is shared between the installer (msi or chocolatey) and crc preflights


## MSI installer
- creates the `crc-users` group
- adds the current user to the `crc-users` group
- sets up the `admin-helper` service
- creates the registry key required by hvsock
- adds the user to the hyper-v admin group
- installs Hyper-V
- configures SMB for file sharing

## Chocolatey
- creates the `crc-users` group
- sets up the `admin-helper` service
- creates the registry key required by hvsock
- installs hyper-v


## CRC preflights
- checks if the `crc-users` group exists
- checks if hyper-v is installed and running
- checks if the hvsock registry key exists
- checks if the admin-helper service is running
- adds the current user to the `crc-users` group and hyper-v admin group
- starts the crc daemon task
