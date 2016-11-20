# Modules

A Module is a self-contained group of [Tasks]() intended to achieve a particular
goal. Optionally, a Module can provide file assets such as configuration
templates for use by its Tasks.

A Module must be first installed in a Droid Project, and then included in one
or more [Targets]().

Droid can use locally developed Modules and those published in git
repositories.  A number of [Modules have already been published as part of the Droid Project
on GitHub][github-modules].  Examples are [droid-module-timezone][] which
reconfigures the tzdata platform package to set the time zone,
[droid-module-apache-vhost][] which will set-up an Apache 2 virtual host, and
[droid-module-mysql-repln][] which will configure a cluster of MySQL servers
for Binary Log replication.

We will now see how to use an existing Module and how to create a new one.

## Using a Module

A Module must be first installed in a Droid Project, and then included in one
or more Targets.

### Installation

To install a Module, we first register it with the Project by adding an entry
in the section `modules` of the [Project Configuration file](conf-project).  We provide a name by
which we will later refer to the Module (in this example: "timezone") and provide the source URL from where
it is obtained:

```yaml
name: "A Droid Project"
description: "This is an example of using a Module in a Droid Project"
modules:
    "timezone": "git@github.com:droid-php/droid-module-timezone.git"
```

Next we execute the `module:install` Command at the command line:

```shell
$ vendor/bin/droid module:install
```

This should result in the following output as the Module is installed:

```shell
Installing droid modules:
- timezone from git git@github.com:droid-php/droid-module-timezone.git
Cloning from git@github.com:droid-php/droid-module-timezone.git into droid-vendor/timezone
Cloning into 'droid-vendor/timezone'...
remote: Counting objects: 16, done.
remote: Total 16 (delta 0), reused 0 (delta 0), pack-reused 16
Receiving objects: 100% (16/16), done.
Resolving deltas: 100% (6/6), done.
Checking connectivity... done.
  Switching to branch master
Done
```

The Module is installed in its own folder, within the Project's
`droid-vendor` folder.  When Droid encounters a reference to the Module, it
will look for it in a folder of the same name in two locations:

- `droid-vendor`: where Droid installs Modules obtained from git repositories.
- `modules`: where locally developed Modules should be placed. See the section
  entitled "Create a Module", for more information.

Now that the Module is installed, we can include it in one or more Targets.

### Including a Module in a Target

The Tasks of a Module are executed as part of a Target in which the Module is
included.  To include a Module in a Target, we add an entry to the section `modules` of the
[Target Configuration file][conf-target].  We provide the same name we earlier gave
to the Module when it was registered with the Project:

```yaml
targets:
    make_website:
        modules:
            - "timezone"
```

To complete the inclusion, we must provide any data required by the Module.
In this example, we want to provide time zone information.  According to the [timezone
Module documentation][droid-module-timezone-doc], we should provide
values for two Variables:

```text
timezone:
    area: <string>  # for example, "Europe"
    zone: <string>  # for example, "Amsterdam"
```
We can provide the Module variables in one of the following locations:
- as Project Variables that are accessible to all Tasks in the Project;
- as Target variables that are accessible to a Module in more than one Target and with different values.
- as Host or Group Variables if we wished to run the Module on a number of Hosts, each with differing values.
For more information on declaring variables, refer to [xxxx]().

In this example, we will provide Module variables as Target Variables.
The Project configuration is now as follows:

```yaml
name: "A Droid Project"
description: "This is an example of using a Module in a Droid Project"
modules:
    "timezone": "git@github.com:droid-php/droid-module-timezone.git"
targets:
    variables:
        timezone:
            area: "Europe"
            zone: "Berlin"
    make_website:
        modules:
            - "timezone"
```

The Module is now included in the Target `make_website`; when the Target is run, the Tasks of the Module will be executed.

Multiple Modules may be included in a Target in this way. Their order of
execution is the order in which they appear in the list of Target `modules`.

The priority of execution is as follows:
1. the Tasks of Modules;
2. the Tasks of the Target.

