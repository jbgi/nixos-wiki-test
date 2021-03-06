---
title: NixOps:Deploying From VirtualBox on any OS
permalink: /NixOps:Deploying_From_VirtualBox_on_any_OS/
---

<http://functional-orbitz.blogspot.com/2013/05/setting-up-nixops-on-mac-os-x-with.html>

5. Setup Distributed Builds

When deploying an instance, nixops needs to build the environment somewhere then it will transfer it to the instance. In order to do this, it needs an already existing NixOS instance to build on. If you were running NixOS already, this would be the machine you are deploying from. To accomplish this, you need a a NixOS running in a VM. Eventually nixops will probably accomplish this for you, but for now it needs to be done manually. Luckily, installing NixOS on VirtualBox is pretty straight forward.

Install a NixOS on VirtualBox from the directions here. This doesn't need any special settings, just SSH.

Setup a port forward so you can SSH into the machine. I'll assume this port forward is 3223.

Make a user called 'nix' on the VM. This is the user that we will SSH through for building. The name of the user doesn't matter, but these directions will assume its name is 'nix'.

On OS X, create two pairs of passwordless SSH keys. One pair will be the login for the nix user. The other will be signing keys.

Install the login public key.

On OS X, create /etc/nix/ (mkdir /etc/nix)

Copy the private signing key to /etc/nix/signing-key.sec. Make sure this is owned by the user you'll be running nixops as and is readable only by that user.

Create a public signing key from your private signing key using openssl. This needs to be in whatever format openssl produces which is not the same as what ssh-keygen created. This output should be in /etc/nix/signing-key.pub. The owner and permissions don't matter as long as the user you'll run nixops as can read it.

` openssl rsa -in /etc/nix/signing-key.sec -pubout > /etc/nix/signing-key.pub`

Copy the signing keys to the build server, putting them in the same location. Make sure the nix user owns the private key and is the only one that can read it.

Tell nix to do distributed builds:

` export NIX_BUILD_HOOK=$HOME/.nix-profile/libexec/nix/build-remote.pl`

Tell the distributed builder where to store load content:

` export NIX_CURRENT_LOAD=/tmp/current-load`
` mkdir /tmp/current-load`

Go into a directory you can create files in:

` cat <`<EOF >` remote-systems.conf`
``  nix@nix-build-server x86_64-linux /Users/`whoami`/.ssh/id_rsa 1 1 ``
` EOF`

Tell the remote builder where to find machine information:

` export NIX_REMOTE_SYSTEMS=$PWD/remote-systems.conf`

Add an entry to ~/.ssh/config the fake host 'nix-build-server' turns into your actual VM:

` Host nix-build-server`
` HostName localhost`
` Port 3223`