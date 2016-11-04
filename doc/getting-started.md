# Getting started

## A first Droid project

Let's see how to use Droid to set-up a web server.  To begin we will create a
new [Composer][] project and install Droid along with a number of standard
Droid commands:-

    $ composer init -n
    $ composer require droid/droid-standard

Next we will create a new Droid project by creating a `droid.yml` file with the
following contents:-

    project:
        name: "My first Droid project"

    targets:
        make-webserver:
            tasks:
                -
                  name: "Install Apache2"
                  command: "apt-get:install"
                  sudo: true
                  arguments:
                      package: "apache2"

And now we may execute the tasks of the `make-webserver` Target:-

    $ vendor/bin/droid make-webserver
    Droid: Running target `make-webserver`
    Task `Install Apache2`: apt-get:install package=apache2  locally
    localhost apt-get install -y apache2
    ... (much output) ...
    localhost  * Starting web server apache2
    ...
    Result: 0
    --------------------------------------------
    $

We have installed the Apache 2 package and its dependencies by running `apt-get
install -y apache2` on the local host.  We did this by writing a `droid.yml`
[Yaml][] configuration file which declares a Droid project and a Target which
we named `make-webserver`.  A Target is like a recipe or a script and defines a
number of steps that Droid must take to achieve a desired outcome.  We have
defined one Task, which we named "Install Apache2", to invoke the
`apt-get:install` Droid command.  We can see that `apt-get:install` takes one
argument, `package`, by invoking the Droid help command:-

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

Finally, we declared that we desire the command to be executed with elevated
privileges by setting `sudo: true`.  The output generated during execution of
the `make-webserver` Target begins by displaying the name of the Target and
then information about and output from each of the Tasks defined by the Target.

Now suppose we want to install Apache 2 on two remote machines: let us see how
to use Droid Inventory to manage remote installation.

## Inventory

Droid is able to execute Commands on remote hosts with a little extra
configuration, but first it needs us to prepare the hosts by installing PHP,
setting-up SSH public key authentication and granting password-less sudo
privileges for the commands we wish to execute.  Please see how to [enable
remote command execution][remote-exec] for more on this.

Let's assume that our remote hosts are suitably prepared; we will need to
provide Droid with the name of a user account on the hosts and paths to private
ssh key files.  We will add two top-level directives, `groups` and `hosts`, to
our `droid.yml`:-

    groups:
        webservers:
            hosts:
                - web-01
                - web-02
    hosts:
        web-01:
            public_ip: "198.51.100.5"
            username: "c3po"
            keyfile: "/path/to/c3po_id_rsa"
        web-02:
            public_ip: "198.51.100.6"
            username: "r2d2"
            keyfile: "/path/to/r2d2_id_rsa"

We have declared two hosts which we named `web-01` and `web-02` and we grouped
them under the name `webservers`.  Now we may declare that our `make-webserver`
Target should execute remotely at these hosts:-

    targets:
        make-webserver:
            hosts: webservers
            tasks:
                -
                  ...

We invoke Droid as before:-

    $ vendor/bin/droid make-webserver
    Droid: Running target `make-webserver`
    Task `Install Apache2`: apt-get:install on webservers
    Host web-01: package=apache2
    Host web-02: package=apache2
    web-01 Begin droid enablement.
    web-01 Finished droid enablement. Success.
    web-02 Begin droid enablement.
    web-02 Finished droid enablement. Success.
    web-01 apt-get install -y apache2
    web-02 apt-get install -y apache2
    ... (much output) ...
    web-01  * Starting web server apache2
    ... (much output) ...
    web-02  * Starting web server apache2
    ...
    Result: 0
    --------------------------------------------
    $

We see that Droid has installed Apache 2 on the two hosts in our Inventory.  In
order to complete each Task, Droid on our local machine will invoke the
corresponding Droid command on the hosts in our Inventory.  Droid will first
make sure that it is installed, at the same version number and with the same
available commands, on those hosts.


## Going further: configure Apache

Let's go further and set-up two Apache virtual hosts on both of our web
servers.  For each virtual host we will install a virtual host config file,
create a document root directory, deploy an index page and enable the virtual
host.  Finally, we will trigger Apache to reload its configuration.

We begin by creating a directory for the file assets in our Droid project:-

    $ mkdir assets

We create a basic HTML index page:-

    $ cat assets/index.html
    <html>
    <head><title>Coming soon!</title></head>
    <body><p>Coming soon!</p></body>
    </html>