So, the Target Task in the following example would display "Europe/Berlin" instead of
the system value prior to changes made by the Module:

```yaml
targets:
    variables:
        timezone: {area: "Europe", zone: "Berlin"}
    make_website:
        tasks:
            - command: "shell:exec"
              arguments:
                  command-line: "head /etc/timezone"
        modules:
            - "timezone"
```

The order in which `tasks` and `modules` were declared in this example is
therefore confusing: it is clearer when we declare them the other way round.

We have seen how to use a Module in a Project.  Let us now look at how to
create one.

## Creating a Module

To demonstrate the creation of a Module, let us go through the process from the
beginning.  We shall see how to create a Module that will set-up scheduled
backups using the duplicity backup software.  The requirements of the Module
are that it should:-

- provide for three cron jobs: full backup, incremental backup and removal of
  old backups
- allow the backup schedule to be configurable
- allow configuration of a list of files to be excluded from the backup
- allow configuration of the destination folder for the backup files
- allow configuration of the number of backup sets to keep
- allow configuration of the backup name

To keep this example from getting too in-depth, we will:-

- backup as the root user: our Module won't need to create user accounts or
  deal with file access privileges
- backup the root of the file system
- set the backup destination as somewhere on the same file system instead of
  dealing with the various destination options duplicity provides

Let's begin. First we will create a folder for the Module in an existing Droid
Project.  We shall name our module "duplicity-backup":-

```shell
$ cd myproject
$ mkdir -p modules/duplicity-backup
```

We create a [Module Configuration][conf-module] at
`modules/duplicity-backup/droid.yml` with the following content:-

```yaml
description: "Set up duplicity backup"
variables:
    mod_duplicity_backup: {}
tasks: []
```

We've given a description of the module and defined a Variable named
`mod_duplicity_backup`.  The Variable will be used to provide data to the Tasks
and can be overridden in the Projects which use the Module.  The name of the
Variable is a convention which helps make it clear that the data pertains to
this Module.

Our first Task will install the `duplicity` package.  For this we will use the
`apt-get:install` Droid Command.  Its help text shows us its arguments and
options:-

```shell
$ vendor/bin/droid help apt-get:install
Usage:
  apt-get:install <package>

Arguments:
  package               Name of the package(s) to install

Options:
  -h, --help            Display this help message
  -q, --quiet           Do not output any message
  -V, --version         Display this application version
      --ansi            Force ANSI output
      --no-ansi         Disable ANSI output
  -n, --no-interaction  Do not ask any interactive question
  -v|vv|vvv, --verbose  Increase the verbosity of messages: 1 for normal output, 2 for more verbose output and 3 for debug

Help:
  Installs package(s) through apt-get on Debian based systems
```

So we add our first Module Task:-

```yaml
description: "Set up duplicity backup"
variables:
    mod_duplicity_backup: {}
tasks:
    - name: "Install duplicity"
      command: "apt-get:install"
      sudo: true
      arguments:
          package: "duplicity"
```

We use the `sudo` property of the Task so that the command is executed with the
privileges it demands.

Next, we want to be able to provide a file to duplicity which will contain a
list of files to exclude from the backup.  We will create a file template that
our next Task will populate and copy to somewhere duplicity can read it.  We
need a Variable to provide a list of excluded files and we can provide a
sensible default list:-

```yaml
variables:
    mod_duplicity_backup:
        excluded_files:
            - "/dev/*"
            - "/media/*"
            - "/proc/*"
            - "/run/*"
            - "/sys/*"
            - "/tmp/*"
```

Now we need a file template into which we'll write the list of excluded files.
We create a folder within the Module to hold such "assets" as this:-

```shell
$ mkdir modules/duplicity-backup/assets
```

and we create a template file `assets/excluded-files.list.template` with the
following content:-

```twig
{% for file in mod_duplicity_backup.excluded_files %}
{{ file }}
{% endfor %}
```

