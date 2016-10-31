# Project configuration

The main purpose of the Droid project configuration, `droid.yml`, is to declare
one or more Targets.  This page details all of the configuration directives one
might use in the configuration of a Droid project.

An example:-

    name: "Example Project"
    description: "This is an example of a Droid project"
    variables:
        something: "useful"
    environment:
        droid_use_private_net: false
    modules:
        - "docker": "git@github.com:droid-php/droid-module-docker.git"
    targets:
        do_my_bidding:
            ...
    hosts:
        host1.example.com:
            ...
    groups:
       group_a:
            ...
    include:
        - "more-configuration.yml"

## `name`

An optional name of the Droid project.

## `description`

An optional description of the Droid project.  This might describe the reason
or purpose of the project.

## `targets`

The `targets` directive is used to declare one or more Targets and is a mapping
of Target name to [Target configuration][conf-target]. For example, we may
declare a Target named `build_everything` which executes a number of commands
on each Host in a group named `all_machines`:-

    targets:
        build_everything:
            hosts: "all_machines"
            tasks:
                - ...

A Target declared here is executed from the command line by giving the name of
the Target as an argument to the Droid command:-

    $ vendor/bin/droid build_everything

## `variables`

Variables are used to provide commands with named values for use during command
execution.  The `variables` directive is a mapping of arbitrary names to
values.  A value may be a string of characters, a number, a list or a mapping.
For example:-

    variables:
        shape: "line"
        length: 2048
        points:
            - {x: 0}
            - {x: 512}
            - {x: 768}
        point_conf:
            colour: "blue"
            size: 1

Variables may be declared in various places; Targets, Hosts and Groups may
declare their own variables which are combined with and override those declared
here at the Project level.  For example, we may declare sensible default values
at the Project level and declare more-specific values in a Target:-

    variables:
        shape: "line"
        colour: "red"
    targets:
        do_a_thing:
            variables:
                colour: "blue"
                thickness: 1

The values available to commands executed by the above example Target
`do_a_thing` would be:-

    shape: "line"
    colour: "blue"
    thickness: 1

## `modules`

The `modules` directive provides Droid with the download URLs of Droid modules
to be used in a Project and provides names to which to refer to those Modules.
The directive is a mapping of name to URL.  For example, we may declare that a
project will use a Module for setting-up an Apache virtual host:-

    modules:
        "apache-vhost": "git@github.com:droid-php/droid-module-apache-vhost.git"

Droid installs Modules by fetching them from the given URLs.  It stores the
Module content in a directory of the same name, under a directory named
`droid-vendor`:-

    $ vendor/bin/droid module:install
    Installing droid modules:
    - apache-vhost from git git@github.com:droid-php/droid-module-apache-vhost.git
    Cloning from git@github.com:droid-php/droid-module-apache-vhost.git into droid-vendor/apache-vhost
    Cloning into 'droid-vendor/apache-vhost'...
    ...
    Done

Currently, only `git@` URLs are supported.  We would later refer the Module by
the name given here, which can be any name so long as it is used consistently
within a Project.  Please see the [Module documentation][modules] to learn how
to use a Module in a Project.

## `hosts`

The `hosts` directive is used to declare one or more Hosts.  It is a mapping of
Host name to [Host configuration][conf-host].  For example, we may declare
Hosts named `www.example.com` and provide Droid with its IP address:-

    hosts:
        "www.example.com":
            droid_ip: "198.51.100.1"

We may then refer to this Host when we wish to execute commands on it:

    targets:
        make_website:
            hosts: "www.example.com"
            tasks:
                - ...

## `groups`

The `groups` directive is used to declare one or more Groups of Hosts.  It is a
mapping of Group name to [Group configuration][conf-group].  For example, we
may declare a group of web servers named `webservers`:-

    groups:
        webservers:
            hosts:
                - www1.example.com
                - www2.example.com

We may then refer to this Group when we wish to execute commands on its Hosts:

    targets:
        make_website:
            hosts: "webservers"
            tasks:
                - ...

## `environment`

The `environment` directive is intended to provide a way alter the behaviour of
Droid for different operating environments.  This is useful when a Droid
Project is to be used on more than one machine and those machines differ in
some way.  For example, one may have access to a private network of the Hosts
it manages, but another may only be able to connect to the public addresses of
the same Hosts.

The current parameters that may be set via the `environment` directive are:-

- `droid_use_private_net`: `true` or `false` (default `false`).  When `true`
  Droid will connect to the `private_ip` of Hosts instead of the `public_ip`.
  This setting has no effect on Hosts for which a `droid_ip` is defined.  It is
  an error if this value is `true` and a Host defines neither `private_ip` nor
  `public_ip`.

## `include`

The `include` directive allows the Droid Project configuration to be composed
of multiple files to ease maintenance of the configuration.  Each of the other
directives described on this page may be placed in separate files and included
in the main `droid.yml`.  The files are joined together such that the resulting
configuration is the union of the content of all of the files.

The `include` directive is a list of paths, each to a file to in included.
Simple wild card matching is possible using the `*` operator, which matches zero
or more characters and `?`, which matches exactly one character.

For example, we may wish to declare:-

- each Target in a separate file in a `target` sub directory
- Hosts and Groups in a file named `inventory.yml`
- Project variables in a file named `variables.yml`

and place the configuration in five separate files:-

    --------------------------
    # droid.yml
    --------------------------
    include:
        - inventory.yml
        - targets/*.yml
        - variables.yml

    --------------------------
    # inventory.yml
    --------------------------
    hosts:
         www.example.com:
             ...
         db.example.com
             ...
    groups:
         web_servers:
             ...
         db_servers:
             ...

    --------------------------
    # targets/make-website.yml
    --------------------------
    targets:
        make-website:
            hosts: web_servers

    --------------------------
    # targets/make-db.yml
    --------------------------
    targets:
        make-db:
            hosts: db_servers

    --------------------------
    # variables:
    --------------------------
    variables:
        dbname: "mydb"

[conf-group]: </configuration-reference/group.html> "Group configuration"
[conf-host]: </configuration-reference/host.html> "Host configuration"
[conf-target]: </configuration-reference/target.html> "Target configuration"
[modules]: </modules.html> "Modules"
