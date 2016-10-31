# Enable remote command execution

## Introduction

Droid reads Tasks from the Droid Project to execute Droid Commands either on
the local host or on remote Hosts.  This page describes how Droid achieves
remote execution and what steps are required to facilitate it.

### Requirements of remote Hosts

To execute Droid Commands on a remote Host, Droid will install itself on the
remote Host.  To achieve this Droid will copy the `composer.json` and
`composer.lock` files from the Droid Project on the local host to the remote
Host. Droid will then install Composer on the remote Host and, finally, will
invoke `composer install` there.

Thus, the remote Host must satisfy some conditions:-

- An SSH service must be running on the remote Host and accepting connections
  from the local host.
- A user account must exist on the remote Host, configured for SSH Public Key
  authentication.
- A PHP Command Line Interpreter (CLI) must be installed on the remote Host, at
  or above version 5.5.9.
- The remote Host must be able to access the Worldwide Web (both HTTP and
  HTTPS).

Droid will install Composer and itself to `/usr/local/droid` on the remote Host
if the user account has sufficient privileges; otherwise it will install to
`/tmp`.  The former location is preferred over the latter because the
installation will be lost when `/tmp` is cleared, as often happens when a Host
is rebooted.

Ideally then, the user account on the remote Host will have permissions to read
and write `/usr/local/droid`.  A good way to achieve this is to create a
dedicated user account on the remote Host, making `/usr/local/droid` its Home
Directory:-

    $ sudo adduser --disabled-password --gecos "Droid" --home "/usr/local/droid" "droid"

The user account may need to be able to execute commands with `sudo` and it
must be able to do so without being required to provide a password.  One way to
achieve this is to issue the following command on the remote Host:-

     $ echo "droid ALL=(root) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/droid

This will allow the `droid` user account to issue `sudo` with all commands
without being required to provide a password.  It is recommended, however, that
the user account be limited in the commands it can execute in this way.  An
example of a more controlled `/etc/sudoers.d/droid` file might be:-

    # Sudoers for Droid
    Cmnd_Alias DROID_COMMANDS = /usr/bin/apt-get install,/bin/chown,/bin/chmod
    droid ALL=(root) NOPASSWD:DROID_COMMANDS

See the sudoers manual page (`man 5 sudoers`) for more information about the
format of a sudoers file.

### Requirements of the local host

In concert with the requirements of remote Hosts, Droid has some requirements
of the local host:-

- A private SSH key must be available for authenticating Droid to the remote
  Host.
- The private SSH key should be registered with a local SSH Agent to avoid
  repeated prompts for the key's pass phrase during remote Command execution.
- The SSH Host key of the remote Host must be verified and recorded in the
  local database of known hosts.

## Enable remote execution

In this section we will see exactly what steps to take to enable Droid to
execute Commands on a remote Host.  We will not cover the installation of SSH
server on the remote Host, nor how to obtain the fingerprint of the SSH Host
key which identifies the Host.  We will assume that you have already done this
and have verified the fingerprint.

### Remote execution checklist

- On the local host:-
  1. generate an SSH key pair (public and private keys)
  2. register the private SSH key with the SSH Agent
  3. copy the public SSH key to the remote Host
- On the remote host:-
  1. create a dedicated user account for Droid
  2. enable SSH public key authentication
  3. enable password-less Sudoing for the Droid user
  4. install PHP CLI

### On the local host

We generate an SSH key pair for use by Droid:-

    $ ssh-keygen -b 2048 -t rsa -C "Identifies Droid" -f ~/.ssh/droid_id_rsa

We are asked for a pass phrase with which to secure the private key.  We are
prompted for this pass phrase each time we use it to authenticate to a remote
Host.  So we register it with our SSH Agent which will "remember" the pass
phrase so that we aren't repeatedly prompted for it:-

    $ ssh-add ~/.ssh/droid_id_rsa

We now copy the public key to a temporary place on the remote Host.  Let's
pretend our remote Host is at 198.51.100.1 and that we have a sufficiently
privileged account there:-

    $ scp ~/.ssh/droid_id_rsa.pub myuser@198.51.100.1:/tmp/

We are now prepared to perform the required steps on the remote Host.

### On the remote Host

We create a dedicated user account for Droid:-

    $ sudo adduser --disabled-password --gecos "Droid" \
           --home "/usr/local/droid" "droid"

We enable public key authentication for our Droid user, using the public key
uploaded in a previous step:-

    $ sudo mkdir /usr/local/droid/.ssh
    $ sudo chown droid:droid /usr/local/droid/.ssh
    $ sudo chmod 0750 /usr/local/droid/.ssh
    $ sudo mv /tmp/droid_id_rsa.pub /usr/local/droid/.ssh/authorized_keys
    $ sudo chown droid:droid /usr/local/droid/.ssh/authorized_keys
    $ sudo chmod 0640 /usr/local/droid/.ssh/authorized_keys

We enable password-less Sudoing for our Droid user, writing the following
content to a sudoers file:

    # Sudoers for Droid
    Cmnd_Alias DROID_COMMANDS = /usr/bin/apt-get install,/bin/chown,/bin/chmod
    droid ALL=(root) NOPASSWD:DROID_COMMANDS

So we create and edit the sudoers file:-

    $ sudo touch /etc/sudoers.d/droid
    $ sudo vi /etc/sudoers.d/droid

Finally, we install the PHP CLI from the system packages:-

    $ sudo apt-get install php5-cli

We're now able to add this Host to our inventory of Droid-managed Hosts.  Let's
test it!

### Test remote execution

We will create a new Droid Project and have Droid run a simple command on our
remote Host.

    $ mkdir myproject && cd myproject
    $ composer init -n
    $ composer require droid/droid-standard

We will create the following Project configuration:-

    project:
        name: "This is just a test project"
    targets:
        testing:
            tasks:
                - name: "Talk to me"
                  command: "debug:echo"
                  arguments:
                      message: "I am speaking to you from afar"
                  hosts: "testhost"
    hosts:
        testhost:
            droid_ip: "198.51.100.1"
            username: droid
            keyfile: "~/.ssh/droid_id_rsa"

We may now execute the Target named `testing` to run our test:-

    $ vendor/bin/droid testing

If all went well, we should see:-

    Droid: Running target `testing`
    Task `Talk to me`: debug:echo on testhost
    Host testhost: message=I am speaking to you from afar
    testhost Begin droid enablement.

At this point, Droid is installing Composer and itself on `testhost`. After a
pause, we should see the remaining output:-

    testhost Finished droid enablement. Success.
    testhost I am speaking to you from afar
    Result: 0
    --------------------------------------------
    $

This was a successful test.
