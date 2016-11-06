# Configuration reference

## Table of Contents

- [Project Configuration][conf-project]
- [Target Configuration][conf-target]
- [Task Configuration][conf-task]
- [Host Configuration][conf-host]
- [Group Configuration][conf-group]
- [Firewall Rule Configuration][conf-fw]
- [Module Configuration][conf-module]

## Droid Project

The configuration of a Droid Project lives in file named `droid.yml` in the
root folder of the Project.  The format of the file is [YAML][].

The Project configuration directives are described in the [Project
configuration reference][conf-project].  Targets and Tasks, which direct Droid
to do work, are described in the [Target configuration reference][conf-target]
and [Task configuration reference][conf-task].  Hosts and Groups, the Inventory
managed by Droid, are described in the [Host configuration
reference][conf-host] and [Group configuration reference][conf-group].

## Droid Module

The configuration of a Droid Module lives in file named `droid.yml` in the
root folder of the Module.  Like a Droid Project configuration, the format of
the Module configuration file is [YAML][].

The Module configuration directives are described in [Module configuration
reference][conf-module].

[conf-fw]: </configuration-reference/firewall-rule.html> "Firewall Rule configuration"
[conf-group]: </configuration-reference/group.html> "Group configuration"
[conf-host]: </configuration-reference/host.html> "Host configuration"
[conf-module]: </configuration-reference/module.html> "Module configuration"
[conf-project]: </configuration-reference/project.html> "Project configuration"
[conf-target]: </configuration-reference/target.html> "Target configuration"
[conf-task]: </configuration-reference/task.html> "Task configuration"
[YAML]: <http://www.yaml.org/spec/1.2/spec.html> "YAML Ain't Markup Language (YAML&#8482;) Version 1.2"
