# Description
This script helps you to automate host provisioning via Satellite 6 with any orchestration tool (e.g. VMWare Orchestrator) of your choice. It also creates hostgroups in Red Hat IPA server according to your Satellite 6 hostgroups as well as an appropriate Red Hat IPA automember rule.
Furthermore it creates Satellite 6 host entries with max. 3 NIC`s:

1. Nic: this is the primary NIC whis is connected to the public network
2. Nic: is for inguest NFS storage which is connected to the storage LAN
3. Nic: is for Oracle databases which uses a dedicated network for replication purposes

On **IPA** you need a service user which you will use wihtin this script to interact with IPA command line. This user needs sufficient rights in IPA to create host groups and add automember rules.

#About this script
- Author: Frank Reimer
- Version: 1.1
- Creation Date: 2016-06-23

# Table of content
* [Description](#description)
* [About this script](#about-this-script)
* [Table of content](#table-of-content)
* [Features](#features)
* [Prerequisites:](#prerequisites)
    * [1. Create IPA automation service user on IPA server](#1-create-ipa-automation-service-user-on-ipa-server)
    * [2. Create Satellite automation service user on IPA server](#2-create-satellite-automation-service-user-on-ipa-server)
    * [3. On Satellite](#3-on-satellite)
    * [4. Change hardcoded variables in this script according to your needs](#4-change-hardcoded-variables-in-this-script-according-to-your-needs)

# Features
This script 
- creates a Satellite host entry with max. three nic`s
- creates Satellite hostgroups as follows:

If you pass the option "--application" to the script it creates child hostgroups as follows:
```
hg-application ==> hg-<application> ==> hg-<application>-<lifecycle-environment>
```											
If you pass the option "--infrastructure" to the script it creates child hostgroups as follows:
```
hg-infrastructure ==> hg-<application> ==> hg-<application>-<lifecycle-environment>
```
- creates Satellite subnets according to your host`s network information you pass to this script (if not already present)
- downloads host iso images for provisioning to a mounted NFS volume on Satellite
- creates Red Hat IPA hostgroups according to your Satellite hosthgroups as follows:
```
hg-<application>-<lifecycle-environment>
```
- creates Red Hat IPA hostgroup automember rule and assings the Satellite host automatically to the Red Hat IPA hostgroup as follows:
```
hg-<application>-<lifecycle-environment>
```
- creates custom host partitioning table and uploads it to Satellite. Furthermore it assigns the parttition table to your host and to the default operating system defined in this script.

#Prerequisites:
### 1. Create IPA automation service user on IPA server
- Login as an IPA admin user:
```
kinit admin
```
- Create a service user and a group. Afterwards assign the group to the user:
```
ipa user-add --first="IPA Automation" --last="Service user" --displayname="svc-ipa-automation" --random svc-ipa-automation
ipa group-add kn-satellite-automation --desc "This group is used for all Satellite automation purposes."
ipa group-add-member kn-satellite-automation --users svc-satellite-automation
```
- Now create a password policy for the Satellite automation user group where your service user is a member of which ensures that the user password never expires:
```
ipa pwpolicy-add kn-satellite-automation --maxlife=20000 --minlength=8 --priority 10
```
- Get Kerberos keytab for user "svc-ipa-automation":
```
ipa-getkeytab -s <ipa-server-name> -p svc-ipa-automation -k svc-ipa-automation.keytab
```
- Configure permission to create automember rules:
```
ipa permission-add "System Add Automember Rule" --bindtype=permission --right=read --right=search --right=compare --right=write --right=add --attr=automemberexclusiveregex --attr=automemberinclusiveregex --attr=automembertargetgroup --attr=cn --attr=createtimestamp --attr=description --attr=entryusn --attr=modifytimestamp --attr=objectclass --type=automember

