---
title: NixOS and OpenStack Compute
permalink: /NixOS_and_OpenStack_Compute/
---

NixOS has some work-in-progress support for running [OpenStack Compute](http://openstack.org/projects/compute/) (also known as **Nova**), an infrastructure-as-a-service cloud computing platform. It's currently limited to running all of Nova on a single machine, so it's mostly useful for experimentation.

Enabling Nova
-------------

Add the following to your `configuration.nix`:

    virtualisation.nova.enableSingleNode = true;

and run `nixos-rebuild switch`. This will enable several daemons:

-   `nova-api` is the API server that receives and executes requests from cloud users, e.g. to start, query or terminate virtual machines.
-   `nova-objectstore` is a simple disk image server. It serves images placed in `/var/lib/nova/images/`.
-   `nova-scheduler` schedules VM execution requests and hands them over to a `nova-compute` node.
-   `nova-network` manages networks and allocates IP addresses.
-   `nova-compute` starts and manages virtual machines. It downloads disk images from `nova-objectstore`.
-   `libvirtd` is the low-level daemon responsible for running and managing virtual machines.
-   `rabbitmq` is a messaging protocol used by the various Nova components to communicate with each other.

Configuring and testing Nova
----------------------------

You can now create a user with administrative privileges:

    $ nova-manage user admin eelco
    2011-04-07 12:57:23,806 AUDIT nova.auth.manager [-] Created user eelco (admin: True)

You should also create a range of IP addresses that can be assigned to VMs:

    $ nova-manage network create 10.0.0.0/8 1 32

Nova will set up `iptables` rules to perform NAT on these addresses automatically.

You should also create a <em>project</em>. In Nova, VMs are always associated with a project, which can have multiple users. To create a project `test` with `eelco` as owner:

    $ nova-manage project create test eelco
    2011-04-07 13:03:01,404 AUDIT nova.auth.manager [-] Created project test with manager eelco

Tools such as `euca2ools`, which allow you to perform operations on the cloud, require a number of environment variables containing stuff such as credentials and the URL of the API server. To obtain the necessary environment variables for a specific project and user:

    $ nova-manage project environment test eelco

This create a file `./novarc`, which you can source from your shell (e.g. `source ./novarc`). The important variables are:

    export EC2_ACCESS_KEY="72389c9d-...:test"
    export EC2_SECRET_KEY="6c145c32-..."
    export EC2_URL="http://192.168.1.20:8773/services/Cloud"

Next we'll get a small VM images so that we can run a VM.

    $ cd /var/lib/nova/images
    $ curl http://images.ansolabs.com/tty.tgz | tar xvfzo -

Check that Nova can see the images:

    $ euca-describe-images
    IMAGE   ami-tty demo/tty        admin   available       public          x86_64  machine aki-tty ari-tty
    IMAGE   aki-tty nova/tty-kernel admin   available       public          x86_64  kernel
    IMAGE   ari-tty nova/tty-ramdisk        admin   available       public          x86_64  ramdisk

To log in to VMs via SSH, you'll need to generate an SSH key pair. The VM will receive the public key on startup, while you get the private key to log in.

    $ euca-add-keypair my_key > my_key
    $ chmod 600 my_key

Now we can start a VM:

    $ euca-run-instances -k my_key ami-tty
    RESERVATION     r-ngnfe3x6      test    default
    INSTANCE        i-00000007      ami-tty                 scheduling      my_key (test, None)     2011-04-07 12:50:19.155448      None    None

After a while, if everything works out, the VM will start. You can get its IP address using `euca-describe-instances`:

    $ euca-describe-instances i-00000007
    RESERVATION     r-ngnfe3x6      test    default
    INSTANCE        i-00000007      ami-tty 10.0.0.3        10.0.0.3        running         my_key (test, stan)     0       m1.small        2011-04-07 12:50:19.155448      nova

So we can now log in using the SSH key generated above:

    $ ssh -i ./my_key 10.0.0.3
    ...
    # uname -a
    Linux ttylinux_host 2.6.35-22-virtual #35-Ubuntu SMP Sat Oct 16 23:19:29 UTC 2010 x86_64 GNU/Linux

To Do
-----

-   Support running Nova components on multiple machines (especially `nova-compute`).
-   Don't use RabbitMQ's guest user.
-   Don't run everything as root.
