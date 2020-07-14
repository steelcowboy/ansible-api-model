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

# Benchmark
For my personal repo that I refactored to fit this model (which already had 
quite a bit in parallel) I found a *20% decrease* in execution time.

## Before
```
~/ansible|5a80a5d ⇒  for x in {1..3}; do time sudo ansible-playbook -i inventory clients.yml > /dev/null 2>&1; done   
sudo ansible-playbook -i inventory clients.yml > /dev/null 2>&1  9.71s user 1.86s system 90% cpu 12.782 total
sudo ansible-playbook -i inventory clients.yml > /dev/null 2>&1  9.80s user 2.01s system 92% cpu 12.768 total
sudo ansible-playbook -i inventory clients.yml > /dev/null 2>&1  9.92s user 1.86s system 91% cpu 12.811 total
```
Average: 12.787s

## After
```
~/ansible|master ⇒  for x in {1..3}; do time sudo ansible-playbook -i inventory clients.yml > /dev/null 2>&1; done
sudo ansible-playbook -i inventory clients.yml > /dev/null 2>&1  7.54s user 1.76s system 88% cpu 10.531 total
sudo ansible-playbook -i inventory clients.yml > /dev/null 2>&1  7.60s user 1.68s system 84% cpu 10.979 total
sudo ansible-playbook -i inventory clients.yml > /dev/null 2>&1  7.18s user 1.71s system 89% cpu 9.952 total
```
Average: 10.487s 

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

PLAYBOOK: clients.yml ******************************************************************************************************************************************************************************************
1 plays in clients.yml

PLAY [clients] *************************************************************************************************************************************************************************************************

TASK [Gathering Facts] *****************************************************************************************************************************************************************************************
ok: [localhost]

TASK [zsh : queue zsh for installation] ************************************************************************************************************************************************************************
ok: [localhost] => {
    "ansible_facts": {
        "packages": {
            "install": [
                "zsh"
            ],
            "remove": []
        }
    },
    "changed": false
}

TASK [git : queue git for installation] ************************************************************************************************************************************************************************
ok: [localhost] => {
    "ansible_facts": {
        "packages": {
            "install": [
                "zsh",
                "git"
            ],
            "remove": []
        }
    },
    "changed": false
}

TASK [fasd : queue autojump for removal] ***********************************************************************************************************************************************************************
ok: [localhost] => {
    "ansible_facts": {
        "packages": {
            "install": [
                "zsh",
                "git"
            ],
            "remove": [
                "autojump"
            ]
        }
    },
    "changed": false
}

TASK [fasd : queue fasd for installation] **********************************************************************************************************************************************************************
ok: [localhost] => {
    "ansible_facts": {
        "packages": {
            "install": [
                "zsh",
                "git",
                "fasd"
            ],
            "remove": [
                "autojump"
            ]
        }
    },
    "changed": false
}

TASK [google_chrome : import Google's apt key] *****************************************************************************************************************************************************************
ok: [localhost]

TASK [google_chrome : set up Chrome repository] ****************************************************************************************************************************************************************
ok: [localhost]

TASK [google_chrome : install Google Chrome] *******************************************************************************************************************************************************************
ok: [localhost]

TASK [google_chrome : set Chrome as the default browser] *******************************************************************************************************************************************************
ok: [localhost]

TASK [repos : import apt keys] *********************************************************************************************************************************************************************************
ok: [localhost] => (item=https://dl.google.com/linux/linux_signing_key.pub)

TASK [repos : set up repositories] *****************************************************************************************************************************************************************************
ok: [localhost] => (item=deb http://dl.google.com/linux/chrome/deb/ stable main)

TASK [packages : debug] ****************************************************************************************************************************************************************************************
ok: [localhost] => {
    "packages": {
        "install": [
            "zsh",
            "git",
            "fasd"
        ],
        "remove": [
            "autojump"
        ]
    }
}

TASK [packages : installing the following packages] ************************************************************************************************************************************************************
ok: [localhost] => {
    "packages.install": [
        "zsh",
        "git",
        "fasd"
    ]
}

TASK [packages : install packages] *****************************************************************************************************************************************************************************
ok: [localhost] => {
    "cache_update_time": 1594529562,
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
                "zsh",
                "git",
                "fasd"
            ],
            "only_upgrade": false,
            "package": [
                "zsh",
                "git",
                "fasd"
            ],
            "policy_rc_d": null,
            "purge": false,
            "state": "latest",
            "update_cache": null,
            "upgrade": null
        }
    }
}

TASK [packages : removing the following packages] **************************************************************************************************************************************************************
ok: [localhost] => {
    "packages.remove": [
        "autojump"
    ]
}

TASK [packages : remove packages] ******************************************************************************************************************************************************************************
ok: [localhost] => {
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
                "autojump"
            ],
            "only_upgrade": false,
            "package": [
                "autojump"
            ],
            "policy_rc_d": null,
            "purge": false,
            "state": "absent",
            "update_cache": null,
            "upgrade": null
        }
    }
}

PLAY RECAP *****************************************************************************************************************************************************************************************************
localhost                  : ok=10   changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```
