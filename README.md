#Deploy nginx webservers on Exoscale with Ansible

In our precedent article we discussed how to use the command line tool called cs to automate the deployment of virtual machines on Exoscale. But how can we go further and not only spawn new vms but also start the needed services on them ?

To do so, devops engineers use provisionning tools. There are many of them, such has puppet or chef, but one is particularly interesting to use with Exoscale: Ansible. Since version 2.0, it natively features a cloudstack module that can be used to create machines on Exoscale. In addition it is really easy to set up, as it only requires a functionning ssh connection with a vm to be able to deploy services.

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

#Instances creation on Exoscale

#Start a nginx webserver on the created Instances

#Go further
