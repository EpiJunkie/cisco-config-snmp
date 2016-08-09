# Cisco Config Backup v0.1:


## Purpose
The purpose of this script is to initiate a transfer of the running and startup configurations from a list of Cisco devices using SNMP. This does not require a configuration on the Cisco devices except for a writable SNMP group. Below are Cisco configuration examples to help you get started.

## Usage
There are two modes as declared in variable `_mode`.

* `insecure` mode initiates the configuration of the transfer over SNMP version 1 which is in cleartext and the transfer occurs over TFTP also in cleartext. If using this mode, obviously you can not be bothered to use any security precautions.

* `secure` mode initiates the configuration of the transfer over SNMP version 3 over an authenticated and encrypted channel. Then the transfer occurs over SCP which is also encrypted. The SSH user and password should be a restricted user which only has access to these configuration files. At the very least this script should be protected by using `chmod 600` and owned by root to prevent access to the stored SSH password.

Commands:
```
cisco-backup-config.sh
cisco-backup-config.sh insecure|secure
cisco-backup-config.sh insecure|secure erase
```

The first command uses the variable `_mode` to determine the mode to run in. The second command uses the user input to determine the mode to run in. The last command above erases the MIB created for that mode, this is normally done at the end of each run but may be required to run independently for debugging purposes.

To run a daily cron at midnight run:

```
root@unix-host:~ # crontab -e
0 0 * * * /root/cisco-backup-config.sh >> /var/logs/cisco-config-backup.log 2>&1
```

## Cisco Example Config:
Below are example Cisco configurations for each mode which reflect the initial defaults.

Insecure
```
router01(config)# ! Create a v1,v2c community name "orgRW" with read and write access.
router01(config)# snmp-server community orgRW rw
```

Secure
```
router01(config)# ! Create a group "ITTEAM", with authPriv security access and write access as
router01(config)# !  described in group view "BACKUPMIBONLY" when connecting only under conditions set in ACL 45.
router01(config)# snmp-server group ITTEAM v3 priv write BACKUPMIBONLY access 45
router01(config)# !
router01(config)# ! Create a group view "BACKUPMIBONLY" that only has access to the MIB "1.3.6.1.4.1.9.9.96.1.1.1.1" for this task.
router01(config)# snmp-server view BACKUPMIBONLY enterprises.9.9.96.1.1.1.1 included
router01(config)# !
router01(config)# ! Create a version 3 user "BACKUPUSER" under the group "ITTEAM" that authenticates with SHA with
router01(config)# !  password "USERPASS" using  AES 128-bit encryption with the key "ENCKEY".
router01(config)# snmp-server user BACKUPUSER ITTEAM v3 auth sha USERPASS priv aes 128 ENCKEY
router01(config)# !
router01(config)# ! Create a standard ACL with access from one host and log all other attempts.
router01(config)# access-list 45 remark "SNMP: ITTEAM access"
router01(config)# access-list 45 permit host XXX.XXX.XXX.XXX
router01(config)# access-list 45 deny any log
```

## Dependancies
This tool requires the Bourne `sh`ell and `net-snmp` on the system running this script. The system receiving the configuration must have a `ssh` or `tftp` server running.

This was tested on FreeBSD, CentOS, and Debian. On Windows, `tftpd32` was successfully tested to receive the configuration files.


## Cisco Device List File:
The file described in variable `_devices_text_file` is the full path to a plain text file which contains the device name, a single space, and then an IP of the device. The next line follows the same format for the next device. An IP is required as `snmpset` requires it. If the device names are in DNS, then an attempt is made to resolve a IPv4 address using `drill` or `dig`, depending on the OS. Blank lines and lines starting with an octothrop "#" are ignored. Example:

```
# Site 1
router01 10.0.0.1
switch01 10.0.0.2
switch02 10.0.0.3

# Site 2
router11 10.0.1.1
switch11 10.0.1.2
switch12 10.0.1.3
```
