# Task configuration

A Task provides Droid with the information it requires to execute a command on
a Host.  It provides the name of a command to execute along with the command
arguments.  It also provides information to control whether the Task should be
executed, such as with the `hosts` and `hosts_filter` properties; how the Task
should be executed and how the Task is executed, such as with the `max_runtime`
and `sudo` properties; and it allows the command to be run repeatedly, with the
`with_items` property.

## `name`

The `name` property of a Task serves two purposes:-

1. a human readable clue to the purpose of a Task
2. Tasks which trigger the execution of other Tasks refer to them by this name
   (see the `trigger` property, below)

The name of a Task is displayed, at the time of execution, in the command line
output.

## `command`

The `command` property of a Task is the name of a command to be executed as it
would be given at the command line.  A list of available Commands may be
obtained with Droid's `list` command:-

```shell
$ vendor/bin/droid list
...
Available commands:
 list                   Lists commands
...
 apache
  apache:dismod         Disable Apache modules.
  apache:dissite        Disable Apache sites.
  apache:enmod          Enable Apache modules.
  apache:ensite         Enable Apache sites.
...
```

The names of the commands appears in the left-hand column.  For example:-

```yaml
tasks:
    - name: "Enable my website"
      command: "apache:ensite"
      ...
```

## `arguments`

The `arguments` property of a Task are the required and optional arguments
passed to a Command when it is executed.  It is a mapping of argument names to
their values.  The names and types of arguments of a command may be obtained
with Droid's `help` command:-

```shell
$ vendor/bin/droid help cron:addjob
Usage:
  cron:addjob [options] [--] <name> <schedule> <username> <job-command>

Arguments:
  name                  The name of the job.
  schedule              The job schedule.
  username              The name of the user account under which to run the job command.
  job-command           The job command.

Options:
      --force           Overwrite an existing job of the same name.
      --mail=MAIL       Comma separated list of user names or email addresses to which to email the job output.
...
```

For example we might define a Task to add a cron job to ping some host every
minute of every day:-

```yaml
tasks:
    - name: "Add cron job to Ping a host"
      command: "cron:addjob"
      arguments:
          name: "Ping service.example.com once a min."
          schedule: "* * * * *"
          username: "pingu"
          job-command: "ping -c 1 service.example.com"
          force: ~
          mail: "alerts@admin.example.com"
```

Notice that both positional and optional arguments are given in the same way
and that options that do not take a value, such as with `--force` in this
example, are given with the `~` (tilde) character which is Yaml's `NULL`
character.

Task arguments may use placeholders which are replaced by the value of a
variable immediately before the arguments are passed to a command.  For
example:-

```yaml
tasks:
    - name: "Add cron job to Ping a host"
      command: "cron:addjob"
      arguments:
          name: "Ping {{ service_to_ping }} once a min."
          job-command: "ping -c 1 {{ service_to_ping }}"
          ...
```

In the above example, `{{ service_to_ping }}` is the placeholder and will be
replaced by the value of a variable name `service_to_ping`.  The variable may
be defined as a Project variable, a Target variable or, if the Task is to be
executed on a remote Host, a Host or Group variable.  Using placeholders can
ease the the maintenance of Tasks by reducing the need to change a particular
value in all the places it may appear in a Project.

Please see [Task Arguments and Variables][task-args] for more information about
how Variables and Task Arguments are used.

## `hosts`

The `hosts` property of a Task is an expression which identifies the Hosts and
Groups of Hosts on which the Task is to be executed.  The expression may be a
single name or a space or comma separated list of names.  The names must match
those declared in the Project, under the `hosts` and `groups` directives.

```yaml
target:
    setup_firewalls:
        tasks:
            - name: "Install Uncomplicated Firewall"
              command: "apt-get:install"
              ...
              hosts: "webservers db.example.com"
```

It is not necessary to give a value for this property if the Task is intended
to execute on the local.

It is not necessary to give a value for this property if the Task is one that
will be triggered by another Task: the triggered Task will execute on the same
set of Hosts as the triggering Task in that case.

## `host_filter`

The `host_filter` property of a Task allows Droid to make a decision about
whether to execute the Task on a particular Host.  Its value should be an
expression that evaluates to `true` if the Task should run on a particular
Host.  The expression is able to test various properties of the Host under
consideration.  For example we might wish to execute a Task on only those hosts
with a certain variable value:-

```yaml
hosts:
    web1:
        variables:
            role: "primary"
    web2:
        variables:
            role: "secondary"
targets:
    make_website:
        hosts: "web1 web2"
        tasks:
            - name: "Enable site"
              host_filter: "host.variables['role'] == 'primary'"
              ...
```

