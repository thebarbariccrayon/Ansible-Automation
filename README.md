# System Management 
This repo maintains the playbook needed to rotate the root password and exchange keys if the ssh keys are found on the server.

## Rotate Password

### Playbook: 
```
rotate_password.yml
```

### Requiremnets: 
In order to utilize this playbook the Community Crypto collection must be present in the execution environment at the time of execution.

At execution the playbook will prompt the user to enter a password which will then be used for the root password on the hosts that this playbook is ran against. If needed the ssh-keys may also be rotated in this process as well.
### Variables:
For servers to exchange ssh keys the following variable would have to be supplied in the specific host's variable field either in the AAP UI or in an inventory file.
```
---
clients:
```
### Ad-Hoc Execution
```
 ansible-playbook rotate_password.yml \
--limit Ansible \  --> Only if you wish to limit the execution to specific group
-u <user with sudo-rights> \
--become-method sudo \
--become-user root \
--tags root_pass ad-hoc \  --> Th ad-hoc tag check for the presence of a secrets.yml vaulted file if a password extra var is not passed
--ask-vault-pass \ --> prompts for the vault password to decrypt the file
--ask-pass \  --> asks for users credentials for SSH auth
--ask-become-pass --> Prompts for sudo credentials, if none is supplied it defaults to the --ask-pass value. This would be used if a specific user needs to be used.
```