---
layout: post
title:  "Creating and Deploying VM Templates with Packer and Powershell"
date:   2018-07-07
author: Jonathan Moss
permalink: blog/automating-template-builds-with-packer.html
tags: powershell packer vcenter
---

__Updated: 12/15/2018__

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
├── Setup
│   ├── Enable-WinRM.ps1
│   ├── vmtools.cmd
│   ├── autounattend.xml
├── Binaries
│   ├── packer.exe
│   ├── packer-builder-vsphere-clone.exe
│   ├── packer-builder-vsphere-iso.exe
├── README.md
├── Scripts
│   ├── Install-Boxstarter.ps1
│   ├── Install-Chocolatey.ps1
│   ├── Invoke-Cleanup.ps1
│   ├── Invoke-SystemCleanup.ps1
│   ├── Invoke-SystemUpdate.ps1
│   ├── Set-ChocolateySettings.ps1
│   ├── Set-Timezone.ps1
├── ConfigurationFiles
│   ├── server-2016.json
```

In a datastore of your choosing in vCenter, create a folder called `ISO` and upload the following:

* Server 2016 ISO `14393.0.161119-1705.RS1_REFRESH_SERVER_EVAL_X64FRE_EN-US.ISO`
* VMware Tools ISO `Windows.iso`
* Windows floppy image `pvscsi-Windows8.flp`

Once you've done that, create your autounattend.xml file and store in your git repository. You can use [my customized autounattend.xml file](https://github.com/jwmoss/powershell_scripts/blob/master/Packer/Setup/autounattend.xml) as an example, as I've taken cues from several repositories and from [JetBrain's Repo](https://github.com/jetbrains-infra/packer-builder-vsphere/tree/master/examples/windows/setup), or build your own.

Once you've created the autounattend.xml file, uploaded the files neceessary to the vCenter Datastore, you're finally ready to begin cusotmizing the [Packer Template](https://www.packer.io/docs/templates/index.html) which are JSON files that configure the various components of Packer in order to create one or more machine images.

Here's an example JSON file that I used with Server 2016. The latest version will be stored in my [Github Repository](https://github.com/jwmoss/powershell_scripts/tree/master/Packer).

Once you've created the JSON file, and have all parts in place, simply execute `Packer.exe` from your workstation. I have an example of doing it from Jenkins:

```
function Invoke-TemplateBuild {
    param (
        [Parameter(Mandatory = $true)]
        [string]$vCenterServer,
        [Parameter(Mandatory = $true)]
        [string]$TemplateFolder,
        [Parameter(Mandatory = $true)]
        [string]$TemplateName,
        [Parameter(Mandatory = $true)]
        [string]$ConfigurationFile
    )
 
    <#
.SYNOPSIS
    Executes Packer Template Build against a vCenter. This script is designed to run from Jenkins.
.EXAMPLE
    PS C:\> Invoke-TemplateBuild.ps1 -vCenter vcenter.fqdn.com -TemplateFolder TemplateFolder -TemplateName Win2016 -ConfigurationFile Server2016.json
    Executes packer.exe, connects to a vCenter and moves the template to the correct folder.
.NOTES
    v1.0.0 - 7/9/18 - Jonathan Moss
    v1.0.1 - 7/11/18 - Remove existing template prior to deploying new one
    v1.0.2 - 10/22/18 - Added parameters to support targetting multiple vCenters
#>

    Import-Module VMware.PowerCLI | Out-Null

    ## Define Jenkins parameters and pass through credentials stored in Jenkins using Credentials Binding plugin.
    $SecurePassword = $env:vCenterPassword | ConvertTo-SecureString -AsPlainText -Force
    $vCenterCred = New-Object System.Management.Automation.PSCredential -ArgumentList $env:vCenterAdmin, $SecurePassword

    Set-PowerCLIConfiguration -InvalidCertificateAction Ignore -Confirm:$false

    ## Connect to vCenter with stored credentials
    try {
        Connect-VIServer -Server $vCenterServer -Protocol https -Credential $vCenterCred -ErrorAction Stop
        Write-Output "Successfully connected to $vcenterServer! Continuing.."   
    }
    catch {
        Write-Output "Could not connect to $vcenterserver"
        Write-Host $_.Exception.Message
        Write-Host $_.Exception.ItemName
    }

    try {
        $VMTemplate = Get-Template -Name $TemplateName -ErrorAction STOP
        if ($VMTemplate) {
            try {
                Remove-Template -Template $VMTemplate -Confirm:$false -ErrorAction STOP
            }
            catch {
                ## If template is found, yet can't be removed, exit the script.
                Write-Output "Could not delete, skipping..."
                Write-Output "Manually delete $VMTemplate from $vcenterServer"
                Break
            }
        }   
    }
    catch {
        Write-Host "No template found! Continuing.."
        Write-Host $_.Exception.Message
        Write-Host $_.Exception.ItemName
    }

    $params = @{
        FilePath         = "$ENV:WORKSPACE\Binaries\packer.exe"
        ArgumentList     = "build -var `"password=$($env:vCenterPassword)`" -var `"winrm_password=$($env:LocalAdminPassword)`" `"$($ENV:WORKSPACE)\ConfigurationFiles\$ConfigurationFile`""
        WorkingDirectory = "$ENV:WORKSPACE\Packer"
    }

    Start-Process @params -NoNewWindow -Wait

    ## Get existing folder where templates are stored

    $TemplateFolder = Get-Folder -Name $TemplateFolder

    ## Move template that was created from script to the destination template folder
    $a = Get-Template -Name $TemplateName
    $b = Get-Folder -Name $TemplateFolder
    Get-Template $a | Move-Template -Destination $b

    ## Disconnect from vCenter
    Disconnect-VIServer $vCenterServer -Confirm:$False
}
```

Once you do this, this will create a Server 2016 template and save it as `Template_Win2016_Std_64bit_Prod` within vCenter.

One of the extra things I did was take cues from [DARKSURGEON](https://darksurgeon.io/) with customizing the OS, but the world is your oyster when building out a customized server image.

Calling a Powershell script is done by using a Packer Provisioner, specifically the [Powershell Provisioner](https://www.packer.io/docs/provisioners/powershell.html).

By this point, you should have a completely automated method of deploying the latest Server 2016 image to vCenter, and convert it to a template using Packer and Powershell.

Feel free to check out the [repository](https://github.com/jwmoss/powershell_scripts/tree/master/Packer) where I store all of this code.