The content of the template is [Twig syntax][twig-syntax].  It will cause each
entry of the `mod_duplicity_backup.excluded_files` Variable to be printed on
its own line.

We must make sure we don't backup the backup!.  Let's add another Variable to
hold the path to the backup destination (again, providing a sensible default)
and then make sure this is also written into our excluded files list:

```yaml
variables:
    mod_duplicity_backup:
        ...
        destination: "/var/backup"
```

Our template now looks like this:-

```twig
{% for file in mod_duplicity_backup.excluded_files %}
{{ file }}
{% endfor %}
{{ mod_duplicity_backup.destination }}
```

We will use the `fs:copy` Droid Command to copy the content of the file to the
Host.  Droid will use the Twig template engine to transform the template into
the desired content.  Here is the next Module Task:-

```yaml
tasks:
    ...
    - name: "Copy the list of excluded files"
      command: "fs:copy"
      sudo: true
      arguments:
          src: "@{{ mod_path }}/assets/excluded-files.list.template"
          dest: "/root/excluded-files.list"
```

Note that we have used a the name of a magic Variable in the `src` Task
Argument: `mod_path` is the path to the Module's folder and allows a Module to
locate itself and its assets.

Next, we schedule the duplicity backups.  We need a few more Variables:-

```yaml
variables:
    mod_duplicity_backup:
        ...
        name: "backup"
        max_sets: 2
        schedule:
            cleanup: "30 12 * * mon"
            full: "30 5 * * mon"
            incremental: "30 5 * * 2-7"
```

We provided some sensible default values for the name of the backup; for the
maximum number of backup sets to keep when older backups are removed; and for
the schedules for each of the backup jobs, in the usual cron format.  We will
use the `cron:addjob` Droid Command to add the three jobs to the crontab:-

```yaml
tasks:
    ...
    - name: "Schedule the removal of old backups"
      command: "cron:addjob"
      sudo: true
      arguments:
          name: "{{ mod_duplicity_backup.name }}_cleanup"
          schedule: "{{ mod_duplicity_backup.schedule.cleanup }}"
          username: "root"
          job-command: >
                         /usr/bin/duplicity
                         remove-all-but-n-full {{ mod_duplicity_backup.max_sets }}
                         --name {{ mod_duplicity_backup.name }} --force
                         file://{{ mod_duplicity_backup.destination }}
    - name: "Schedule the full backup"
      command: "cron:addjob"
      sudo: true
      arguments:
          name: "{{ mod_duplicity_backup.name }}_full"
          schedule: "{{ mod_duplicity_backup.schedule.full }}"
          username: "root"
          job-command: >
                         /usr/bin/duplicity full --no-encryption
                         --no-print-statistics
                         --name {{ mod_duplicity_backup.name }}
                         --exclude-globbing-filelist /root/excluded-files.list
                         / file://{{ mod_duplicity_backup.destination }}
    - name: "Schedule the incremental backup"
      command: "cron:addjob"
      sudo: true
      arguments:
          name: "{{ mod_duplicity_backup.name }}_incremental"
          schedule: "{{ mod_duplicity_backup.schedule.incremental }}"
          username: "root"
          job-command: >
                         /usr/bin/duplicity incr --no-encryption
                         --no-print-statistics
                         --name {{ mod_duplicity_backup.name }}
                         --exclude-globbing-filelist /root/excluded-files.list
                         / file://{{ mod_duplicity_backup.destination }}
```

The `job-command` Task Argument takes advantage of Yaml's "folded style" which
will result in one (long) line with the new lines and extra white space
removed.

Our Module is now functionally complete and can be integrated into a Target in
our Droid Project.  Let's create the Project configuration file:-

```yaml
name: "Testing the duplicity-backup Module"
modules:
    "duplicity-backup": ""
targets:
    setup_backup:
        hosts: my-test-machine
        modules:
            - "duplicity-backup"
hosts:
    my-test-machine:
        droid_ip: "198.51.100.1"
variables:
    mod_duplicity_backup:
        excluded_files:
            - "/dev/*"
            - "/media/*"
            - "/proc/*"
            - "/run/*"
            - "/sys/*"
            - "/tmp/*"
            - "/usr/local/droid/*"
```

