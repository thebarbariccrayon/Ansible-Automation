# Ansible Automation Platform Execution Environments
This repo contains the configurations and instructions on how to create and publish executions environments in the Automation Hub that will be used in the Ansible Automation Platform

## Introduction
The following document is designed to guide a use in the creation of supported and customized execution environments in the Ansible Automation Platform? This process does cover basice usage of Podman. This is Red Hats version of Docker with many of the same commands. 

## Requirements
- Basic understanding of containers and tools used to manage and build containers like Docker.
- SSH access to the Ansible Automation infrastructure. 
- Adminstrative rights to the AAP infrastructure.  

# Getting Started

## For Prodution Environment

Log into a designated system, and using sudo become the user 'ansible'.
```
@builder-system ~]$ sudo su - ansible
```
Then navigate to the directory /home/ansible/execution_environments
```
@builder-system ~]$ cd /home/ansible/execution_environments/  
```
If this doesn't exist then create the directory. You may name it what ever you want. Its your build environment.

# Container / EE Developement 

## Pulling a Container Image

Red Hat provides a catalog of base and bundled supported images that be found [here](https://catalog.redhat.com/software/containers/search?p=1&build_categories_list=Automation%20Execution%20Environment)  
Check if there are any container images present:   
```
[builder-system ~]$ podman image list
REPOSITORY  TAG         IMAGE ID    CREATED     SIZE
```
Log into the Red Hat supported image registry:  
```
[builder-system ~]$ podman login registry.redhat.io
Username: **********
Password:
Login Succeeded!
```
Pull the container image you wish to use:  
```
[ansible@builder-system ~]$ podman pull registry.redhat.io/ansible-automation-platform-21/ee-supported-rhel8:latest
Trying to pull registry.redhat.io/ansible-automation-platform-21/ee-supported-rhel8:latest...
Getting image source signatures
Checking if image destination supports signatures
Copying blob 8eb59a5a5023 done
Copying blob 38b71301a1d9 done
Copying blob 9af5bc7de6e7 done
Copying blob 0a7de2f4bd66 done
Copying blob a9e23b64ace0 done
Copying config 9c416fb667 done
Writing manifest to image destination
Storing signatures
9c416fb66747c066d091486eec35e42d19b0a72a5ef3ee1bbcf96af9ee34858d
```
Check the results:  
```
[ansible@builder-system ~]$ podman image list
REPOSITORY                                                            TAG         IMAGE ID      CREATED     SIZE
registry.redhat.io/ansible-automation-platform-21/ee-supported-rhel8  latest      9c416fb66747  3 days ago  1.17 GB
```
## Configuring a custom Execution Environment
Create a directory for the custom EE you wish to build. For this example we will be building an EE that conatains the Red Supported Zabbix modules provided in Ansible Galaxy

### Required Files
- execution-environment.yml --> Introdution file used by ansible builder 
- requirements.yml --> File used to specify collections and roles installed by ansible-galaxy
- requirements.txt --> File used to specify what python packages to install

### Optional Files
- system-req.txt --> This file can be used to install DNF packages that may be needed
- ansible.cfg  --> This file sets ansible configs for this environment
```
[ansible@builder-system execution_environments]$ mkdir zabbix_supported_ee && cd zabbix_supported_ee/
```
The first file that will be created will be the execution-environment.yml file. This is the introdution file that ansible-builder will use to determine what components to install.

## Ansible Builder
Once you have defined the requirements in your execution environment run:
```
[ansible@builder-system execution_environments]$ ansible-builder build --tag <custom-image-tag>
Running command:
  podman build -f context/Containerfile -t <custom-image-tag>
Complete! The build context can be found at: /home/ansible/execution_environments/<custom-image-tag>/context
```

## Publishing to Ansible Controller
After you have finished building a local image you will still have to take the following steps to create a manifest abd publish it for use in Ansible Controller:

First you will need to create a manifest registry to push the new image too.
For this example we will be using the zabbix supported EE we just built
```
[ansible@builder-system zabbix_supprted_ee]$ podman manifest create builder-system.mydomain.com/repo-name/<custom-image-tag>:latest
```
After which you will see a manifest created in the image list:
```
[ansible@builder-system ~]$ podman image list
REPOSITORY                                                            TAG         IMAGE ID      CREATED       SIZE
builder-system.mydomain.com/repo-name/<custom-image-tag>                    latest      e3add6771088  2 hours ago   110 B
```
Once you have verified that push the image to the manifest for export:
```
[ansible@aap-hub02 ~]$ podman push builder-system.mydomain.com/repo-name/<custom-image-tag>:v1.0.0 docker://builder-system.mydomain.com/repo-name/<custom-image-tag>:latest --tls-verify=false
Getting image source signatures
-- redacted for brevity --
Copying config 0785b63fd1 done
Writing manifest to image destination
Storing signatures
```
If you recieve authentication errors then you simple log back into the local repo using podman:
```
podamn login builder-system.mydomain.com
```

## Use in the Ansible Automation Environment

As it stands there are two ways to use this execution environment in Ansible Automation.

1. You can pull the conatiner image/ Execution Environment form the Automation Hub Through the UI, but currently that method 
   seems bugged.

2. Pull the container image / EE form the Automation Hub via CLI on AAP. This method has provent to be the most reliable method.

Log into the AAP (Tower) server through SSH and become the user 'awx'
```
% ssh samiam@ansible-controller.mydomian.com
(samiam@ansible-controller.mydomian.com) Password: greeneggsandham
Last login: Mon Apr 18 12:57:48 2022 from 10.136.192.147
[samiam@ansible-controll~]$ sudo su -
[sudo] password for samiam: greeneggsandham
Last login: Mon Apr 18 12:57:56 EDT 2022 on pts/13
[root@ansible-controll~]# su - awx
Last login: Mon Apr 18 11:26:23 EDT 2022 on pts/9
```
Using podman log into the aap-hub02 container repo:
```
[awx@ansible-controll~]$ podman login builder-system.mydomain.com
Username: **********
Password:
Login Succeeded!
```
Pull the container image/EE that you wish to use: (In this instance the container we pushed to the manifest)
```
[awx@ansible-controll~]$ podman pull builder-system.mydomain.com/repo-name/<custom-image-tag>:v1.0.0
-- redacted for brevity ---
```

Finally check the list of repos on AAP. Once you configure the UI to use this image you should not have to pull 
it over again unless the image is updated.
```
[awx@ansible-controll~]$ podman image list
REPOSITORY                                                            TAG         IMAGE ID      CREATED       SIZE
builder-system.mydomain.com/repo-name/<custom-image-tag>                    v1.0.0      0785b63fd156  3 days ago    1.86 GB
registry.redhat.io/ansible-automation-platform-21/ee-minimal-rhel8    latest      322f68e2af37  6 days ago    394 MB
registry.redhat.io/ansible-automation-platform-21/ee-supported-rhel8  latest      95e4933abcaa  4 months ago  1.13 GB
registry.redhat.io/ansible-automation-platform-21/ee-29-rhel8         latest      c324ca074ac2  4 months ago  785 MB
```
To verify that you packages were installed you can enter the conatiner you wish to inspect using the following command:
```
podman run -ti --name package-check --hostname zabbix_ee_supported --network host builder-system.mydomain.com/repo-name/<custom-image-tag>:v1.0.0 /bin/bash
```
From there you can check the packages. For this case we will be looking for the ansible and zabbix-api Python modules:
```
bash-4.4# pip3 list | egrep 'ansible|zabbix'
ansible                           5.6.0
ansible-core                      2.12.4
ansible-pylibssh                  0.2.0
ansible-runner                    2.1.3
zabbix-api                        0.5.4
```
Once you are done remove the conatiner you created:
```
$podman rm package-check
3ec7b48432bc95b9d937035ff43e9cc06d0cbc22639a0555a26831406e87e30d
```
