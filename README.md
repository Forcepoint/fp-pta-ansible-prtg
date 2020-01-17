# prtg-server

PRTG is a monitoring solution provided by the company Paessler. https://www.paessler.com/prtg

> Monitor all systems, devices, traffic and applications of your IT infrastructure. Everything you need is contained in PRTG, 
> no additional downloads are required.

As of this writing, PRTG is a Windows only application, and it does NOT run in docker. This complicates the traditional
approach used here of being able to maintain an application without issue even when the VM is replaced. It can still be done
but you must backup the needed application files and registry settings to restore from later.

* https://kb.paessler.com/en/topic/463-how-and-where-does-prtg-store-its-data
* https://kb.paessler.com/en/topic/523-how-do-i-backup-all-data-and-configuration-of-my-prtg-installation
* https://kb.paessler.com/en/topic/413-how-can-i-move-migrate-my-prtg-installation-to-another-computer

Data in question as of this writing...
* `C:\Program Files (x86)\PRTG Network Monitor\`
* `C:\ProgramData\Paessler\PRTG Network Monitor`
* Registry `HKEY_LOCAL_MACHINE\SOFTWARE\Wow6432Node\Paessler\PRTG Network Monitor`

The reason PRTG was chosen as the monitoring solution here is because it's easy to use and setup. Granted,
the product leaves a lot of be desired from an Infrastructure as Code perspective, but it is crazy easy to use
and it has a free license if you keep your sensor count under one hundred. The learning curve for other popular
monitoring solutions is **EXTREMELY** steep by comparison.

The first time installed, you need to...

1. Change the prtgadmin user's password immediately. The default username is `prtgadmin`
and the default password is `prtgadmin`.
1. Switch to SSL.
1. Ensure emails can be sent.
1. Add the default credentials for Windows and Linux machines to the Root Group so the devices you add can 
   inherit this automatically.
1. Add any desired groups under Local Probe.

It is recommended to NOT use auto discovery as that will very quickly eat up your count of free sensors.

## Disaster Recovery

If you have utilized the backup-local-win role to make backups, follow these steps to recover from backups of the data outlined above...

1. Run ansible to setup the application for you. As long as ansible completes, you're good to proceed.
1. `sc stop PRTGCoreService`
1. `sc stop PRTGProbeService`
1. `sc query PRTGCoreService`
1. `sc query PRTGProbeService`
1. `rmdir /S /Q "C:\Program Files (x86)\PRTG Network Monitor"`
1. `rmdir /S /Q "C:\ProgramData\Paessler\PRTG Network Monitor"`
1. `reg delete HKLM\SOFTWARE\Wow6432Node\Paessler /f`
1. Extract the program_files backup to `C:\`.
1. Extract the program_data backup to `C:\`.
1. Extract the registry backup to `C:\`.
1. `cmd.exe /C C:\backups\registry.reg`
1. `rmdir /S /Q "C:\backups"`
1. `sc start PRTGCoreService`
1. `sc start PRTGProbeService`
1. `sc query PRTGCoreService`
1. `sc query PRTGProbeService`

## Requirements

If you have trouble setting up the certificates appropriately, try using the Certificate Importer and then copying
the files off and encrypting them for the role to consume. FYI, PRTG will want the key file to be in RSA format.
I couldn't figure out what the format of the root.pem was supposed to be, so I ended up using the certificate importer. 
https://www.paessler.com/tools/certificateimporter

The following files are required to be present. 
Ensure they're vaulted before committing them to source code control.

* `files/{{ prtg_server_dns }}.key` The private key.
* `files/{{ prtg_server_dns }}.pem` The web certificate.

### OPTIONAL

* `files/root.pem` The root CA's certificate, possibly combined with an intermediate CA, if needed.
  This would likely be the case if you are using your own private CA.

## Role Variables

### REQUIRED
* prtg_server_dns: The DNS name for the PRTG server. This DNS entry must already exist.
* prtg_server_installer: The download URL For the PRTG EXE installer. 
  PRTG provides a zip which I didn't want to deal with, so you should unzip and 
  stage the EXE somewhere like Artifactory.
* prtg_server_product_id: The Product ID as defined in the registry. 
  If you use version 18.3.43.2323, this is `{5EC294B8-98F8-4C20-BE73-F11A04295CA5}_is1`.
  Sorry, chicken and egg problem. You have to have this to use the win_package module here.
* prtg_server_admin_email: The email address for the PRTG administrator.
* prtg_server_license_key: The license key provided by PRTG on their download page. 
  https://www.paessler.com/download/prtg-download

### OPTIONAL
* prtg_server_log: The path for the installation log. Defaults to `C:\prtg_install.log`. Recommend not using spaces in the path.
* prtg_server_license_name: The name of the license as provided by PRTG on their download page. Default is `prtgtrial`.

## Dependencies

None

## Example Playbook

    - hosts: servers
      vars:
        prtg_server_dns: prtgmonitor.COMPANY.com
        prtg_server_installer: "https://artifactory.COMPANY.com/artifactory/software/Paessler/PRTG/18.3.43/prtg.zip!/PRTG%20Network%20Monitor%2018.3.43.2323%20Setup%20(Stable).exe"
        prtg_server_product_id: "{5EC294B8-98F8-4C20-BE73-F11A04295CA5}_is1"
        prtg_server_admin_email: "FIRST.LAST@COMPANY.com"
        prtg_server_license_key: 000014-11111-111111-111111-11111-111111
      roles:
         - role: prtg-server

## License

BSD

## Author Information

Jeremy Cornett <jeremy.cornett@forcepoint.com>