``` 
- Create privilege:
```
ipa privilege-add "Automember Create Rule"
```
- Add permission to privilege:
```
ipa privilege-add-permission "Automember Create Rule" --permission="System Add Automember Rule"
```
- Create Role:
```
ipa role-add "IPA Automation"
```
- Add privileges to role:
```
ipa role-add-privilege "IPA Automation" --privileges="Automember Readers" --privileges="Host Group Administrators"  --privileges="Automember Create Rule"
```
- Assign role to user:
```
ipa role-add-member "IPA Automation" --users="svc-ipa-automation"
```

### 2. Create Satellite automation service user on IPA server
- Create the service user:
```
ipa user-add --first="Satellite Automation" --last="Service user" --displayname="svc-satellite-automation" --password svc-satellite-automation
```
- Create HBAC rule:
```
ipa hbacrule-add allow_svc-satellite-automation_on_satellite --servicecat=all
```
- Assign user to HBAC rule:
```
ipa hbacrule-add-user allow_svc-satellite-automation_on_satellite --users=svc-satellite-automation
```
- Assign Satellite host to HBAC rule:
```
ipa hbacrule-add-host allow_svc-satellite-automation_on_satellite --hosts=<your-satellite-server>
```

### 3. On Satellite
- Install IPA client on Satellite 6 server and configure it accordingly to authenticate your Satellite 6 server against IPA server
- Install IPA admin tools:
```
yum install ipa-admintools
```
- As root switch to your Satellite automation service user and create a passwordless SSH key pair:
```
su - svc-satellite-automation
ssh-keygen -t rsa -b 4096
``` 
- As your Satellite automation service user upload the SSH public key to IPA. Please change your password if this is your first login attempt after you created the user:
```
[svc-satellite-automation@satellite ~]$ kinit 
Password for svc-satellite-automation@<YOUR-IPA-REALM>: 
Password expired.  You must change it now.
Enter new password: 
Enter it again: 

ipa user-mod svc-satellite-automation --sshpubkey="<insert the content of your Satellite automation service user SSH pub key you`ve created before>"
```
- Login as **root** to Satellite via SSH and create the file "~/.hammer/cli_config.yml" with the following content:
```
:foreman:
        :host: 'https://<your-satellite-server>'
        :username: '<satellite-admin-user>'
        :password: '<satellite-admin-password>'
```
**Please replace the variable names according to your setup.**
- Create a local Satellite user with the same name as the IPA managed Satellite service user:
```
hammer user create --firstname "Satellite Automation" --lastname "Service user" --login svc-satellite-automation --auth-source-id 1 --mail <user-email> --password <secret-password> --organizations <your-organizations> --locations <your-locations>
```
- Now create and assign appropriate role for your Satellite automation service user via hammer:
```
hammer role create --name "Satellite automation Subnets"
hammer role create --name "Satellite automation Hostgroups"
hammer role create --name "Satellite automation Hosts"
hammer role create --name "Satellite automation Locations"
hammer role create --name "Satellite automation Organizations"
hammer role create --name "Satellite automation Medium"
hammer role create --name "Satellite automation Architecture"
hammer role create --name "Satellite automation Bootdisk"
hammer role create --name "Satellite automation Lifecycle Environment"
hammer role create --name "Satellite automation Environment"
hammer role create --name "Satellite automation ActivationKey"
hammer role create --name "Satellite automation Realm"
hammer role create --name "Satellite automation Operatingsystem"
hammer role create --name "Satellite automation ForemanTask"
hammer role create --name "Satellite automation Contentview"
hammer role create --name "Satellite automation Domain"
hammer role create --name "Satellite automation SmartProxy"
hammer role create --name "Satellite automation ComputeResource"
hammer role create --name "Satellite automation Partition tables"