In the Project configuration we have registered the Module with the Droid
Project by providing an entry in the Project `modules`.  Since this is a
locally developed Module there is a need neither to provide a source URL for
it, nor to install the Module with the `module:install` Command.  We do need to
make sure that the name we gave matches the Module's folder name, so that Droid
can locate it.  We have included the Module in a Target named `setup_backup`
which execute Commands on the Host named `my-test-machine`.  Finally, we
override one of the Variables `mod_duplicity_backup.excluded_files` so that we
may exclude some additional files.

Our Project folder should now look like this:-

```text
myproject
|_ droid.yml
|_ modules
   |_ duplicity-backup
      |_ droid.yml
      |_ assets
         |_ excluded-files.list.template
```
We are ready to run the `setup_backup` Target:-

```shell
$ vendor/bin/droid setup_backup
```

We should see something like the following output (this example has been
shortened for clarity):-

```shell
Droid: Running target `setup_backup`
Task `Install duplicity`: apt-get:install on my-test-machine
Host my-test-machine: package=duplicity
my-test-machine Begin droid enablement.
my-test-machine Finished droid enablement. Success.
my-test-machine apt-get install -y duplicity
my-test-machine Reading package lists...
...
my-test-machine The following NEW packages will be installed:
my-test-machine   duplicity librsync1 python-lockfile
...
Task `Copy the list of excluded files`: fs:copy on my-test-machine
Host my-test-machine: src=data:application/octet-stream;base64,L2i...
my-test-machine {"changed":true,"start":1478215336.3923,"end":1478215336.3924}
Task `Schedule the removal of old backups`: cron:addjob on my-test-machine
Host my-test-machine: name=backup_cleanup schedule=30 12 * * mon ...
my-test-machine I have successfully created the job "droid_backup_cleanup".
my-test-machine {"changed":true,"start":1478215337.0049,"end":1478215337.0057}
Task `Schedule the full backup`: cron:addjob on my-test-machine
Host my-test-machine: name=backup_full schedule=30 5 * * mon username=root ...
my-test-machine I have successfully created the job "droid_backup_full".
my-test-machine {"changed":true,"start":1478215337.6269,"end":1478215337.6277}
Task `Schedule the incremental backup`: cron:addjob on my-test-machine
Host my-test-machine: name=backup_incremental schedule=30 5 * * 2-7 ...
my-test-machine I have successfully created the job "droid_backup_incremental".
my-test-machine {"changed":true,"start":1478215338.2342,"end":1478215338.2348}
Result: 0
--------------------------------------------
```
Success!  Duplicity is installed and three jobs were placed into `/etc/cron.d`:
one for the cleanup; one for the full backup; and one for the incremental.

This Module may now be reused in this or in other Projects and is easily
configured with different schedules, excluded files and so on.

[conf-module]: </configuration-reference/module.html> "Module Configuration"
[conf-project]: </configuration-reference/project.html> "Project Configuration"
[github-modules]: <https://github.com/droid-php/?query=droid-module-> "Droid Modules at GitHub"
[droid-module-timezone]: <https://github.com/droid-php/droid-module-timezone> "A Droid module to change the time zone"
[droid-module-timezone-doc]: <https://github.com/droid-php/droid-module-timezone/blob/master/README.md> "Documentation of the timezone Droid module"
[droid-module-apache-vhost]: <https://github.com/droid-php/droid-module-apache-vhost> "A Droid module to generate and enable Apache2 virtual hosts"
[droid-module-mysql-repln]: <https://github.com/droid-php/droid-module-mysql-repln> "A Droid module to configure MySQL Server for Binary Log Replication"
[twig-syntax]: <http://twig.sensiolabs.org/doc/templates.html> "Twig for Template Designers"
