# Task Arguments and Variables

There are two ways to provide Droid with data needed to execute Commands.  The
first is the direct supply of data, in the form of command arguments, when
running Droid Commands from the command line:-

```shell
$ vendor/bin/droid fs:mkdir --mode="0750" /path/to/new/directory
$ vendor/bin/droid fs:copy my-file /path/to/new/directory/
```

The second is through the use of **Task Arguments** and **Variables** when
Droid executes the Targets of a Droid Project:-

```yaml
variables:
    deployment:
        source:
            path: "my-file"
        destination:
            path: "/path/to/new/directory"
            mode: "0750"
targets:
    deploy:
        tasks:
            - name: "Create the destination"
              command: "fs:mkdir"
              arguments:
                  directory: "{{ deployment.destination.path }}"
                  mode: "{{ deployment.destination.mode }}"
            - name: "Copy my file to the destination"
              command: "fs:copy"
              arguments:
                  src: "{{ deployment.source.path }}"
                  dest: "{{ deployment.destination.path }}/"
```

The benefits of this approach are realised when we want to be able to repeat
the execution of Commands in the future, or with different data, or on multiple
Hosts.

## Task Arguments

Droid processes Task arguments to resolve them into the values it will provide
to a Command.  There are three forms of Task argument which differ by the
processing applied to them.

A literal value is not processed; it is passed unchanged to the Command:-

```yaml
command: "debug:echo"
arguments:
    message: "Hello"
```

In the above example, the `message` argument is the literal string of
characters `Hello` which will be printed by the `debug:echo` Command.

A placeholder is replaced by the corresponding Variable value:-

```yaml
command: "debug:echo"
arguments:
    message: "Hello {{ username }}"
```

In the above example, the `{{ username }}` placeholder is replaced with the
value of a Variable named `username`.

The third form of Task Argument is a path to a file which, when prefixed with
the `@` character, is replaced by the encoded content of the file at that path.
See the section entitled "**File Arguments**" for more detail.

Finally, a Task Argument can combine all three forms:-

```yaml
arguments:
    file: "@/path/to/folder/{{ filename }}"
```

### Placeholder syntax

Placeholders are processed using the Twig template engine and their syntax is
therefore [Twig syntax][twig-syntax].  Twig is greatly expressive; this is a
basic introduction to its syntax.

At its simplest, there is the replacement of a named variable with its value:-

```twig
"The sky looks {{ colour }}."
```

might be transformed to:-

```text
"The sky looks cyan."
```

There is "dot notation" for referencing nested variables or attributes of an
object; given the following variables:-

```yaml
variables:
    user_account:
        name: "alice"
        password: "5ekre7"
```

the following is transformed:-

```twig
"The password of {{ user_account.name }} is {{ user_account.password }}"
```

into:-

```text
"The password of alice is 5ekre7"
```

There are a number of useful [filters][twig-filter], such as
[`capitalize`][twig-filter-capitalize]:-

```twig
"Hello there {{ user_account.name|capitalize }}."
```

which becomes:-

```text
"Hello there Alice."
```

The [Twig syntax documentation][twig-syntax] is comprehensive and recommended
reading, especially when constructing template files.  Bear in mind, when
reading the documentation, these points about the way the Twig engine is used
by Droid:-

- HTML escaping is not done by default.  Use the [`escape`][twig-filter-escape]
  filter when escaping is a requirement.
- Twig will not silently ignore attempted use of variables or attributes that
  do not exist.  It will complain loudly and halt execution.
- Caching of templates is disabled.
- Twig expects characters in the UTF-8 character set.

### File Arguments

When a Task Argument is given as a path to a file and prefixed with the `@`
character it is replaced by the content of that file.  The content is
transformed into a base64, binary data URI. It looks like this:-

```text
data:application/octet-stream;base64,RHJvaWQgd2FzICdlcmUuCg==
```

This is primarily useful when copying a relatively small file to a remote Host,
using the `fs:copy` Command.  This mechanism avoids the need for a separate
step to copy the file from host to host: the content of the file is given
directly to the `fs:copy` Command on the remote Host, as the `src` argument.
For example, given the following Task:-

```yaml
name: "Copy a file"
command: "fs:copy"
arguments:
    src: "@/path/to/a/file"
    dest: "/usr/local/share/my-file.txt"
hosts: my_host
```

might become the following command, executed on the Host `my_host`:-

```shell
$ /usr/local/droid/vendor/bin/droid fs:copy \
  data:application/octet-stream;base64,RHJvaWQgd2FzICdlcmUuCg== \
  /usr/local/share/my-file.txt
```

Additionally, when the name of the file given as a Task Argument ends with
".template" or ".twig", Droid processes the file content as a file template.
The content is rendered by the Twig engine in exactly the same way as the
aforementioned placeholders in Task Arguments and with same set of Variables.
For example, given the following file template:-

```twig
{# This file is a template and this is a comment within it #}
Hello {{ name|capitalize }}.
```

and the following Variables and Task:

```yaml
variables:
    name: "joshua"
tasks:
    - name: "Copy a template file"
      command: "fs:copy"
      arguments:
          src: "@/path/to/some.template"
          dest: "/usr/local/share/my-file.txt"
```

the content of the file becomes the following:-

```text
Hello Joshua.
```

before being encoded into a data uri:-

```text
data:application/octet-stream;base64,SGVsbG8gSm9zaHVhLgo=
```

Read on to find out how Droid assembles a set of Variables for the arguments of
a particular Command invocation.

## Variables

Variables hold the data of a Droid Project.  They are mappings of arbitrary
names to values.  A value may be a string of characters, a number, a list or a
mapping.  For example:-

