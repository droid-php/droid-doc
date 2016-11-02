# Module configuration

A Module, like a Target, is essentially a list of Tasks which Droid will
execute to achieve a desired outcome.  A Module is intended to be "imported"
into a Target and to be reusable.

The configuration of a Module lives in its own directory, in a file named
`droid.yml`.

## `description`

The `description` property of a Module is intended to convey the purpose of the
Module, that is, what it will accomplish.

## `tasks`

This `tasks` property of a Module is a list of Tasks which will be executed in
the order of their definition.

    tasks:
        - name: "Install Apache"
          ...
        - name: "Configure Apache"
          ...

Please see the [Task configuration][conf-task] for the configuration of a Task.

## `triggers`

The `triggers` property of a Module is a list of Tasks whose execution depends
upon the outcome of the execution of other Tasks.  A Task can trigger the
execution of another Task when its Command is able to report whether or not it
made a change.  For example, a Task which modifies the configuration of a
system service may cause the service to reload its configuration:-

    tasks:
        - name: "Enable Apache module"
          command: "apache:enmod"
          arguments: {module-name: ...}
          trigger: "Reload Apache"
    triggers:
        - name: "Reload Apache"
          command: "service:reload"
          arguments: {name: "apache2"}

A Task is triggered only when such a Command reports that a change was made.

Apart from its conditional execution, a `triggers` Task is no different from a
`tasks` Task.  Please see the [Task configuration][conf-task] for the
configuration of a Task.

## `variables`

The `variables` property of a Module is intended to provide sensible default
values for the Command arguments of its Tasks.  Variables declared here will,
when desired, be overridden by the Projects which make use of the Module.

By way of example, consider a Module in which a Task will create a number of
directories as part of a larger objective.  The Task will use the `with_items`
property and use the name of a variable which will hold the list of
directories:-

    tasks:
        - name: "Create directories"
          command: "fs:mkdir"
          arguments:
              directory: {{ item.path }}
              mode: {{ item.mode }}
          with_items: mod_fantastic_dirs

The module should provide sensible default values for the variables it uses:-

    variables:
        mod_fantastic_dirs:
            - {path: "/var/www/fantastic", mode: "0700"}
    tasks:
        - name: "Create directories"
          ...
          with_items: mod_fantastic_dirs

Projects which use the Module may be happy to use the default values, or they
may set their own:-

    targets:
        do_something_fantastic:
            variables:
                mod_fantastic_dirs:
                    - {path: "/usr/share/fantastic", mode: "0550"}
            modules:
                - "fantastic"

[conf-task]: </configuration-reference/task.html> "Task configuration"