Notice that, in the example above, the Tasks of the `make_website` Target are
intended to be executed on the two Hosts `web1` and `web2`.  Each of the Hosts
defines a `role` variable and the "Enable site" Task uses the `host_filter` to
test the value of that variable on each Host.  If the value of the variable is
"primary" then the expression is true and execution proceeds on that Host.

Notice also that the expression given in the example above uses two different
notations to access the `role` variable.  First we access the Host's variables
by using "dot notation":

```yaml
host.variables
```

and we can access other properties of a host in the same way:-

```yaml
host.public_ip
host.username
```

Second, the variables property is a mapping, or array, and we use "subscript
notation" to access values of the mapping:-

```yaml
variables['role']
```

and we can access nested values in the same way:-

```yaml
variables['dbuser']['name']
variables['dbuser']['password']
```

Please see the [Host configuration reference][conf-host] for the properties of
a Host available for use in these expressions.  For the full syntax of these
expressions, please see the [ExpressionLanguage Component
documentation][symfony-expr-syntax].

## `with_items`

The `with_items` property of a Task instructs Droid to repeatedly execute the
Task, once for each item in the list of data given as the value of this
property.  For example we might declare a Task to create a number of
directories:-

```yaml
target:
    make_website:
        tasks:
            - name: "Create directories"
              command: "fs:mkdir"
              arguments:
                  directory: {{ item }}
              ...
              with_items:
                  - "/var/www/mywebsite"
                  - "/var/db/mywebsite"
```

Notice that the `directory` argument to the command uses the placeholder `{{
item }}` which refers to the current item in the list of `with_items` on each
execution.  The value of `with_items` may also be the name of a variable:-

```yaml
target:
    make_website:
        variables:
            webdirs:
                - "/var/www/mywebsite"
                - "/var/db/mywebsite"
        tasks:
            - name: "Create directories"
              ...
              with_items: "webdirs"
```

The items in the list may also be mappings of multiple values:-

```yaml
target:
    make_website:
        variables:
            webdirs:
                - {path: "/var/www/mywebsite", mode: "0750"}
                - {path: "/var/db/mywebsite", mode: "0700"}
        tasks:
            - name: "Create directories"
              arguments:
                  directory: {{ item.path }}
                  mode: {{ item.mode }}
              ...
              with_items: "webdirs"
```

## `with_items_filter`

The `with_items_filter` property of a Task allows Droid to make a decision
about whether to execute the Task with a particular item.  Its value should be
an expression that evaluates to `true` if the Task should run with a particular
item.  The expression is able to test various values of the item under
consideration.  For example we might wish to execute a Task with only those
items having a particular named value:-

```yaml
target:
    make_website:
        variables:
            websites:
                - {path: "/var/www/mywebsite", ssl_cert: "/etc/ssl/mywebsite.cert"}
                - {path: "/var/www/myotherwebsite"}
        tasks:
            - name: "Install SSL certificate"
              ...
              with_items: "websites"
              with_items_filter: "item['ssl_cert']"
```

In the `make_website` Target in the above example, the "Install SSL
certificate" Task is run repeatedly for each item in the list at the
variable named `websites`.  The Task will execute only with those items for
which the `with_items_filter` evaluates to `true`: items having a value for
`ssl_cert` in this case.

For the full syntax of these expressions, please see the [ExpressionLanguage
Component documentation][symfony-expr-syntax].

## `max_runtime`

The `max_runtime` property of a Task may be used to extend the amount of time
for which its Command is permitted to execute.  The value is a number of
seconds and the default is 60.

## `sudo`

A command is run with the privileges of a super user when the `sudo` property
of a Task is `true`.  The user account under which Droid is running must be
permitted to execute commands with sudo (see how in [Enable remote command
execution][remote-exec]).

## `trigger`

The `trigger` property of a Task is used to trigger the execution of another
Task.  The value is the name of the Task that will be triggered.  A Task may
trigger another when its Command is able to report whether or not a change was
made.  Such commands output machine-readable information when they complete and
are usually those which support the `--check` option.

[conf-host]: </configuration-reference/host.html> "Host configuration"
[symfony-expr-syntax]: <https://symfony.com/doc/current/components/expression_language/syntax.html> "The Expression Syntax"
[remote-exec]: </enable-remote-command-execution.html> "Enable remote command execution"
[task-args]: </task-arguments-and-variables.html> "Task Arguments and Variables"
