---
# published: true
# layout: post
# date: 2017-10-06T00:00:00.000Z
# comments: true
description: A Windows Docker Container Series (Part 1)
category: docker
tags:
  - docker
  - windows
  - packer
title: A Windows Docker Container Series (Part 1)
---
#### Goals
In order to begin my journey into exploring Docker, I felt the need to incorporate as many of the [12 Factor principles](https://12factor.net) as I could into my development pipeline. I also wanted to have a clean place to get started on a Docker workflow, that was isolated from my normal .NET Development environment so as not to conflict, or pickup and unneeded dependencies. I had heard a lot about Hashicorp, and the tools they build for “provisioning, securing, connecting, and running any infrastructure for any application”. So, I decided I wanted to start there, with a fresh Packer Image that could be built, and reused from “scratch” as many times as I would need.

#### An Introduction to Packer
Packer is an open source tool that was made to create machine images (“single static unit that contains a pre-configured operating system and installed software which is used to quickly create new running machines”) for multiple platforms from a single .json configuration file. It allows for faster deployments because provisioning time is eliminated, by allowing machines to pre-provisioned and configured, ahead of time, by Packer. This can also improve stability of deployment because Packer installs and configures all software for a machine when the image is built. If there are any issues with install scripts or configurations, they'll be caught early, rather than several minutes after a machine is launched. Due to nature of how Packer creates machine images it allows for identical images to be created for multiple platforms (GCP, AWS, OpenStack, VMware, VirtualBox, HyperV, etc.), as transitioning between them can be nearly seamless.
#### Principle #10
12 Factor principle #10 is on the “Dev/Prod parity”, and by keeping any images used identical between environments, we can take a large step towards reducing the gap that commonly exists. The. json which describes the machine images can also be used in a CI/CD scenario to have a blank machine created with each new build as a way to prevent configuration drift.

### Why does it matter?
I believe this can be a crucial step in reducing the “works on my machine but not on his/hers” issue that is common in development. I also feel that in the context of Agency business, it would be unreasonable to try and have someone maintain VMs for each client’s environment (especially at scale), in order to cater to their specific needs or requirements. Therefore, we end up with shared environments. With Packer, it would be much easier to maintain a few Packer files in the client’s repo that could be an exact environment, which matches prod, for any developer joining the project, no background knowledge required.

### Walkthrough
#### Windows Requirements
To start off, I began learning about what would be required to run Docker Containers on Windows. Windows Containers are offered with two different base images, Windows Server Core and Nano Server. There are also 2 different mythologies supported for running containers “Windows Server Container” and “Hyper-V Container”. Windows Server Containers share the kernel between containers and the host, therefore mismatched versions are not supported. Hyper-V Containers utilize their own instance of the Windows kernel so you can miss-match the container host and container image versions. That lead me into determining which Windows version would support which of the different base-images, and container methodologies. Microsoft offer the following table under their [“Windows Container Requirements” documentation](https://docs.microsoft.com/en-us/virtualization/windowscontainers/deploy-containers/system-requirements):

| Host Operating System                    | Windows Server Container  |     Hyper-V Container     |
| ---------------------------------------- | :-----------------------: | :-----------------------: |
| Windows Server 2016 (Standard or Datacenter) | Server Core / Nano Server | Server Core / Nano Server |
| Nano Server                              |        Nano Server        | Server Core / Nano Server |
| Windows 10 Pro / Enterprise              |       Not Available       | Server Core / Nano Server |

Based on this chart I felt it would be best use a workflow centered around Windows Server 2016, as it had the broadest support, covering all variations, of running Docker Containers on Windows.
After determining I wanted to use Windows Server 2016 as my staring point I began to browse GitHub to see if I could find any existing Windows Server 2016 Packer templates.

#### Packer Template
[Packer.io](https://www.packer.io/docs/builders/hyperv-iso.html) had the below template for getting started with Hyper-V (which is what comes by default in Windows 10 Enterprise, so that's what I went with):

```javascript
{
  "builders": [
    {
      "vm_name":"windows2012r2",
      "type": "hyperv-iso",
      "disk_size": 61440,
      "floppy_files": [],
      "secondary_iso_images": [
        "./windows/windows-2012R2-serverdatacenter-amd64/answer.iso"
      ],
      "http_directory": "./windows/common/http/",
      "boot_wait": "0s",
      "boot_command": [
        "a<wait>a<wait>a"
      ],
      "iso_url": "http://download.microsoft.com/download/6/2/A/62A76ABB-9990-4EFC-A4FE-C7D698DAEB96/9600.16384.WINBLUE_RTM.130821-1623_X64FRE_SERVER_EVAL_EN-US-IRM_SSS_X64FREE_EN-US_DV5.ISO",
      "iso_checksum_type": "md5",
      "iso_checksum": "458ff91f8abc21b75cb544744bf92e6a",
      "communicator":"winrm",
      "winrm_username": "vagrant",
      "winrm_password": "vagrant",
      "winrm_timeout" : "4h",
      "shutdown_command": "f:\\run-sysprep.cmd",  
      "ram_size": 4096,
      "cpu": 4,
      "generation": 2,
      "switch_name":"LAN",
      "enable_secure_boot":true
    }
  ],
  "provisioners": [
    {
      "type": "powershell",
      "elevated_user":"vagrant",
      "elevated_password":"vagrant",
      "scripts": [
        "./windows/common/install-7zip.ps1",
        "./windows/common/install-chef.ps1",
        "./windows/common/compile-dotnet-assemblies.ps1",
        "./windows/common/cleanup.ps1",
        "./windows/common/ultradefrag.ps1",
        "./windows/common/sdelete.ps1"
      ]
    }
  ],
  "post-processors": [
    {
      "type": "vagrant",
      "keep_input_artifact": false,
      "output": "{{.Provider}}_windows-2012r2_chef.box"
    }
  ]
}
```
Also, want to give credit to the most helpful public Github repo I came across from Stefan Scherer: [Packer Windows](https://github.com/StefanScherer/packer-windows).

Since this file is the main configuration file, I will do my best to explain what it's doing.

The first section defines a builder object. Builders are how you write definitions for creating different machines, and for generating an image for plaforms such as Azure, Docker, EC2, VMware, VirtualBox, and of course, Hyper-V.

Within the builder, there are many parameters defined, without going into them all indivdually, they provide Packer with the information needed to create a machine image via the command-line on the specified platform. In this case we are creating a Windows Server 2016 image so we need to provide Packer all the variables need to perform a ["silent install"](https://msdn.microsoft.com/en-us/library/ee251019%28v=bts.10%29.aspx) of the OS.

The next section defines a provisioner object. According to Packer documentation, "Provisioners use builtin, and third-party software, to install, and configure, the machine image after booting to prepare the system for use". Commom uses are:
* installing packages (zip utility, windows features)
* patching the kernel (installing Windows updates
* creating users (creating the default 'vagrant' user)
* downloading application code (Docker)

The final section defines an optional post-processor object. Post-processers are used to do things with your image after they have been prepeared. In my case, like the example from Packer above also, I chose to use the [Vagrant post-processor](https://www.packer.io/docs/post-processors/vagrant.html), which prepares the Hyper-V VM for use with another Hashicorp tool [Vagrant](https://www.vagrantup.com) in conjunction with Hyper-V.

#### Running Packer
The brilliance of Packer is in it's simplicity. Once the configuration is complete, all one has to do to create the machine image is the following command from PowerShell using elevated privileges:
```powershell
packer build -var 'hyperv_switchname=Ethernet' .\windows_2016_docker.json
```
I specified as a variable my exisitng Hyper-V switch, so that when the machine boots during the install, it is able to connect to the internet for Windows Updates (optional), and various other tasks in the provisioning step.

### Next Steps
The next steps I would like to take with packer are (in no particular order):
* continuing to add provisiong scripts which add my presonal preferences to the Windows Server machine.
* create new builders for other plaforms so I can ensure my machine image is able to run anywhere
* use my [Packer Image Builds in a full CI/CD workflow](https://cloud.google.com/solutions/automated-build-images-with-jenkins-kubernetes) with Jenkins and Kubernetes


Hope you learned something with me; till next time.