```yaml
variables:
    shape: "line"
    length: 2048
    points:
        - {x: 0}
        - {x: 512, y: 64}
        - x: 768
          y: 128
          z: -64
    point_conf:
        colour: "blue"
        size: 1
```

Droid assembles a set of Variables for each individual invocation of a Command.
It then transforms Task Arguments into the arguments to a command, replacing
placeholders with values from the assembled Variables.

### Assembling a data set

Variables may be declared in various places in a Droid Project and these are
merged in a specific order before use in Task Arguments.  Variables are merged
from the least specific to the most specific so that the result is a
combination of them all.  More specific values take the place of less specific
ones with the same Variable name.  The various Variables are, in order of
increasing specificity:-

- Module Variables
- Project Variables
- Target Variables
- Group Variables
- Host Variables

By way of example, in the following Project:-

```yaml
variables:
    dns:
        domain_name: "example.com"
        server: "ns1.example.com"
hosts:
    web-01:
        variables:
            dns:
                domain_name: "apps.example.com"
    db-master:
        variables:
            replication_role: "master"
```

the variables available to Tasks to be executed on the Host `web-01` are:-

```yaml
dns:
    domain_name: "apps.example.com"
    server: "ns1.example.com"
```

That is, the Host inherits the `dns.server` Variable and overrides the value of
`dns.domain_name`.  For the Host `db-master`:-

```yaml
dns:
    domain_name: "example.com"
    server: "ns1.example.com"
replication_role: "master"
```

That is, the Host inherits all of the values of `dns` and augments the
Variables with its `replication_role`.

A more detailed look at the various places to define Variables follows.

#### Module Variables

A Module is a reusable list of Tasks and its Variables are meant to hold
sensible default values for those Tasks.  A Project which makes use of a Module
is expected to override the default values with ones that have some meaning
within the Project.  As such, Module Variables are the least specific.

#### Project Variables

Project Variables hold the next more specific values.  They are intended to be
a central repository of Project data available to all Tasks.  Project data can
easily be augmented with extra data or, where necessary, some or all of the
data can be replaced by more specific data.

#### Target Variables

A Target is a list of Tasks which, like a recipe, is intended to achieve a
particular goal.  Target Variables hold the next more specific values and are
intended to provide to its Tasks the kind of data which aren't meaningful to
Tasks in other Targets.

Another use for Target Variables is to provide data to the Tasks of a Module
which the Target incorporates.  This use of Target Variables allows the Module
to be used in multiple Targets and to provide different data for each usage.

#### Group Variables

A Group is a logical collection of Hosts, such as a group of web servers.
Group Variables hold the next more specific values and are intended for the
association of data with all members of the Group.  The Variables of a
particular Group are available to only those Task Arguments applicable to its
member Hosts.  The Variables of all Groups of which a Host is a member are made
available to the applicable Task Arguments.

#### Host Variables

Host Variables hold the most specific values and are intended to apply to a
particular Host.  When a Command is to be executed on a Host, its Variables are
available to the Task Arguments, along with all that it inherits.

### Dynamic Variables

Sometimes the value of a Variable needs to be some attribute of a Host or Group
in Droid's inventory.  For example, a number of Tasks might need to refer to
the `public_ip` address of a Host.  This is achieved with the dynamic variable
syntax, which is an expression surrounded by a pair of `%` characters:-

```yaml
variables:
    web_servers:
        - "%web-01.public_ip%"
        - "%web-02.public_ip%"
```

Such expressions are evaluated during the initial processing of the Droid
Project configuration.  A name, which precedes the first dot, is resolved to
either a Host or a Group in Droid's inventory.  Then a named attribute, which
is everything after the first dot, is resolved to an attribute of the resulting
object.

### Magic data

There are occasions when a magic value is inserted into the set of data
available to the Task Arguments of a particular invocation.  These values are
inserted in specific circumstances and are named:

- `host`
- `item`
- `mod_path`

#### `host`

The `host` magic value is made available to Task Arguments for a Command to be
executed on a Host.  Its value is an object which represents that Host.  The
Task Arguments are therefore able to refer to attributes of that Host by using
Twig's "dot notation".  For example:-

```yaml
name: "Print some info about a Host"
command: "debug:echo"
arguments:
    message: "I am {{ host.name }} at {{ host.public_ip }}."
hosts: "web_servers"
```

The above example will run the command on each Host in a Group named
`web_servers` and the name and IP address of the Host will be printed in each
case.

#### `item`

The `item` magic value is made available to Task Arguments for a Command
intended to be executed repeatedly, each time with a different item of data.
Its value is the current item of data.  For example:-

```yaml
name: "Touch some files"
command: "fs:touch"
arguments:
    file: "/var/www/{{ item.name }}"
    mtime: "{{ item.time }}
with_items:
    - {path: "index.php", time: "now"}
    - {path: "index.htm", time: "last Tuesday"}
```

#### `mod_path`

The `mod_path` magic value is made available to Task Arguments of Tasks in
Modules.  The value is the path to the root of the Module installation and is
intended to enable Modules to locate assets, such as file templates, which they
provide.

[twig-syntax]: <http://twig.sensiolabs.org/doc/templates.html> "Twig for Template Designers"
[twig-filter]: <http://twig.sensiolabs.org/doc/filters/index.html> "Twig Filters"
[twig-filter-capitalize]: <http://twig.sensiolabs.org/doc/filters/capitalize.html> "Twig filter: capitalize"
[twig-filter-escape]: <http://twig.sensiolabs.org/doc/filters/escape.html> "Twig filter: escape"
