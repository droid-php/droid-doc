# Target configuration

A Target is like a recipe: it defines a number of steps that Droid will take to
achieve a desired outcome.

The steps of a Target may be a list of Tasks or a list Modules (which are
themselves reusable lists of Tasks) or a mixture of both.  The Tasks of Modules
are always executed before the Tasks of the Target.

A Target may be executed on the local machine or on one or more remote Hosts.
The decision to execute locally or remotely and the selection of remote Hosts
may be declared as part of the Target or as part of individual Tasks.

## `tasks`

This `tasks` property of a Target is a list of Tasks which will be executed in
the order of their definition.

    target:
        make_website:
            tasks:
                - name: "Install Apache"
                  ...
                - name: "Configure Apache"
                  ...

Please see the [Task configuration][conf-task] for the configuration of a Task.

## `hosts`

The `hosts` property of a Target is an expression which identifies the Hosts
and Groups of Hosts on which the Target is to be executed.  The expression may
be a single name or a space or comma separated list of names.  The names must
match those declared in the Project, under the `hosts` and `groups` directives.

    target:
        setup_firewalls:
            hosts: "webservers db.example.com"
            tasks:
                - ...
                - ...

A Task may declare that it is to execute on some a subset of the Hosts
identified here, or even on a completely different set of Hosts (see
`host_filter` and `hosts` of the [Task configuration][conf-task]).

## `modules`

The `modules` property of a Target is a list of Module names.  The Tasks of
each of the specified Modules will be executed before those in `tasks` list.
Modules are executed in the order in which they are declared here.

    target:
        make_webservers:
            modules:
                - "install-apache"
                - "apache-vhost"

The names of Modules must match those declared in the main `modules` directive
(see [Project configuration][conf-project]).

## `triggers`

The `triggers` property of a Target is a list of Tasks whose execution depends
upon the outcome of the execution of other Tasks.  A Task can trigger the
execution of another Task when its Command is able to report whether or not it
made a change.  For example, a Task which modifies the configuration of a
system service may cause the service to reload its configuration:-

    target:
        make_website:
            triggers:
                - name: "Reload Apache"
                  command: "service:reload"
                  arguments: {name: "apache2"}
            tasks:
                - name: "Enable Apache module"
                  command: "apache:enmod"
                  arguments: {module-name: ...}
                  trigger: "Reload Apache"

A Task is triggered only when such a Command reports that a change was made.

Apart from its conditional execution, a `triggers` Task is no different from a
`tasks` Task.  Please see the [Task configuration][conf-task] for the
configuration of a Task.

## `variables`

The `variables` property of a Target is intended to provide concrete values for
the Command arguments of the Target's Tasks.  For example, we might define the
name of a user account for use by a number of Tasks:-

    targets:
        make_website:
            variables:
                owner_username: "devops-one"
                docroot: ...
            tasks:
                - name: "Create owner user account"
                  command: "user:create"
                  arguments:
                      username: "{{{ owner_username }}}"
                - name: "Own the docroot"
                  command: "fs:chown"
                  arguments:
                     file: "{{{ docroot }}}"
                     user: "{{{ owner_username }}}"
                     group: "nobody"

Variables declared here can augment and override values declared in the main
`variables` directive (see [Project configuration][conf-project]) and those
declared in Modules (see [Module configuration][conf-module]).

[conf-module]: </configuration-reference/module.html> "Module configuration"
[conf-project]: </configuration-reference/project.html> "Project configuration"
[conf-task]: </configuration-reference/task.html> "Task configuration"