and a template for the virtual host config file:-

    $ cat assets/vhost.conf.template
    # {{ item.name }}
    <VirtualHost *:80>
      ServerName "{{ item.name }}"
      DocumentRoot "{{ item.docroot }}"
      <Directory "{{ item.docroot }}">
        Options Indexes FollowSymLinks MultiViews
        AllowOverride All
        Require all granted
      </Directory>
      ErrorLog "/var/log/apache2/{{ item.name }}_error.log"
      CustomLog "/var/log/apache2/{{ item.name }}_access.log" combined
    </VirtualHost>

This vhost config template contains placeholders `{{ item.name }}` and `{{
item.docroot }}` which will be replaced with concrete values.  We will add to
our `droid.yml` a Target variable in which to declare the concrete values:-

    targets:
        make-webserver:
            hosts: webservers
            variables:
                websites:
                    -
                      name: "one.example.com"
                      docroot: "/var/www/one.example.com"
                    -
                      name: "another.example.com"
                      docroot: "/var/www/another.example.com"
            tasks:
                -
                  ...

We add a Task to the `make-webserver` Target to write the vhost config files to
our web server hosts:-

            tasks:
                -
                  ...
                -
                  name: "Copy site config to sites-available"
                  command: "fs:copy"
                  sudo: true
                  arguments:
                      src: "@assets/vhost.conf.template"
                      dest: "/etc/apache2/sites-available/{{ item.name }}.conf"
                  with_items: websites

Here we use the `fs:copy` command to copy the content of our template to
`/etc/apache2/sites-available`.  The `@` character at the beginning of the
`src` argument indicates that Droid should substitute the file path with the
contents of that file and, because the file is a template, it should replace
placeholders in the content with concrete values.  The `with_items` directive
instructs Droid to perform a Task for each item in the list of items declared
at the named variable: our Target variable `websites` in this case.  Droid
makes available a special variable named `item` containing the values of the
current item and this is how Droid is able to replace the `{{ item.name }}` and
`{{ item.docroot }}` placeholders.

Next we will add tasks to create the document root directories and assign
ownership and permissions:-

            tasks:
                -
                  ...
                -
                  name: "Create the document root directory"
                  command: "fs:mkdir"
                  sudo: true
                  arguments:
                      directory: "{{ item.docroot }}"
                  with_items: websites
                -
                  name: "Change ownership of the document root directory"
                  command: "fs:chown"
                  sudo: true
                  arguments:
                      file: "{{ item.docroot }}"
                      user: "root"
                      group: "www-data"
                  with_items: websites
                -
                  name: "Change permissions of the document root directory"
                  command: "fs:chmod"
                  sudo: true
                  arguments:
                      filename: "{{ item.docroot }}"
                      mode: "2750"
                  with_items: websites

We deploy our HTML index page:-

            tasks:
                -
                  ...
                -
                  name: "Copy a placeholder index page to the document root directory"
                  command: "fs:copy"
                  sudo: true
                  arguments:
                      src: "@assets/index.html"
                      dest: "{{ item.docroot} }}/index.html"
                  with_items: websites

The `@` symbol at the beginning of the `src` argument instructs Droid to
substitute the file path with the content of that file without replacing
placeholders in the content.

Our final Task is to enable our Apache virtual hosts and trigger a
configuration reload:-

            tasks:
                -
                  ...
                -
                  name: "Enable the vhost configuration"
                  command: "apache:ensite"
                  sudo: true
                  arguments:
                      site-name: "{{ item.name }}"
                  with_items: websites
                  trigger: "Reload the Apache2 service"
            triggers:
                -
                  name: "Reload the Apache2 service"
                  command: "service:reload"
                  sudo: true
                  arguments:
                      name: "apache2"

Some Droid commands are able to report when they have made a change of some
kind.  Tasks which invoke such commands can declare a `trigger` which is a
special Task that will be run after all other Tasks have completed and only
when the triggering command reports that a change was made.  In this example
the `service:reload` command is executed only when a virtual host was not
already enabled.

Our project directory now looks like this:-

    assets/index.html
    assets/vhost.conf.template
    composer.json
    composer.lock
    droid.yml
    vendor/

We may now execute our Target exactly as we did before:-

    $ vendor/bin/droid make-webserver

## Further reading

- [Task Arguments and Variables][task-args]
- [Enable remote command execution][remote-exec]
- [Configuration reference][conf-index]
- [Modules - reusable Task lists][modules]

[Composer]: <https://getcomposer.org/>
[Yaml]: <http://www.yaml.org/spec/1.2/spec.html> "YAML Ain’t Markup Language (YAML&#8192;) Version 1.2"
[conf-index]: </configuration-reference/index.html> "Configuration reference"
[remote-exec]: </enable-remote-command-execution.html> "Enable remote command execution"
[task-args]: </task-arguments-and-variables.html> "Task Arguments and Variables"
[modules]: </modules.html> "Modules"
