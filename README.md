#Deploy nginx webservers on Exoscale with Ansible

In our (precedent article)[https://www.exoscale.ch/syslog/2016/02/23/get-started-with-the-exoscale-api-client/] we discussed how to use the command line tool called cs to automate the deployment of virtual machines on Exoscale. This tool is very useful to provision VMs or execute some network related commands. But how can we go further and also start needed services on those VMs ?

To do so, devops engineers are using provisionning tools. There are many of them available, such has puppet or chef, but one is particularly is pairing particularly well with Exoscale: Ansible. Since version 2.0, it natively features a cloudstack module that can be used to provision VMs on Exoscale. It is really easy to set up, as it only requires a functionning ssh connection with a VM to be able to deploy services.

This article is going to showcase a basic example of a couple Ansible playbooks aiming to deploy some web servers on Exoscale in a few seconds.

##Requirements

First get access to the playbooks by cloning the repository:

    $ git clone git@github.com:MBuffenoir/ansible-exoscale-nginx.git

All the needed tools (Ansible and cs) are listed in the requirements.txt file and you can install them with a simple:

    $ pip install -r requirements

As per cs (documentation)[https://github.com/exoscale/cs], export the following values in your shell:

```
CLOUDSTACK_ENDPOINT="https://api.exoscale.ch/compute"
CLOUDSTACK_KEY="your api key"
CLOUDSTACK_SECRET_KEY="your secret key"
EXOSCALE_ACCOUNT_EMAIL="your@email.net"
```

##A quick tour of Ansible

Ansible, meant to be run from a management computer (like a (bastion)[https://www.exoscale.ch/syslog/2016/01/15/secure-your-cloud-computing-architecture-with-a-bastion/] for example), is taking its instructions from text files called playbooks. Those files contains a succession of instructions in YAML format. Those instructions can be run on localhost or remote machines. Ansible being agentless, it does not need anything else than a functionning ssh connection to a machine in order to execute commands on it.

Our goal in this example is to run two playbooks:
- The first one is creating the ssh keys, a specific network security group and Ubuntu virtual machines.
- The second one is installing our webserver services via apt in those previously created virtual machines.

##Ansible project organization

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
- Variables that help define our infrastructure (you can modify this if need be)
- A list of roles to apply. Roles are defined in folders and contains list of playbooks (described in the main.yml at the root of each folders). Those playbooks are executed sequentially as listed in the main.yml files at the root of each folders. You could view then as "sub-playbook" helping to keep your project organized and reusable.
This playbook will define and install the infrastructure part of our little project.

The second playbook is more straighforward and will be used to start some nginx webservers.

##Instances creation on Exoscale

As just mentionnend, in our create-instances-playbook.yml file we will find a `vars` section that define the number of webservers we want to deploy and the kind of linux distribution to be installed. Of course you can modify those values according to your preferences.

```
  vars:
    ssh_key: nginx
    num_nodes: 2
    security_group_name: nginx
    template: Linux Ubuntu 14.04 LTS 64-bit 10G Disk (2015-04-22-c2595b)
    instance_type: Tiny
```

Now, to run the playbook we use the `ansible-playbook` command:

    $ ansible-playbook create-instances-playbook.yml

You should now see your playbook being run through and the different roles referenced in it being called sequentially.
Here it is what it looks like (some part have been pruned):

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
As we can see in the play recap (the last part) 6 changes have been done. In our Exoscale console we can now see:
- a new ssh public key (the private key being in your .ssh folder)
- a new security group and rules
- 2 new instances

Important point to know, all Ansible playbooks are idempotent, which mean that you can run them as many times as you want, if the required state is already met, no change is going to be made.
So if we run a second time our playbook, we'll a get a recap looking like this and nothing will have changed:

```
PLAY RECAP *********************************************************************
localhost                  : ok=12   changed=0    unreachable=0    failed=0
```

We can use the ping module from Ansible to check that we can connect to our newly created instances:

    $ ansible all -m ping

The ping module does more than just a regular ping, it will uses the ssh key defined in your ansible.cfg file and ensure that Ansible can run command on the remote host. If everything is OK it returns something similar to:

    185.19.x.x | SUCCESS => {
        "changed": false,
        "ping": "pong"
    }
    185.19.x.x | SUCCESS => {
        "changed": false,
        "ping": "pong"
    }


Of course, you should also be able to connect to your newly created instances with:

        $ ssh -i ~/.ssh/id_rsa_nginx root@<instance-ip-address>


##Start a nginx webserver on the created Instances

Now that our infrastructure is set up, let's see how we can use Ansible to start services.
A quick look into our second playbook reveals one simple task:

    - name: Install nginx web server
      apt: name=nginx state=latest update_cache=true

Since we installed Ubuntu on our machines, we chose to use in this playbook the [apt](http://docs.ansible.com/ansible/apt_module.html) module from the Ansible [collection](http://docs.ansible.com/ansible/list_of_all_modules.html).
Of course Ansible supports all major distribution package managers, so you could easily adapt this to something else than apt.

Running `ansible-playbook install-nginx-playbook.yml` gives us the following recap:

```
PLAY RECAP *********************************************************************
185.19.x.x              : ok=2    changed=1    unreachable=0    failed=0
185.19.x.x              : ok=2    changed=1    unreachable=0    failed=0
```

And we can now test that our webservers are live with a simple:

    $ curl http://185.19.x.x

##Go further

Of course the possibilities with Ansible are endless, as it is also possible to write down you own module.
A good exercise for the reader would be to start by writing some playbooks to do some routine commands on the instances, like upgrading the system, ensure the webservers are correctly started, etc ..

An other nice way to experiment with the cloudstack module would be to create a playbook that remove the installed virtual machines and related configurations.
