# Ansible API Model
An example showing a model for Ansible Roles to consolidate work.

## Background
When I set up my new desktop, I wanted to create a system for me to "check in"
configuration, meaning that I could easily replicate my setup if I were to ever
switch to a new computer. While most people choose to do this with bash or
similar, I had two goals:
1. Don't reinvent the wheel
2. Use a system that can work for my PCs and my server(s)

As it was one of the first configuration management systems I learned, I jumped
to Ansible. My initial plan was to have roles for each "type" of operation: one
for packages, one for config files, one for symlinks, etc. That way, I could
simply do all similar operations in a single task and it would be very
efficient.

However, Facebook's Chef cookbooks (== Ansible roles) utilize an idea called [API
Cookbooks](https://github.com/facebook/chef-cookbooks#apis): that is, you
represent a service setting as a hash and then various other cookbooks can
change configure that has. For example, if in my `cpe_chrome` cookbook I
configure Chrome to be version 83, maybe I want to create a cookbook called
`cpe_beta_testers` that sets
```ruby
node.default['cpe_chrome']['version'] = 84
```
And viola! This is evaluated at *compile time* (i.e. when node attributes, or
host variables) are being determined. As long as `cpe_beta_testers` is included
after `cpe_chrome`, its configuration will win and during runtime Chrome version
84 will be installed.

## How can this work for Ansible?
From some super basic research, it seems that Ansible doesn't support this kind
of model very readily. The main reason is that Ansible doesn't have the same
idea of *compile time* vs *run time*, which means we have to fake it. In order
to do this, we set empty variables for the kinds of actions we want to take in
the inventory and build them up in "configuration" roles, and finally have a
role to read those variables and execute them in one go.

Here, I present an example for packages that demonstrates this:
1. Create an empty packages array as some kind of variable (in this case, group
   var)
2. Create roles that configure the package variable
3. Create an action role that takes installs all packages in the package list

## What does this give us?
The biggest benefit here is executing similar actions in paralell. If there is a
logical order to management capabilities (e.g. first create files, then
directories, then clone git repos, then install packages, etc) you can store
similar configuration with each other (e.g. all the settings for ssh, all the
setting for a developer environment) while still getting the benefit of having
various actions execute in parallel.

## Test Run
Here is what this looks like in action!
```
⇒  sudo ansible-playbook -vvv -i inventory clients.yml
ansible-playbook 2.9.6
  config file = /etc/ansible/ansible.cfg
  configured module search path = ['/root/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python3/dist-packages/ansible
  executable location = /usr/bin/ansible-playbook
  python version = 3.8.2 (default, Apr 27 2020, 15:53:34) [GCC 9.3.0]
Using /etc/ansible/ansible.cfg as config file

PLAYBOOK: clients.yml *****************************************************************************************************************************************************************************************
1 plays in clients.yml

PLAY [clients] ************************************************************************************************************************************************************************************************

TASK [Gathering Facts] ****************************************************************************************************************************************************************************************
ok: [localhost]

TASK [pkg1 : queue git for installation] **********************************************************************************************************************************************************************
ok: [localhost] => {
    "ansible_facts": {
        "pkgs_install": [
            "git"
        ]
    },
    "changed": false
}

TASK [pkg2 : queue zsh for installation] **********************************************************************************************************************************************************************
ok: [localhost] => {
    "ansible_facts": {
        "pkgs_install": [
            "git",
            "zsh"
        ]
    },
    "changed": false
}

TASK [init : pkgs to install] *********************************************************************************************************************************************************************************
ok: [localhost] => {
    "pkgs_install": [
        "git",
        "zsh"
    ]
}

TASK [init : install packages] ********************************************************************************************************************************************************************************
ok: [localhost] => {
    "cache_update_time": 1594303134,
    "cache_updated": false,
    "changed": false,
    "invocation": {
        "module_args": {
            "allow_unauthenticated": false,
            "autoclean": false,
            "autoremove": false,
            "cache_valid_time": 0,
            "deb": null,
            "default_release": null,
            "dpkg_options": "force-confdef,force-confold",
            "force": false,
            "force_apt_get": false,
            "install_recommends": null,
            "name": [
                "git",
                "zsh"
            ],
            "only_upgrade": false,
            "package": [
                "git",
                "zsh"
            ],
            "policy_rc_d": null,
            "purge": false,
            "state": "latest",
            "update_cache": null,
            "upgrade": null
        }
    }
}

TASK [init : pkgs to remove] ***********************************************************************************************************************************************************************************
ok: [localhost] => {
    "pkgs_remove": [
        "autojump"
    ]
}

TASK [init : remove packages] **********************************************************************************************************************************************************************************
ok: [localhost]

PLAY RECAP *****************************************************************************************************************************************************************************************************
localhost                  : ok=8    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```
