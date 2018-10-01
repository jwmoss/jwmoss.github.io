---
layout: post
title:  "Creating and Deploying VM Templates with Packer and Powershell"
date:   2018-07-07
author: Jonathan Moss
permalink: blog/automating-template-builds-with-packer.html
---

I stumbled upon an issue at work which allowed for an opportunity to build virtual machine (VM) templates within vCenter. I wanted to find a way to automate the process because there are lots of ways to screw up a VM template, and if you screw it up in the beginning, the issue(s) will be present through the lifecycle of the Server. No fun!

Here are the things required in order to automate the creation of a VM template:

* Packer
* [JetBrain's Packer Plugin for vSphere](https://github.com/jetbrains-infra/packer-builder-vsphere)
* [JetBrain's Packer Executable](https://github.com/jetbrains-infra/packer-builder-vsphere/releases)
* Windows Server 2016 ISO (Evaluation copy works)
* Paravirtual SCSI floppy image for Windows 8, [found on your ESXi host](https://kb.vmware.com/s/article/1010398)
* VMware Tools ISO (I've renamed to Windows.iso for ease of use)

Let's start!

I begun by creating a repository with the following structure:

```bash
├── setup
│   ├── Enable-WinRM.ps1
│   ├── vmtools.cmd
│   ├── autounattend.xml
├── binaries
│   ├── packer.exe
│   ├── packer-builder-vsphere-clone.exe
│   ├── packer-builder-vsphere-iso.exe
├── README.md
├── server-2016.json
├── build.ps1
```

In a datastore of your choosing in vCenter, create a folder called `ISO` and upload the following:

* Server 2016 ISO `14393.0.161119-1705.RS1_REFRESH_SERVER_EVAL_X64FRE_EN-US.ISO`
* VMware Tools ISO `Windows.iso`
* Windows floppy image `pvscsi-Windows8.flp`

Once you've done that, create your autounattend.xml file and store in your git repository. You can use [my customized autounattend.xml file](https://github.com/jwmoss/Packer/blob/master/Setup/autounattend.xml) as an example, as I've taken cues from several repositories and from [JetBrain's Repo](https://github.com/jetbrains-infra/packer-builder-vsphere/tree/master/examples/windows/setup), or build your own.

Here's an example of what the autounattend.xml file should look like.

```xml
<?xml version="1.0" encoding="utf-8"?>
<unattend xmlns="urn:schemas-microsoft-com:unattend">
    <settings pass="windowsPE">
        <component name="Microsoft-Windows-International-Core-WinPE" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS" xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
            <UILanguage>en-US</UILanguage>
        </component>

        <component name="Microsoft-Windows-PnpCustomizationsWinPE" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS" xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
            <DriverPaths>
                <PathAndCredentials wcm:action="add" wcm:keyValue="A">
                    <Path>B:\</Path>
                </PathAndCredentials>
            </DriverPaths>
        </component>

        <component name="Microsoft-Windows-Setup" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS" xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
            <UserData>
                <AcceptEula>true</AcceptEula>
            </UserData>

            <ImageInstall>
                <OSImage>
                    <InstallFrom>
                        <MetaData wcm:action="add">
                            <Key>/IMAGE/NAME</Key>
                            <Value>Windows Server 2016 SERVERSTANDARD</Value>
                        </MetaData>
                    </InstallFrom>
                    <InstallToAvailablePartition>true</InstallToAvailablePartition>
                </OSImage>
            </ImageInstall>

            <DiskConfiguration>
                <Disk wcm:action="add">
                    <DiskID>0</DiskID>
                    <CreatePartitions>
                        <CreatePartition wcm:action="add">
                            <Order>1</Order>
                            <Extend>true</Extend>
                            <Type>Primary</Type>
                        </CreatePartition>
                    </CreatePartitions>
                </Disk>
            </DiskConfiguration>
        </component>
    </settings>

    <settings pass="offlineServicing">
        <component name="Microsoft-Windows-LUA-Settings" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS" xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
            <!-- Disable user account control -->
            <EnableLUA>false</EnableLUA>
        </component>
    </settings>

    <settings pass="specialize">
        <component name="Microsoft-Windows-Deployment" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS" xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
            <RunSynchronous>
                <RunSynchronousCommand wcm:action="add">
                    <Order>1</Order>
                    <!-- Install VMware Tools from windows.iso -->
                    <Path>a:\vmtools.cmd</Path>
                    <WillReboot>Always</WillReboot>
                </RunSynchronousCommand>
            </RunSynchronous>
        </component>
    </settings>

    <settings pass="oobeSystem">
        <component xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" name="Microsoft-Windows-Shell-Setup" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS">
            <AutoLogon>
                <Password>
                    <Value>admin</Value>
                    <PlainText>true</PlainText>
                </Password>
                <Enabled>true</Enabled>
                <Username>admin</Username>
            </AutoLogon>
            <FirstLogonCommands>
                <SynchronousCommand wcm:action="add">
                    <CommandLine>cmd.exe /c powershell -Command "Set-ExecutionPolicy -ExecutionPolicy Unrestricted -Force"</CommandLine>
                    <Description>Set Execution Policy 64 Bit</Description>
                    <Order>1</Order>
                    <RequiresUserInput>true</RequiresUserInput>
                </SynchronousCommand>
                <SynchronousCommand wcm:action="add">
                    <CommandLine>cmd.exe /c C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe -File a:\Enable-WinRM.ps1</CommandLine>
                    <Description>Enable WinRM and Reset AutoLogon Key</Description>
                    <Order>2</Order>
                    <RequiresUserInput>true</RequiresUserInput>
                </SynchronousCommand>
            </FirstLogonCommands>
            <OOBE>
                <HideEULAPage>true</HideEULAPage>
                <HideLocalAccountScreen>true</HideLocalAccountScreen>
                <HideOEMRegistrationScreen>true</HideOEMRegistrationScreen>
                <HideOnlineAccountScreens>true</HideOnlineAccountScreens>
                <HideWirelessSetupInOOBE>true</HideWirelessSetupInOOBE>
                <NetworkLocation>Home</NetworkLocation>
                <ProtectYourPC>1</ProtectYourPC>
            </OOBE>
            <UserAccounts>
                <AdministratorPassword>
                    <Value>admin</Value>
                    <PlainText>true</PlainText>
                </AdministratorPassword>
                <LocalAccounts>
                    <LocalAccount wcm:action="add">
                        <Password>
                            <Value>admin</Value>
                            <PlainText>true</PlainText>
                        </Password>
                        <Group>administrators</Group>
                        <DisplayName>admin</DisplayName>
                        <Name>admin</Name>
                        <Description>Admin User</Description>
                    </LocalAccount>
                </LocalAccounts>
            </UserAccounts>
            <RegisteredOwner/>
        </component>
    </settings>
</unattend>
```

Once you've created the autounattend.xml file, uploaded the files neceessary to the vCenter Datastore, you're finally ready to begin cusotmizing the [Packer Template](https://www.packer.io/docs/templates/index.html) which are JSON files that configure the various components of Packer in order to create one or more machine images.

Here's an example JSON file that I used with Server 2016. The latest version will be stored in my [Packer Github Repository](https://github.com/jwmoss/Packer).

```json
{
    "builders": [
        {
            "type": "vsphere-iso",
            "vcenter_server": "vCenter.fqdn.com",
            "username": "domain\\user",
            "password": "{{ user `password` }}",
            "insecure_connection": "true",
            "vm_version": 13,
            "vm_name": "Template_Win2016_Std_64bit_Prod",
            "guest_os_type": "windows9Server64Guest",
            "communicator": "winrm",
            "cluster": "cluster1",
            "datacenter": "datacenter1",
            "host": "esxihost.fqdn.com",
            "datastore": "datastore1",
            "network": "Virtual Switch 1",
            "winrm_username": "admin",
            "winrm_password": "{{ user `winrm_password` }}",
            "winrm_port": 5985,
            "CPUs": 2,
            "RAM": 4096,
            "convert_to_template": "true",
            "CPU_limit": -1,
            "boot_order": "disk,cdrom",
            "RAM_reserve_all": true,
            "disk_controller_type": "pvscsi",
            "disk_size": 71680,
            "disk_thin_provisioned": true,
            "network_card": "vmxnet3",
            "iso_paths": [
                "[datastore1] ISO/14393.0.161119-1705.RS1_REFRESH_SERVER_EVAL_X64FRE_EN-US.ISO",
                "[datastore1] ISO/Windows.iso"
            ],
            "floppy_files": [
                "./setup/"
            ],
            "floppy_img_path": "[datastore1] ISO/pvscsi-Windows8.flp"
        }
    ],
    "provisioners": [
        {
            "type": "powershell",
            "scripts": [
                "./Apply-Updates.ps1"
            ]
        }
    ]
}
```

Once you've created the JSON file, and have all parts in place, simply execute `Packer.exe` from your workstation:

```powershell
$params = @{
    FilePath = ".\binaries\packer.exe"
    ArgumentList = "build -var `"password=vcenterpassword`" -var `"winrm_password=localadminpassword`" `".\server-2016-prod.json`""
}

Start-Process @params -NoNewWindow -Wait
```

Once you do this, this will create a Server 2016 template and save it as `Template_Win2016_Std_64bit_Prod` within vCenter.

One of the extra things I did was take this `Get-LatestUpdate.ps1` script from [Aaron Parker's Github](https://github.com/aaronparker/MDT/blob/master/Updates/Get-LatestUpdate.ps1) and modify it to run inside the VM template build so that each time the template is built, the latest Windows Cumulative Update is applied. Here's my `Apply-Updates.ps1` [script](https://github.com/jwmoss/Packer/blob/master/Apply-Updates.ps1) that I called from within Packer.

Calling a Powershell script is done by using a Packer Provisioner, specifically the [Powershell Provisioner](https://www.packer.io/docs/provisioners/powershell.html).

By this point, you should have a completely automated method of deploying the latest Server 2016 image to vCenter, and convert it to a template using Packer and Powershell. I've added the `jenkins-build.ps1` [script](https://github.com/jwmoss/Packer/blob/master/jenkins-build.ps1) to my github repo that I use within Jenkins to automate this process each month, although I still need to update it to remove the template upon execution (but you get the idea).