hammer filter create --permissions view_subnets,create_subnets,edit_subnets --organizations <your-organizations> --locations <your-locations> --role "Satellite automation Subnets"
hammer filter create --permissions view_hostgroups,create_hostgroups,edit_hostgroups --organizations <your-organizations> --locations <your-locations> --role "Satellite automation Hostgroups"
hammer filter create --permissions build_hosts,destroy_hosts,edit_hosts,create_hosts,view_hosts --role "Satellite automation Hosts"
hammer filter create --permissions view_locations,assign_locations --role "Satellite automation Locations"
hammer filter create --permissions view_organizations,assign_organizations --role "Satellite automation Organizations"
hammer filter create --permissions view_media --organizations <your-organizations> --locations <your-locations> --role "Satellite automation Medium"
hammer filter create --permissions view_architectures --role "Satellite automation Architecture"
hammer filter create --permissions download_bootdisk --role "Satellite automation Bootdisk"
hammer filter create --permissions view_lifecycle_environments --role "Satellite automation Lifecycle Environment"
hammer filter create --permissions view_environments --organizations <your-organizations> --locations <your-locations> --role "Satellite automation Environment"
hammer filter create --permissions view_activation_keys --role "Satellite automation ActivationKey"
hammer filter create --permissions view_realms --organizations <your-organizations> --locations <your-locations> --role "Satellite automation Realm"
hammer filter create --permissions view_operatingsystems --role "Satellite automation Operatingsystem"
hammer filter create --permissions view_foreman_tasks --role "Satellite automation ForemanTask"
hammer filter create --permissions view_content_views,publish_content_views,promote_or_remove_content_views --role "Satellite automation Contentview"
hammer filter create --permissions view_domains --role "Satellite automation Domain"
hammer filter create --permissions view_smart_proxies,view_smart_proxies_autosign --role "Satellite automation SmartProxy"
hammer filter create --permissions view_compute_resources,view_compute_resources_vms --role "Satellite automation Domain"
hammer filter create --permissions view_ptables,create_ptables,edit_ptables,destroy_ptables --role "Satellite automation Partition tables"

hammer user add-role --login svc-satellite-automation --role "Satellite automation Subnets"
hammer user add-role --login svc-satellite-automation --role "Satellite automation Hostgroups"
hammer user add-role --login svc-satellite-automation --role "Satellite automation Hosts"
hammer user add-role --login svc-satellite-automation --role "Satellite automation Locations"
hammer user add-role --login svc-satellite-automation --role "Satellite automation Organizations"
hammer user add-role --login svc-satellite-automation --role "Satellite automation Medium"
hammer user add-role --login svc-satellite-automation --role "Satellite automation Architecture"
hammer user add-role --login svc-satellite-automation --role "Satellite automation Bootdisk"
hammer user add-role --login svc-satellite-automation --role "Satellite automation Lifecycle Environment"
hammer user add-role --login svc-satellite-automation --role "Satellite automation Environment"
hammer user add-role --login svc-satellite-automation --role "Satellite automation ActivationKey"
hammer user add-role --login svc-satellite-automation --role "Satellite automation Realm"
hammer user add-role --login svc-satellite-automation --role "Satellite automation Operatingsystem"
hammer user add-role --login svc-satellite-automation --role "Satellite automation ForemanTask"
hammer user add-role --login svc-satellite-automation --role "Satellite automation Contentview"
hammer user add-role --login svc-satellite-automation --role "Satellite automation Domain"
hammer user add-role --login svc-satellite-automation --role "Satellite automation SmartProxy"
hammer user add-role --login svc-satellite-automation --role "Satellite automation Domain"
hammer user add-role --login svc-satellite-automation --role "Satellite automation Partition tables"
```
**Please replace the variable names according to your setup.**
- Create parent hostgroups for applications as well as infrastructure services:
```
hammer hostgroup create --name hg-application --organizations <your-organization>
hammer hostgroup create --name hg-infrastructure --organizations <your-organization>
```
**Please replace the variable names according to your setup.**
- Login as a Satellite admin user to the Satellite web-ui and assign your Puppet classes or Puppet configuration groups (according to your configuration standard or SOE - Standard Operating Environment) to both hostgroups. Furthermore you should set your default root password for host provisioning in Satellite web-ui here:
```
Administer -> Settings -> Provisioning -> root_pass
```
- Furthermore you need to create a default Operating System entry as well as a default Activation Key in Satellite which you use in this script for host provisioning.
- Login as **svc-satellite-automation** to Satellite via SSH and create the file "~/.hammer/cli_config.yml" with the following content:
```
:foreman:
        :host: 'https://<your-satellite-server>'
        :username: '<satellite-automation-service-user>'
        :password: '<satellite-automation-service-user-password>'
```
**Please replace the variable names according to your setup.**
- Copy the Kerberos keytab "svc-ipa-automation.keytab" you created in chapter 1. to svc-satellite-automation home directory.

### 4. Change hardcoded variables in this script according to your needs
- Open the script and search for **"# Change this variable"**.
- Change all variables according to your needs or create an option for this variable to pass by this script as an argument.
