#Deploy nginx webservers on Exoscale with Ansible

In our (precedent article)[link here] we discussed how to use the command line tool called cs to automate the deployment of virtual machines on Exoscale. This tool is very useful to provision VMs or execute some network related commands. But how can we go further and also start the needed services on those VMs ?

To do so, devops engineers use provisionning tools. There are many of them, such has puppet or chef, but one is particularly interesting to use with Exoscale: Ansible. Since version 2.0, it natively features a cloudstack module that can be used to provision VMs on Exoscale. It is really easy to set up, as it only requires a functionning ssh connection with a VM to be able to deploy services.

This article is going to showcase a basic example of a couple Ansible playbooks aiming to deploy some web services on Exoscale in a few seconds.

#Requirements

Besides an account on Exoscale the only needed tool are Ansible and cs

    $ pip install Ansible
    $ pip install cs

As per cs documentation, export the following values in your shell:

```
CLOUDSTACK_ENDPOINT="https://api.exoscale.ch/compute"
CLOUDSTACK_KEY="your api key"
CLOUDSTACK_SECRET_KEY="your secret key"
EXOSCALE_ACCOUNT_EMAIL="your@email.net"
```

Is it also possible to put those value in a .cloudstack.ini in your current folder. Refer to the documentation for more informations.

#A quick tour of ansible

Ansible, meant to be run from a management computer, is taking its instructions from text files called playbooks. Those files contains a succession of instructions in YAML format. Those instructions can be run on localhost or remote machines. Ansible being agentless, it does not need anything else than a functionning ssh connection to a machine in order to execute commands on it.

Our goal in this example is to run two playbooks:
- A first one creating the ssh keys, a specific network security group and Ubuntu virtual machines.
- A second one installing our webserver services via apt in those previously created virtuals machines.

#Ansible project organization

Let's analyze a little bit the content of our project:

```
├── ansible.cfg
├── cloudstack.ini
├── create-instances-playbook.yml
├── install-nginx-playbook.yml
├── inventory
└── roles
    ├── common
    │   └── tasks
    │       ├── create_sshkey.yml
    │       └── main.yml
    └── infra
        ├── tasks
        │   ├── create_inv.yml
        │   ├── create_secgroup.yml
        │   ├── create_secgroup_rules.yml
        │   ├── create_vm.yml
        │   └── main.yml
        └── templates
            └── inventory.j2
```
Our two playbooks are sitting at the root of the project.
If we dig a little bit into the create-instances-playbook.yml file, we will see several interesting informations:
- variables that help define our infrastructure (you can modify this if need be)
- a list of roles to apply. Roles are defined in folders and contains list of playbooks (described in the main.yml at the root of each folders). Those playbooks are executed sequentially as listed in the main.yml files at the root of each folders.

#Instances creation on Exoscale

    $ ansible-playbook create-instances-playbook.yml

```
[...]

TASK [common : include] ********************************************************
included: /Users/lalu/dev/exoscale/ansible/nginx/roles/common/tasks/create_sshkey.yml for localhost

TASK [common : Create SSH Key] *************************************************
ok: [localhost -> localhost]

TASK [common : debug] **********************************************************
ok: [localhost] => {
    "msg": "private key is -----BEGIN RSA PRIVATE KEY-----\nMIICXQIBAAKBgQCOkc5K8TbK[...]0FJCJ5i7zsCQQCylMpFz8RBS/GE0GIS7nRh8G2HNpynSzvDLgQ6OAtG\nEKZPXowdGhtIwjRCihBXK88qik1nbOI4b2v2JRSBL/+T\n-----END RSA PRIVATE KEY-----\n"
}

[...]

TASK [infra : include] *********************************************************
included: /Users/lalu/dev/exoscale/ansible/nginx/roles/infra/tasks/create_secgroup.yml for localhost

TASK [infra : Create Security Group] *******************************************
changed: [localhost -> localhost]

TASK [infra : include] *********************************************************
included: /Users/lalu/dev/exoscale/ansible/nginx/roles/infra/tasks/create_secgroup_rules.yml for localhost

[...]

TASK [infra : include] *********************************************************
included: /Users/lalu/dev/exoscale/ansible/nginx/roles/infra/tasks/create_vm.yml for localhost

TASK [infra : Create instances] ************************************************
changed: [localhost -> localhost] => (item=1)
changed: [localhost -> localhost] => (item=2)

TASK [infra : include] *********************************************************
included: /Users/lalu/dev/exoscale/ansible/nginx/roles/infra/tasks/create_inv.yml for localhost

TASK [infra : Create inventory file] *******************************************
changed: [localhost]

PLAY RECAP *********************************************************************
localhost                  : ok=12   changed=6    unreachable=0    failed=0
```
As we can see in the play recap 6 changes have been undergone. In our Exoscale console we can now see:
- a new ssh public key (the private key being in your .ssh folder)
- a new security group and rules
- 2 new instances

You should be able to connect to your newly created instance with:

    $ ssh -i ~/.ssh/id_rsa_nginx root@<instance-ip-address>

Important point to know, all Ansible playbooks are idempotent, which mean that you can run them as many times as you want, if the required state is already met, no change is going to be made.
So if we run a second time our playbook, we'll a get a recap looking like this and nothing will have changed:

```
PLAY RECAP *********************************************************************
localhost                  : ok=12   changed=0    unreachable=0    failed=0
```


#Start a nginx webserver on the created Instances

#Go further
