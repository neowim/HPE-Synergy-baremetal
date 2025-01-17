# HPE Synergy bare metal provisioning

Ansible project for automatic provisioning of HPE Synergy bare metal servers.

Operating system provisioning (ESXi, RHEL and Windows server) is performed using kickstart/unattend files, automatically generated ISO files and HPE OneView server profile templates.

## Use case

The different playbooks can be used to provision 3 types of operating systems:

- Red Hat Enterprise Linux or equivalent
- VMware ESXi 6.7 and 7
- Windows Server 2022 or equivalent

One playbook can provision one OS type on one or multiple servers as defined by the Ansible inventory file.

## Pre-requisites

- HPE Synergy frame configured and at least one unused Synergy 480 Gen10 compute module.
- OneView Server Profile Templates defined for each desired OS with a local storage boot drive or a boot from SAN storage volume (see below for configuration).
- OneView server profile templates must include the creation of an iLO local account. This account is required by the [community.general](https://galaxy.ansible.com/community/general) collection to manage an HPE iLO interface.
- Ansible control node (see below for configuration) with a drive large enough to host the generated ISO files.
- Windows DNS server configured to be managed by Ansible (see below for configuration).

## Ansible control node information

- It runs Ansible
- It can be a physical server or a Virtual Machine
- It is used as the staging destination for the preparation of the ISO file(s)
- It runs `nginx` web services to host the created and customized ISO files from which the bare metal servers will boot from using iLO virtual media.

## Configure Ansible control node

To configure the Ansible control node, see [Ansible_control_node_requirements.md](https://github.com/jullienl/HPE-Synergy-baremetal/blob/master/files/Ansible_control_node_requirements.md) in `/files`

## Configure Windows DNS Server

The Windows DNS Server to be managed by Ansible should meet below requirements:

- PowerShell 3.0 or newer
- .NET 4.0 to be installed
- A WinRM listener should be created and activated

To configure WinRM, you can simply run [ConfigureRemotingForAnsible.ps1](https://github.com/ansible/ansible/blob/devel/examples/scripts/ConfigureRemotingForAnsible.ps1) on the Windows Server to set up the basics. This script sets up both HTTP and HTTPS listeners with a self-signed certificate and enables the `Basic` authentication option on the service. This is an offical PowerShell script from Ansible repository.
To learn more about **Setting up Windows host**, see https://docs.ansible.com/ansible/2.5/user_guide/windows_setup.html#winrm-setup

## Preparation

1. Update all variables located in `/vars` and in `/group_vars/Windows.yml` for the Windows host provisioning

2. Copy the desired OS ISO versions on a web server defined by `{{ src_iso_url }}` and `{{ src_iso_file }}` variables.

3. Create an HPE Oneview Server Profile Template for each OS type.

   The following playbooks can be used to create the appropriate server profile templates:

   - `ESXi_SPT_creation_Boot_from_Logical_Drive.yml`
   - `ESXi_SPT_creation_Boot_from_SAN.yml`
   - `RHEL_SPT_creation_Boot_from_Logical_Drive.yml`
   - `RHEL_SPT_creation_Boot_from_SAN.yml`
   - `WIN_SPT_creation_Boot_from_Logical_Drive.yml`
   - `WIN_SPT_creation_Boot_from_SAN.yml`

     > **Note**: For the boot from SAN playbooks, it is necessary to activate the native mode of Jinji2 to define the size of the SAN volume, for more information, see the notes in the boot from SAN playbooks.

   Server profile templates must meet the following parameters to be compatible with bare metal provsioning playbooks:

   - They must be defined with at least 6 network connections:

     - 2 for management
     - 2 for FCoE
     - 2 for production using a network set

       > **Note**: ESXi playbook adds a second management NIC for vswitch0 and looks for the two NICs connected to the defined network set to create the Distibuted switch for VM traffic. RHEL and Windows playbooks only create a team using the first two management NICs.

   - They can be defined with either a boot from local storage (e.g. RAID 1 logical drive using the two internal drives) or a boot from SAN volume for operating system storage.

     > **Notes**: Additional shared/private SAN volumes for vmfs datastore/cluster volumes can also be defined (RHEL and ESXi playbooks look for the boot LUN to install the OS).

     > **IMPORTANT NOTE**: For successful boot LUN detection with RHEL and ESXi, it is essential to ensure that there are no other LUNs of the same size as the boot LUN presented to the host. Otherwise, LUN detection may fail and operating system files may be mistakenly copied to an incorrect LUN, resulting in OS boot failure.

   - They must be defined with a firmware baseline with at least the `Firmware only` installation method as for RHEL and Windows provisioning, the HPE drivers are installed at the end of the playbook.
     > **Note**: For ESXi, there is no need to install HPE drivers because HPE ESXi images include all the drivers and management software required to run ESXi on HPE servers, therefore there is no need to define a firmware baseline.

## How to protect sensitive credentials

Vault files can be used to secure all passwords. To learn more, see https://docs.ansible.com/ansible/latest/user_guide/vault.html

- To encrypt a var file: `ansible-vault create --vault-id @prompt vars/encrypted_credentials.yml`
- To run a playbook with encrypted credentials: `ansible-playbook <playbook.yml> --ask-vault-pass`

## Description of the playbooks

### RHEL_provisioning.yml

This playbook performs for each inventory host the automated installation of RHEL 8.3 Boot from SAN using a customized kickstart, the main steps are as follows:

- Create a DNS record for the bare metal server in the defined Windows DNS server
- Download the OS vendor ISO file from a web server
- Mount the ISO and copy all files from the RHEL ISO image to a staging directory
- Modify Legacy bios and UEFI bootloaders for kickstart installation from CDROM
- Create a HPE OneView Server Profile from an existing Server Profile Template
  - Display Server hardware automaticaly selected by HPE OneView
- Capture the size of the primary boot volume (if any) for the customization of the kickstart file
- Customize the kickstart file with among others:
  - Set IP parameters
  - Set root password
  - Create a %pre script to detect the primary boot from SAN volume (if any)
  - Create a %post to set RHEL repositories and hostname
  - Add Ansible control node SSH public key to .ssh/authorized_keys
  - Set keyboard and language settings
  - Set time zone
  - Set ntp servers
- Generate a temporary ISO file with the customized kickstart file
- Power on and boot the inventory host from created ISO using iLO virtual media
- Wait until RHEL installation is complete
- Delete the created ISO and all temporary files in the stagging location after the custom installation is complete
- Install the HPE drivers for RHEL
  - Install HPE iSUT and HPE AMS on the newly provisioned server
  - Configure HPE iSUT for online installation of HPE drivers for RHEL using HPE OneView
  - Update the HPE OneView Server Profile to initiate operating system driver (and firmware if necessary) updates using HPE SUT
  - Wait until the SUT installation is complete
- Reboot the server for the activation of HPE drivers/firmware

### RHEL_unprovisioning.yml

This playbook performs for each inventory host the automated un-provisioning of the RHEL OS:

- Power off the server
- Delete the HPE OneView Server Profile
- Remove the DNS record
- Remove the host SSH key from .ssh/known_hosts

### ESXi_provisioning.yml

This playbook performs for each inventory host the automated installation of VMware ESXi 7.0.2 Boot from SAN using a customized kickstart, the main steps are as follows:

- Create a DNS record for the bare metal server in the defined Windows DNS server
- Download the HPE ESXi Custom Image ISO file from a web server
- Mount the ISO and copy all files from the ESXi HPE Custom ISO image to a staging directory
- Modify Legacy bios and UEFI bootloaders for kickstart installation from CDROM
- Create an HPE OneView Server Profile from an existing Server Profile Template
  - Display Server hardware automaticaly selected by HPE OneView
- Capture the size of the primary boot volume (if any) for the customization of the kickstart file
- Capture MAC addresses of the production NICs attached to the defined network set for subsequent configuration of the Distributed vSwitch.
- Customize the kickstart file with among others:
  - Set IP parameters
  - Set root password
  - Create a %pre script to detect the primary boot from SAN volume (if any)
  - Create a %firstboot to set hostname, DNS suffix and FQDN
  - Add Ansible control node SSH public key to /etc/ssh/keys-root/authorized_keys at %firstboot
- Generate a temporary ISO file with the customized kickstart file
- Power on and boot the inventory host from created ISO using iLO virtual media
- Wait until ESXi installation is complete
- Delete the created ISO and all temporary files in the stagging location after the custom installation is complete
- Add the ESXi host to the defined vCenter server
- Assign defined vSphere ESXi license to the host
- Add vmnic1 to the standard switch vSwitch0 as the second active adapter
- Add vMotion Portgroup to the standard switch vSwitch0
- Connect the host to the defined Distributed vSwitch using the available vmnics
- Set the Power Management Policy to high-performance
- Enable SSH and Shell services

### ESXi_unprovisioning.yml

This playbook performs for each inventory host the automated unprovisioning of the VMware ESXi OS:

- Put the host in maintenance mode
- Remove the Host from the defined Distributed vSwitch
- Remove the host from the defined vCenter Server
- Power off the server
- Delete HPE OneView Server Profile
- Delete DNS record

### WIN_provisioning.yml

This playbook performs for each inventory host the automated installation of Windows Server 2022 Boot from SAN using an unattended custom file, the main steps are as follows:

- Download the OS vendor ISO file from a web server
- Mount the ISO and copy all files from the Windows Server ISO image to a staging directory
- Create an HPE OneView Server Profile from an existing Server Profile Template
  - Display Server hardware automaticaly selected by HPE OneView
- Capture MAC address of first two management NICs for the configuration of the network settings in configure_network.ps1
- Create $OEM$ resources to host the scripts that need to be executed at startup:
  - Import a PowerShell script from the Ansible repository to %OEM% to configure Windows for remote management with Ansible
  - Create a PowerShell script configure_network.ps1 to be launched by SetupComplete.cmd at startup to:
    - Configure the NIC teaming with the first two NICs
    - Configure the IP parameters
- Customize the unattend file with among others:
  - Generate a standard Windows disk configuration for UEFI with four GPT partitions (WinRE/EFI/MSR/Windows):
    - WinRE: 450MB
    - EFI: 100MB
    - MSR: 16MB
    - Windows: use all available space
  - Set region and language settings
  - Set product key
  - Set registered user and organization
  - Set time zone
  - Set administrator password
  - Launch winRM installation for Ansible at startup
  - Set remote desktop
  - Set computer name
- Generate a temporary ISO file with customized unattend file and scripts
- Power on and boot the inventory host from created ISO using iLO virtual media
- Wait until Windows Server installation is complete
- Delete the created ISO and all temporary files in the stagging location after the custom installation is complete
- Add a DNS record for the newly provisioned server in the defined DNS server
- Install the HPE drivers for Windows Server
  - Install HPE iSUT and HPE AMS on the newly provisioned server
  - Configure HPE iSUT for online installation of HPE drivers for Windows Server using HPE OneView
  - Update the HPE OneView Server Profile to initiate operating system driver (and firmware if necessary) updates using HPE SUT
  - Wait until the SUT installation is complete
- Join the newly provisioned server to the defined Windows domain
- Reboot the server for activation of HPE drivers/firmware and domain membership

### WIN_unprovisioning.yml

This playbook performs for each inventory host the automated unprovisioning of the Windows Server OS:

- Power down the server
- Delete the HPE OneView server profile
- Delete the DNS record

## Built and tested with

The resources in this repository were tested with:

- Ansible control node running on CentOS 8.2 VM:
  - Ansible 2.9.25 - Python 3.6.8 - python-hpOneView 6.30
  - Community.general 3.8.0
  - Community.windows 1.7.0
  - Community.vmware 1.15.0
  - Ansible Collection for HPE OneView 6.30
- HPE OneView 6.30
- Synergy 480 Gen10
- SSP 2021-05.03

- Provisioned OS tested successfully:
  - RHEL-8.3.0-20201009.2-x86_64-dvd1.iso
  - VMware-ESXi-7.0.2-17630552-HPE-702.0.0.10.6.5.27-Mar2021-Synergy.iso
  - en-us_windows_server_version_2022_updated_october_2021_x64_dvd_b6e25591.iso

## Output sample of ESXi bare metal provisioning playbook

```
ansible-playbook -i hosts ESXi_provisioning.yml

PLAY [Creating a DNS record for the bare metal ESXi server] ***********************************************************************

TASK [Adding "ESX-1" with "192.168.3.171" on "dc.lj.lab" in "lj.lab" DNS domain] **************************************************
changed: [ESX-1 -> dc.lj.lab]

PLAY [Performing an automated ESXi 7.0 U2 Boot from SAN installation on a Gen10 Synergy Module using a kickstart and a OneView Server Profile Template] ***

TASK [Checking if HPE ESXi Custom ISO file exists on "ansible.lj.lab"] ************************************************************
ok: [ESX-1 -> localhost]

TASK [Creating the directory "/opt/esxiisosrc" to host the ISO file on "ansible.lj.lab"] ******************************************
ok: [ESX-1 -> localhost]

TASK [Downloading file "VMware-ESXi-7.0.2-17630552-HPE-702.0.0.10.6.5.27-Mar2021-Synergy.iso" to "ansible.lj.lab" if not present] ***
skipping: [ESX-1]

TASK [Checking if HPE ESXi Custom ISO file extraction is necessary on "ansible.lj.lab"] *******************************************
ok: [ESX-1 -> localhost]

TASK [Creating /mnt/ESX-1 on "ansible.lj.lab" if it does not exist] ***************************************************************
changed: [ESX-1 -> localhost]

TASK [Creating /opt/baremetal/ESX-1/ on "ansible.lj.lab" if it does not exist] ****************************************************
changed: [ESX-1 -> localhost]

TASK [Creating /opt/baremetal/temp/ESX-1/ on "ansible.lj.lab" if it does not exist] ***********************************************
changed: [ESX-1 -> localhost]

TASK [Creating /opt/baremetal/temp/ESX-1/etc/vmware/weasel on "ansible.lj.lab" if it does not exist] ******************************
changed: [ESX-1 -> localhost]

TASK [Mounting HPE ESXi Custom ISO and copying ISO files to /opt/baremetal/ESX-1/ on "ansible.lj.lab"] ****************************
changed: [ESX-1 -> localhost]

TASK [Modifying legacy bios SYSLINUX bootloader for kickstart installation from CDROM] ********************************************
changed: [ESX-1 -> localhost]

TASK [Modifying UEFI bootloader for kickstart installation from CDROM] ************************************************************
changed: [ESX-1 -> localhost]

TASK [Creating Server Profile "ESX-1" from Server Profile Template "ESXi7 BFS"] ***************************************************
changed: [ESX-1 -> localhost]

TASK [Capturing the boot information of the first fiber channel interface of the server profile] **********************************
ok: [ESX-1]

TASK [Capturing network set information from "Production_network_set" attached to the two production NICs] ************************
ok: [ESX-1 -> localhost]

TASK [Capturing the URI of network "Production_network_set" attached to the two production NICs] **********************************
ok: [ESX-1]

TASK [Capturing the server hardware name selected for Server Profile creation] ****************************************************
ok: [ESX-1]

TASK [Capturing MAC addresses of the production NICs attached to "Production_network_set" for subsequent configuration of the Distributed vSwitch.] ***
ok: [ESX-1]

TASK [Capturing LUN uri of the primary boot volume (if any) for the customization of the kickstart file] **************************
ok: [ESX-1]

TASK [Showing the result of the Server Profile creation task] *********************************************************************
ok: [ESX-1] => {
    "msg": "Hardware selected: Frame4, bay 4 - Result: Server Profile created."
}

TASK [Capturing boot volume information (if any)] *********************************************************************************
ok: [ESX-1 -> localhost]

TASK [Capturing boot LUN size defined in the Server Profile to ensure that OS will be installed on this disk using the kickstart file] ***
ok: [ESX-1]

TASK [Setting boot LUN size as 'undefined' if booting from local logical drive] ***************************************************
skipping: [ESX-1]

TASK [Creating kickstart file with %pre script to detect the "20GB" Boot From SAN volume if it exists] ****************************
changed: [ESX-1 -> localhost]

TASK [Preparing ks.cfg kickstart file to make the new ISO] ************************************************************************
changed: [ESX-1 -> localhost]

TASK [Creating the ks.tgz kickstart file to make the new ISO] *********************************************************************
changed: [ESX-1 -> localhost]

TASK [Copying new ks.tgz to /opt/baremetal/ESX-1/] ********************************************************************************
changed: [ESX-1 -> localhost]

TASK [Creating customized bootable ISO] *******************************************************************************************
changed: [ESX-1 -> localhost]

TASK [Creating /usr/share/nginx/html/isos/ on "ansible.lj.lab" if it does not exist] **********************************************
ok: [ESX-1 -> localhost]

TASK [Moving created ISO to the nginx default html folder of "ansible.lj.lab"] ****************************************************
changed: [ESX-1 -> localhost]

TASK [Powering on and booting "Frame4, bay 4" from created ISO using iLO Virtual Media] *******************************************
changed: [ESX-1 -> localhost]

TASK [Waiting for ESX installation to complete - waiting for "192.168.3.171" to respond...] ***************************************
ok: [ESX-1 -> localhost]

TASK [debug] **********************************************************************************************************************
ok: [ESX-1] => {
    "msg": "ESX-1 installation took 14 minutes"
}

TASK [Wait a little longer so that the ESX host is truly ready to be added to the vcenter] ****************************************
ok: [ESX-1 -> localhost]

TASK [Deleting all related files from staging location and web server] ************************************************************
changed: [ESX-1 -> localhost]

TASK [Adding ESXi host "esx-1.lj.lab" to vCenter "vcenter.lj.lab"] ****************************************************************
changed: [ESX-1 -> localhost]

TASK [Assigning ESXi license to Host] *********************************************************************************************
changed: [ESX-1 -> localhost]

TASK [Adding vmnic1 to standard switch vSwitch0] **********************************************************************************
changed: [ESX-1 -> localhost]

TASK [Adding vMotion Portgroup to standard switch vSwitch0] ***********************************************************************
changed: [ESX-1 -> localhost]

TASK [Gathering facts about vmnics] ***********************************************************************************************
ok: [ESX-1 -> localhost]

TASK [Capturing Production vmnics information for the distributed switch creation] ************************************************
ok: [ESX-1]

TASK [Connecting host to "DSwitch-VC100G" distributed switch] *********************************************************************
changed: [ESX-1 -> localhost]

TASK [Adding vmkernel mk1 port to "DSwitch-VC100G" distributed Switch] ************************************************************
changed: [ESX-1 -> localhost]

TASK [Changing Advanced Settings with Core Dump Warning Disable] ******************************************************************
changed: [ESX-1 -> localhost]

TASK [Setting the Power Management Policy to high-performance] ********************************************************************
changed: [ESX-1 -> localhost]

TASK [Configuring NTP servers] ****************************************************************************************************
changed: [ESX-1 -> localhost]

TASK [Starting NTP Service and set to start at boot.] *****************************************************************************
changed: [ESX-1 -> localhost]

TASK [Starting ESXi Shell Service and setting to enable at boot . . .] ************************************************************
[WARNING]: Found internal 'results' key in module return, renamed to 'ansible_module_results'.
changed: [ESX-1 -> localhost]

TASK [Starting SSH Service and setting to enable at boot.] ************************************************************************
changed: [ESX-1 -> localhost]

TASK [Displaying install completed message] ***************************************************************************************
ok: [ESX-1] => {
    "msg": [
        "ESX-1.lj.lab Installation completed !",
        "ESXi is configured and running. It has been added to the vCenter cluster 'Synergy Frame4'."
    ]
}

PLAY RECAP ************************************************************************************************************************
ESX-1                      : ok=48   changed=29   unreachable=0    failed=0    skipped=2    rescued=0    ignored=0

```

## Output sample of ESXi bare metal unprovisioning playbook

```
ansible-playbook -i hosts ESXi_unprovisioning.yml

PLAY [Deleting a provisioned ESXi compute module] *********************************************************************************

TASK [Taking "esx-1.lj.lab" to maintenance mode] **********************************************************************************
changed: [ESX-1 -> localhost]

TASK [Removing vmkernel mk1 port from "DSwitch-VC100G" distributed Switch] ********************************************************
changed: [ESX-1 -> localhost]

TASK [Gathering facts about vmnics] ***********************************************************************************************
ok: [ESX-1 -> localhost]

TASK [Capturing available vmnics for the distributed switch creation] *************************************************************
ok: [ESX-1]

TASK [Removing host from "DSwitch-VC100G" distributed Switch] *********************************************************************
changed: [ESX-1 -> localhost]

TASK [Removing ESXi host "esx-1.lj.lab" from vCenter "vcenter.lj.lab"] ************************************************************
changed: [ESX-1 -> localhost]

TASK [Getting server profile "ESX-1" information] *********************************************************************************
ok: [ESX-1 -> localhost]

TASK [Powering off server hardware "Frame4, bay 4"] *******************************************************************************
changed: [ESX-1 -> localhost]

TASK [Deleting server profile "ESX-1"] ********************************************************************************************
changed: [ESX-1 -> localhost]

TASK [Result of the task to delete the server profile] ****************************************************************************
ok: [ESX-1] => {
    "msg": "Deleted profile"
}

PLAY [Removing the DNS record for "{{ inventory_hostname }}"] *********************************************************************

TASK [Removing "192.168.3.171"" from "dc.lj.lab"] *********************************************************************************
changed: [ESX-1 -> dc.lj.lab]

PLAY RECAP ************************************************************************************************************************
ESX-1                      : ok=11   changed=7    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```

## Output sample of RHEL bare metal provisioning playbook

```
ansible-playbook -i hosts RHEL_provisioning.yml

PLAY [Creating a DNS record for the bare metal RHEL server] ************************************************************************************************************************************************************

TASK [Adding "RHEL-1" with "192.168.3.173" on "dc.lj.lab" in "lj.lab" DNS domain] **************************************************************************************************************************************
ok: [RHEL-1 -> dc.lj.lab]

PLAY [Performing an automated RHEL 8.3 Boot from SAN installation on a Gen10 Synergy Module using a kickstart and a OneView Server Profile Template] *******************************************************************

TASK [Checking if RHEL ISO file exists on "ansible.lj.lab"] ************************************************************************************************************************************************************
ok: [RHEL-1 -> localhost]

TASK [Creating the directory "/opt/rhelisosrc" to host the ISO file on "ansible.lj.lab"] *******************************************************************************************************************************
ok: [RHEL-1 -> localhost]

TASK [Downloading file "RHEL-8.3-minimum.iso" to "ansible.lj.lab" if not present] **************************************************************************************************************************************
skipping: [RHEL-1]

TASK [Collecting ISO label (can be required for some booltloader modifications)] ***************************************************************************************************************************************
changed: [RHEL-1 -> localhost]

TASK [set_fact] ********************************************************************************************************************************************************************************************************
ok: [RHEL-1]

TASK [Checking if RHEL ISO file extraction is necessary on "RHEL-1"] ***************************************************************************************************************************************************
ok: [RHEL-1 -> localhost]

TASK [Creating /mnt/RHEL-1 on "ansible.lj.lab" if it does not exist] ***************************************************************************************************************************************************
ok: [RHEL-1 -> localhost]

TASK [Creating /opt/baremetal/RHEL-1/ on "ansible.lj.lab" if it does not exist] ****************************************************************************************************************************************
ok: [RHEL-1 -> localhost]

TASK [Mounting RHEL ISO and copying ISO files to /opt/baremetal/RHEL-1/ on "ansible.lj.lab"] ***************************************************************************************************************************
skipping: [RHEL-1]

TASK [Modifying legacy bios SYSLINUX bootloader for kickstart installation from CDROM] *********************************************************************************************************************************
changed: [RHEL-1 -> localhost]

TASK [Modifying UEFI bootloader for kickstart installation from CDROM] *************************************************************************************************************************************************
changed: [RHEL-1 -> localhost]

TASK [Creating Server Profile "RHEL-1" from Server Profile Template "RHEL BFS"] ****************************************************************************************************************************************
changed: [RHEL-1 -> localhost]

TASK [Capturing the boot information of the first fiber channel interface of the server profile] ***********************************************************************************************************************
ok: [RHEL-1]

TASK [Capturing the server hardware name selected for Server Profile creation] *****************************************************************************************************************************************
ok: [RHEL-1]

TASK [Capturing LUN uri of the primary boot volume (if any) for the customization of the kickstart file] ***************************************************************************************************************
ok: [RHEL-1]

TASK [Showing the result of the Server Profile creation task] **********************************************************************************************************************************************************
ok: [RHEL-1] => {
    "msg": "Hardware selected: Frame4, bay 3 - Result: Server Profile created."
}

TASK [Capturing boot volume information (if any)] **********************************************************************************************************************************************************************
ok: [RHEL-1 -> localhost]

TASK [Capturing boot LUN size defined in the Server Profile to ensure that OS will be installed on this disk using the kickstart file] *********************************************************************************
ok: [RHEL-1]

TASK [Setting boot LUN size as 'undefined' if booting from local logical drive] ****************************************************************************************************************************************
skipping: [RHEL-1]

TASK [Creating kickstart file with %pre script to detect the "50GB" Boot From SAN volume if it exists] *****************************************************************************************************************
ok: [RHEL-1 -> localhost]

TASK [Creating customized bootable ISO] ********************************************************************************************************************************************************************************
changed: [RHEL-1 -> localhost]

TASK [Implanting MD5 checksum into the ISO to make it bootable] ********************************************************************************************************************************************************
changed: [RHEL-1 -> localhost]

TASK [Creating /usr/share/nginx/html/isos/ on "ansible.lj.lab" if it does not exist] ***********************************************************************************************************************************
ok: [RHEL-1 -> localhost]

TASK [Moving created ISO to the nginx default html folder of "ansible.lj.lab"] *****************************************************************************************************************************************
changed: [RHEL-1 -> localhost]

TASK [Powering on and booting "Frame4, bay 3" from created ISO using iLO Virtual Media] ********************************************************************************************************************************
changed: [RHEL-1 -> localhost]

TASK [Waiting for RHEL installation to complete - Waiting for "192.168.3.173" to respond...] ***************************************************************************************************************************
ok: [RHEL-1 -> localhost]

TASK [debug] ***********************************************************************************************************************************************************************************************************
ok: [RHEL-1] => {
    "msg": "RHEL-1 installation took 14 minutes"
}

TASK [Deleting all temporary files in the stagging location on "ansible.lj.lab"] ***************************************************************************************************************************************
changed: [RHEL-1 -> localhost]

TASK [Deleting created ISO file in the web server directory on "ansible.lj.lab"] ***************************************************************************************************************************************
changed: [RHEL-1 -> localhost]

TASK [Unmounting original ISO file on "ansible.lj.lab"] ****************************************************************************************************************************************************************
changed: [RHEL-1 -> localhost]

TASK [Copying HPE iSUT rpm file to RHEL-1] *****************************************************************************************************************************************************************************
changed: [RHEL-1]

TASK [Copying HPE AMS rpm file to RHEL-1] ******************************************************************************************************************************************************************************
changed: [RHEL-1]

TASK [Installing iSUT] *************************************************************************************************************************************************************************************************
changed: [RHEL-1]

TASK [Installing AMS] **************************************************************************************************************************************************************************************************
changed: [RHEL-1]

TASK [Waiting for iSUT installation to complete] ***********************************************************************************************************************************************************************
ok: [RHEL-1 -> localhost]

TASK [Configuring iSUT mode to allow OS driver updates via HPE OneView Server Profile] *********************************************************************************************************************************
changed: [RHEL-1]

TASK [debug] ***********************************************************************************************************************************************************************************************************
ok: [RHEL-1] => {
    "msg": "Set Mode: autodeploy\nService will be registered and started\nSUT Service started successfully\nRegistration successful"
}

TASK [Updating Server Profile to enable Firmware and OS Drivers using SUT] *********************************************************************************************************************************************
changed: [RHEL-1 -> localhost]

TASK [debug] ***********************************************************************************************************************************************************************************************************
ok: [RHEL-1] => {
    "msg": "Server profile updated"
}

TASK [Monitoring SUT status for 'reboot the system' message] ***********************************************************************************************************************************************************
FAILED - RETRYING: Monitoring SUT status for 'reboot the system' message (50 retries left).
FAILED - RETRYING: Monitoring SUT status for 'reboot the system' message (49 retries left).
FAILED - RETRYING: Monitoring SUT status for 'reboot the system' message (48 retries left).
FAILED - RETRYING: Monitoring SUT status for 'reboot the system' message (47 retries left).
FAILED - RETRYING: Monitoring SUT status for 'reboot the system' message (46 retries left).
FAILED - RETRYING: Monitoring SUT status for 'reboot the system' message (45 retries left).
FAILED - RETRYING: Monitoring SUT status for 'reboot the system' message (44 retries left).
FAILED - RETRYING: Monitoring SUT status for 'reboot the system' message (43 retries left).
FAILED - RETRYING: Monitoring SUT status for 'reboot the system' message (42 retries left).
FAILED - RETRYING: Monitoring SUT status for 'reboot the system' message (41 retries left).
FAILED - RETRYING: Monitoring SUT status for 'reboot the system' message (40 retries left).
FAILED - RETRYING: Monitoring SUT status for 'reboot the system' message (39 retries left).
FAILED - RETRYING: Monitoring SUT status for 'reboot the system' message (38 retries left).
FAILED - RETRYING: Monitoring SUT status for 'reboot the system' message (37 retries left).
ok: [RHEL-1 -> localhost]

TASK [Rebooting host for the HPE drivers/firmware activation and waiting for it to restart] ****************************************************************************************************************************
changed: [RHEL-1]

TASK [Displaying install completed message] ****************************************************************************************************************************************************************************
ok: [RHEL-1] => {
    "msg": [
        "RHEL-1.lj.lab Installation completed !",
        "OS is configured and running with HPE OS drivers."
    ]
}

PLAY RECAP *************************************************************************************************************************************************************************************************************
RHEL-1                     : ok=40   changed=18   unreachable=0    failed=0    skipped=3    rescued=0    ignored=0

```

## Output sample of RHEL bare metal unprovisioning playbook

```
ansible-playbook -i hosts RHEL_unprovisioning.yml

PLAY [Deleting provisioned RHEL compute module(s)] *********************************************************************************************************************************************************************

TASK [Getting server profile "RHEL-1" information] *********************************************************************************************************************************************************************
ok: [RHEL-1 -> localhost]

TASK [Powering off server hardware "Frame4, bay 3"] ********************************************************************************************************************************************************************
changed: [RHEL-1 -> localhost]

TASK [Deleting server profile "RHEL-1"] ********************************************************************************************************************************************************************************
changed: [RHEL-1 -> localhost]

TASK [Result of the task to delete the server profile] *****************************************************************************************************************************************************************
ok: [RHEL-1] => {
    "msg": "Deleted profile"
}

TASK [Removing RHEL-1 SSH key] *****************************************************************************************************************************************************************************************
changed: [RHEL-1 -> localhost]

PLAY [Removing the DNS record for "{{ inventory_hostname }}"] **********************************************************************************************************************************************************

TASK [Removing "192.168.3.173"" from "dc.lj.lab"] **********************************************************************************************************************************************************************
changed: [RHEL-1 -> dc.lj.lab]

PLAY RECAP *************************************************************************************************************************************************************************************************************
RHEL-1                     : ok=6    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```

## Output sample of Windows Server bare metal provisioning playbook

```
ansible-playbook -i hosts WIN_provisioning.yml

PLAY [Performing an unattended Windows Server 2022 Boot from SAN installation on a Gen10 Synergy Module using a OneView Server Profile Template] ***

TASK [Checking if Windows Server ISO file exists on "ansible.lj.lab"] *************************************************************
ok: [WIN-1 -> localhost]

TASK [Creating the directory "/opt/winisosrc" to host the ISO file on "ansible.lj.lab"] *******************************************
ok: [WIN-1 -> localhost]

TASK [Downloading file "en-us_windows_server_version_2022_updated_october_2021_x64_dvd_b6e25591.iso" to "ansible.lj.lab" if not present] ***
skipping: [WIN-1]

TASK [Checking if Windows Server ISO file extraction is necessary on "ansible.lj.lab"] ********************************************
ok: [WIN-1 -> localhost]

TASK [Creating /mnt/WIN-1 on "ansible.lj.lab" if it does not exist] ***************************************************************
changed: [WIN-1 -> localhost]

TASK [Creating /opt/baremetal/WIN-1/ on "ansible.lj.lab" if it does not exist] ****************************************************
changed: [WIN-1 -> localhost]

TASK [Mounting Windows Server ISO and copying ISO files to /opt/baremetal/WIN-1/ on "ansible.lj.lab"] *****************************
changed: [WIN-1 -> localhost]

TASK [Creating $OEM$ on "ansible.lj.lab" in /opt/baremetal/WIN-1/sources to execute scripts at startup] ***************************
changed: [WIN-1 -> localhost] => (item=/opt/baremetal/WIN-1/sources/$OEM$)
changed: [WIN-1 -> localhost] => (item=/opt/baremetal/WIN-1/sources/$OEM$/$1)
changed: [WIN-1 -> localhost] => (item=/opt/baremetal/WIN-1/sources/$OEM$/$1/Temp)
changed: [WIN-1 -> localhost] => (item=/opt/baremetal/WIN-1/sources/$OEM$/$$)
changed: [WIN-1 -> localhost] => (item=/opt/baremetal/WIN-1/sources/$OEM$/$$/Setup)
changed: [WIN-1 -> localhost] => (item=/opt/baremetal/WIN-1/sources/$OEM$/$$/Setup/Scripts)

TASK [Download POSH script from GitHub to configure Windows for remote management with Ansible] ***********************************
changed: [WIN-1 -> localhost]

TASK [Creating Server Profile "WIN-1" from Server Profile Template "Windows BFS"] *************************************************
changed: [WIN-1 -> localhost]

TASK [Capturing the server hardware name selected for Server Profile creation] ****************************************************
ok: [WIN-1]

TASK [Capturing MAC of first two management NICs for the configuration of the network settings in configure_network.ps1] **********
ok: [WIN-1]

TASK [Showing the result of the Server Profile creation task] *********************************************************************
ok: [WIN-1] => {
    "msg": "Hardware selected: Frame4, bay 5 - Result: Server Profile created."
}

TASK [Creating configure_network.ps1 that will be launched by SetupComplete.cmd (creation of a team using the first two NICs and configuration of IP parameters)] ***
changed: [WIN-1 -> localhost]

TASK [Creating SetupComplete.cmd for the network settings] ************************************************************************
changed: [WIN-1 -> localhost]

TASK [Updating autounattend.xml file] *********************************************************************************************
changed: [WIN-1 -> localhost]

TASK [Creating customized bootable ISO] *******************************************************************************************
changed: [WIN-1 -> localhost]

TASK [Creating /usr/share/nginx/html/isos/ on "ansible.lj.lab" if it does not exist] **********************************************
ok: [WIN-1 -> localhost]

TASK [Moving created ISO to the nginx default html folder of "ansible.lj.lab"] ****************************************************
changed: [WIN-1 -> localhost]

TASK [Powering on and booting "Frame4, bay 5" from created ISO using iLO Virtual Media] *******************************************
changed: [WIN-1 -> localhost]

TASK [Waiting for Windows Server installation to complete - Waiting for "192.168.3.175" to respond...] ****************************
ok: [WIN-1 -> localhost]

TASK [debug] **********************************************************************************************************************
ok: [WIN-1] => {
    "msg": "WIN-1 installation took 22 minutes"
}

TASK [Deleting all temporary files in the stagging location on "ansible.lj.lab"] **************************************************
changed: [WIN-1 -> localhost]

TASK [Deleting created ISO file in the web server directory on "ansible.lj.lab"] **************************************************
changed: [WIN-1 -> localhost]

TASK [Unmounting original ISO file on "ansible.lj.lab"] ***************************************************************************
changed: [WIN-1 -> localhost]

TASK [Collecting product_id found in install.xml file of the HPE iSUT package] ****************************************************
changed: [WIN-1 -> localhost]

TASK [Collecting product_id found in install.xml file of the HPE AMS package] *****************************************************
changed: [WIN-1 -> localhost]

PLAY [Creating a DNS record for the bare metal Windows Server] ********************************************************************

TASK [Adding "WIN-1" with "192.168.3.175" on "dc.lj.lab" in "lj.lab" DNS domain] **************************************************
changed: [WIN-1 -> dc.lj.lab]

PLAY [Installing HPE iSUT and HPE AMS on the server for online installation of HPE drivers for Windows Server] ********************

TASK [Copying HPE iSUT package file to WIN-1] *************************************************************************************
changed: [WIN-1]

TASK [Copying HPE AMS package file to WIN-1] **************************************************************************************
changed: [WIN-1]

TASK [Installing Integrated Smart Update Tools] ***********************************************************************************
ok: [WIN-1]

TASK [Installing HPE Agentless Management Service] ********************************************************************************
ok: [WIN-1]

TASK [Configuring iSUT mode to allow OS driver updates via HPE OneView Server Profile] ********************************************
changed: [WIN-1]

TASK [debug] **********************************************************************************************************************
ok: [WIN-1] => {
    "msg": "Set Mode: autodeploy\r\nService will be registered and started\r\nSUT Service started successfully\r\nRegistration successful\r\n"
}

TASK [Updating Server Profile to enable Firmware and OS Drivers using SUT] ********************************************************
changed: [WIN-1 -> localhost]

TASK [debug] **********************************************************************************************************************
ok: [WIN-1] => {
    "msg": "Server profile updated"
}

TASK [Joining domain lj.lab] ******************************************************************************************************
changed: [WIN-1]

TASK [Monitoring SUT status for 'reboot the system' message] **********************************************************************
FAILED - RETRYING: Monitoring SUT status for 'reboot the system' message (50 retries left).
FAILED - RETRYING: Monitoring SUT status for 'reboot the system' message (49 retries left).
FAILED - RETRYING: Monitoring SUT status for 'reboot the system' message (48 retries left).
FAILED - RETRYING: Monitoring SUT status for 'reboot the system' message (47 retries left).
FAILED - RETRYING: Monitoring SUT status for 'reboot the system' message (46 retries left).
FAILED - RETRYING: Monitoring SUT status for 'reboot the system' message (45 retries left).
FAILED - RETRYING: Monitoring SUT status for 'reboot the system' message (44 retries left).
FAILED - RETRYING: Monitoring SUT status for 'reboot the system' message (43 retries left).
FAILED - RETRYING: Monitoring SUT status for 'reboot the system' message (42 retries left).
FAILED - RETRYING: Monitoring SUT status for 'reboot the system' message (41 retries left).
FAILED - RETRYING: Monitoring SUT status for 'reboot the system' message (40 retries left).
FAILED - RETRYING: Monitoring SUT status for 'reboot the system' message (39 retries left).
FAILED - RETRYING: Monitoring SUT status for 'reboot the system' message (38 retries left).
FAILED - RETRYING: Monitoring SUT status for 'reboot the system' message (37 retries left).
FAILED - RETRYING: Monitoring SUT status for 'reboot the system' message (36 retries left).
FAILED - RETRYING: Monitoring SUT status for 'reboot the system' message (35 retries left).
ok: [WIN-1 -> localhost]

TASK [Rebooting host for the HPE drivers/firmware activation and waiting for it to restart] ***************************************
changed: [WIN-1]

TASK [Displaying install completed message] ***************************************************************************************
ok: [WIN-1] => {
    "msg": [
        "WIN-1.lj.lab Installation completed !",
        "OS is configured and running with HPE OS drivers."
    ]
}

PLAY RECAP ************************************************************************************************************************
WIN-1                      : ok=39   changed=24   unreachable=0    failed=0    skipped=1    rescued=0    ignored=0

```

## Output sample of Windows Server bare metal unprovisioning playbook

```
ansible-playbook -i hosts WIN_unprovisioning.yml

PLAY [Deleting provisioned Windows Server compute module(s)] **********************************************************************

TASK [Getting server profile "WIN-1" information] *********************************************************************************
ok: [WIN-1 -> localhost]

TASK [Powering off server hardware "Frame4, bay 5"] *******************************************************************************
changed: [WIN-1 -> localhost]

TASK [Deleting server profile "WIN-1"] ********************************************************************************************
changed: [WIN-1 -> localhost]

TASK [Result of the task to delete the server profile] ****************************************************************************
ok: [WIN-1] => {
    "msg": "Deleted profile"
}

PLAY [Removing the DNS record for "{{ inventory_hostname }}"] *********************************************************************

TASK [Removing "192.168.3.175"" from "dc.lj.lab"] *********************************************************************************
changed: [WIN-1 -> dc.lj.lab]

PLAY RECAP ************************************************************************************************************************
WIN-1                      : ok=5    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```

## Thank you

Thank you to bryansullins for his inspiring work for [baremetalesxi](https://github.com/bryansullins/baremetalesxi).

## License

This project is licensed under the MIT License - see the LICENSE file for details
