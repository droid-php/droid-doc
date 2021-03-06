# Introduction

Droid executes Commands.  The Commands to be executed and the computers on
which to execute them are described in a human readable configuration file.
Droid reads the configuration file and executes the Commands exactly as
described.

Droid is employed to:-

- Automate the provisioning of IT infrastructure
- Deploy applications
- Manage IT infrastructure configuration

## Concepts

### Commands

Droid is a [Symfony Console Application][symfony-console].  The Droid Standard
meta package provides a variety of Droid Commands which are [Symfony Console
Commands][symfony-command].  Droid Commands may be executed from the console, or
they may be invoked as part of a Droid configuration.

### Tasks

A Task is part of a Droid configuration and instructs Droid to execute a
Command, providing the command Arguments and other meta data.  A Task also
specifies the computers, or Hosts, on which the Command will execute.

### Targets

A Target is part of a Droid configuration and is an ordered list of Tasks, like
a recipe.  A Target is intended to achieve a specific goal, such as deploying a
web application to a web server.  Droid reads each Task of a Target in turn and
determines on which Hosts to execute the associated Command.  Droid executes
the Command on each Host, in parallel.

### Inventory - Hosts and Groups

A Host is part of a Droid configuration and describes a computer, real or
virtual, on which Commands may be executed.  Hosts may be grouped logically
into Groups, such as all web servers, so that Commands may be executed on all
members of the Group.

## Where to go from here

[Get started][get-started]!

[index]: </index.html> "Introduction"
[get-started]: </getting-started.html> "Getting Started"
[task-args]: </task-arguments-and-variables.html> "Task Arguments and Variables"
[remote-exec]: </enable-remote-command-execution.html> "Enable Remote Command Execution"
[conf-index]: </configuration-reference/index.html> "Configuration Reference"
[conf-fw]: </configuration-reference/firewall-rule.html> "Firewall Rule Configuration"
[conf-group]: </configuration-reference/group.html> "Group Configuration"
[conf-host]: </configuration-reference/host.html> "Host Configuration"
[conf-module]: </configuration-reference/module.html> "Module Configuration"
[conf-project]: </configuration-reference/project.html> "Project Configuration"
[conf-target]: </configuration-reference/target.html> "Target Configuration"
[conf-task]: </configuration-reference/task.html> "Task Configuration"
[hacking]: </hacking.html> "Hacking"
[modules]: </modules.html> "Modules"
[command-index]: </command-reference/index.html> "Command Reference"
[command-plugins]: </command-plugins.html> "command-plugins"
[symfony-console]: <https://symfony.com/doc/current/components/console.html> "The Console Component"
[symfony-command]: <https://symfony.com/doc/current/console.html> "Console Commands"
