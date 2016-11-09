# Command Plugins

Commands are made available for use with Droid through Command Plugins.  A
Plugin is a package of Commands under a common namespace.  For example, the
`droid-mysql` Plugin package provides a number of Commands in the `mysql`
namespace, including `mysql:dump` and `mysql:load`.

This document shows how to create a Command Plugin and a Command; how to make
the Plugin available to Droid during the development of a Command; and finally
how to publish the Plugin so that the Command is made available for remote
execution as part of a Droid project.

## Prepare a new Command Plugin

The [Plugin Generation Command][command-plugin-gen] may be used to kick-start
the development of a Command Plugin: it populates a specified folder to form
self-contained project.

We begin by creating a basic Droid Project which will make use of our Command
Plugin:-

```shell
$ cd /path/to/projects
$ mkdir myproject
$ cd myproject
$ composer init -n && composer require droid/droid
$ echo 'name: "A basic Droid Project"' > droid.yml
$ cd ..
```

We create a skeleton Command Plugin project which we shall give the name
"chuck-norris" - this will be the namespace of the Plugin's Commands:-

```shell
$ mkdir chuck-norris
$ myproject/vendor/bin/droid generate:plugin chuck-norris
Generating: chuck-norris in /path/to/projects/chuck-norris
$ cd chuck-norris && composer install
```

The `chuck-norris` folder now contains all that we need for developing Commands
for the Plugin.

The last of our preparatory steps is make the new Plugin available to our basic
Droid Project.  For this, we will use [Autotune][autotune] to give the Droid
project the ability to autoload our Plugin:-

```shell
$ cd /path/to/projects/myproject
$ touch autotune.json
```

We add the following content to the `autotune.json`:-

```json
{
  "autoload": {
    "psr-4": {
      "Droid\\Plugin\\ChuckNorris\\": "/path/to/projects/chuck-norris/src"
    }
  }
}
```

And our Droid project is now able to load the chuck-norris Plugin and list and
execute its Commands:-

```shell
$ vendor/bin/droid list chuck-norris
...
Available commands for the "chuck-norris" namespace:
  chuck-norris:example  This is an example command
```

The skeleton Plugin includes an example Command `chuck-norris:example` which we
can use as the starting point for further development.

## Create a Droid Command

We shall create a Droid Command the purpose of which shall be to set-up a
Message Of The Day (MOTD) on a Host.  It will:-

- fetch a random Chuck Norris joke from [The Internet Chuck Norris
  Database][icndb], using [its API][icndb-api].
- write the joke to `/etc/motd`, overwriting any previous MOTD.
- provide an optional argument to allow the the MOTD to be written to any
  alternative file.

A suitable name for our Command might be `jotd`.  By convention, the PHP class
name will be `ChuckNorrisJotdCommand`, that is, the Command namespace
`chuck-norris`, title case, followed by the Command name `jotd`, also title
case, followed by `Command`.

We make a copy of the example Command:-

```shell
$ cd /path/to/projects/chuck-norris
$ cp src/Command/ChuckNorrisExampleCommand.php src/Command/ChuckNorrisJotdCommand.php
```

Next, edit the `ChuckNorrisJotdCommand.php` file to look like the following:-

```php
<?php

// src/Command/ChuckNorrisJotdCommand.php

namespace Droid\Plugin\ChuckNorris\Command;

use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputArgument;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;

class ChuckNorrisJotdCommand extends Command
{
    public function configure()
    {
        $this
            ->setName('chuck-norris:jotd')
            ->setDescription('Install a Chuck Norris Joke of the Day')
            ->addArgument(
                'file',
                InputArgument::OPTIONAL,
                'Write the JotD to this file',
                '/etc/motd'
            )
        ;
    }

    public function execute(InputInterface $input, OutputInterface $output)
    {
        $raw = file_get_contents('http://api.icndb.com/jokes/random/');
        if ($raw === false) {
            $output->writeln(
                'Did not get a sensible response from api.icndb.com'
            );
            return;
        }

        $response = json_decode($raw, true);
        if ($response === null
            || ! array_key_exists('type', $response)
            || $response['type'] !== 'success'
            || ! array_key_exists('value', $response)
            || ! array_key_exists('joke', $response['value'])
        ) {
            $output->writeln(
                'Did not get a well-formed json response from api.icndb.com'
            );
            return;
        }

        $result = file_put_contents(
            $input->getArgument('file'),
            html_entity_decode($response['value']['joke']) . PHP_EOL
        );

        if ($result === false) {
            $output->writeln('Failed to write to the <file>.');
            return;
        }

        $output->writeln('Successfully wrote to the <file>.');
    }
}
```

We have:-

- changed the PHP class name: `ChuckNorrisJotdCommand`
- configured the Command, in the `configure` method, to set the name of the
  Command `chuck-norris:jotd` and provide a description
- configured the Command to accept an optional argument `file` which will take
  the default value `/etc/motd`
- populated the `execute` method with the code to be executed: it will make an
  API request and write a joke, if successfully obtained, to the file indicated
  by the `file` argument.

Our Command is now complete.  We must now register it with the Plugin.  The
`src/DroidPlugin.php` contains the main Plugin class `DroidProject`.  Its
method `getCommands` is called by the Droid Console Application which expects a
list of Command objects.  We modify the `getCommands` method to create an
instance of our Command and add it to the list of objects to be returned by the
method:-

```php
<?php

//src/DroidPlugin.php

namespace Droid\Plugin\ChuckNorris;

use Droid\Plugin\ChuckNorris\Command\ChuckNorrisExampleCommand;
use Droid\Plugin\ChuckNorris\Command\ChuckNorrisJotdCommand;

class DroidPlugin
{
    public function __construct($droid)
    {
        $this->droid = $droid;
    }

    public function getCommands()
    {
        $commands = [];
        $commands[] = new ChuckNorrisExampleCommand();
        $commands[] = new ChuckNorrisJotdCommand();
        return $commands;
    }
}
```

In our basic Droid project, Droid should now have two Commands in the
`chuck-norris` namespace:-

```shell
$ cd /path/to/myproject
$ vendor/bin/droid list chuck-norris
...
Available commands for the "chuck-norris" namespace:
  chuck-norris:example  This is an example command
  chuck-norris:jotd     Install a Chuck Norris Joke of the Day

```

Our Command Plugin is now complete and its Commands are available to our basic
Droid Project for execution the local host, either directly at the command line
or as part of a Task.

Next we shall publish the Command so that it may be executed on a remote Host.

## Publish a Command

A Plugin must be published somewhere, accessible to [Composer][composer], in
order that Droid may execute the Plugin's Commands on a remote Host.  Usually,
a Droid Project will make use of Command Plugins published on
[Packagist][packagist], but for this example we will publish our Plugin at
[GitHub][github] and Composer will be instructed to obtain it from GitHub by
way of a [custom repository][composer-repo].

The first thing to do is [create a GitHub repository][github-make-repo] for the
Plugin.  We shall pretend that the repository has been created and its URL is:-

`https://github.com/someone/chuck-norris.git`

Next, we initialise the local git repository:-

```shell
$ cd /path/to/projects/chuck-norris
$ git init
$ git config user.name someone
$ git config user.email someone@users.noreply.github.com
$ git remote add origin https://github.com/someone/chuck-norris.git
```

We remove the `chuck-norris:example` Command from the Plugin so that it
provides just the `chuck-norris:jotd` Command:-

```php
<?php

//src/DroidPlugin.php

namespace Droid\Plugin\ChuckNorris;

use Droid\Plugin\ChuckNorris\Command\ChuckNorrisJotdCommand;

class DroidPlugin
{
    public function __construct($droid)
    {
        $this->droid = $droid;
    }

    public function getCommands()
    {
        $commands = [];
        $commands[] = new ChuckNorrisJotdCommand();
        return $commands;
    }
}
```

```shell
$ rm src/Command/ChuckNorrisExampleCommand.php
```

Next, we provide some information about the Plugin's Command in its
`README.md`:-

```markdown
Droid plugin: chuck-norris
======================

For more information on Droid, please check out [droidphp.com](http://droidphp.com)

Provides the commands:-

- `chuck-norris:jotd`: Install a Chuck Norris Joke of the Day
```

And we commit and publish our work:-

```shell
$ git add .
$ git commit -m 'Add Droid plugin "chuck-norris".'
$ git push -u origin master
```

Now that the Plugin is published, we can update our Droid Project to fetch the
Plugin from GitHub.  We remove `autotune.json`:-

```shell
$ cd /path/to/projects/myproject
$ rm autotune.json
```

add the remote git repository to the Project's `composer.json`:-

```json
{
    "require": {
        "droid/droid": "^1.20",
        "droid/droid-chuck-norris": "dev-master as 1.0.x-dev"
    },
    "require-dev": {
        "linkorb/autotune": "^1.2"
    },
    "repositories": [
        {
            "type": "vcs",
            "url": "https://github.com/someone/chuck-norris.git"
        }
    ]
}
```

We can now fetch the Plugin from GitHub, into our Droid Project:-

```shell
$ composer update droid/droid-chuck-norris
$ vendor/bin/droid list chuck-norris
...
Available commands for the "chuck-norris" namespace:
  chuck-norris:jotd     Install a Chuck Norris Joke of the Day
```

Our Command is now available for use both locally and on remote Hosts.

Let's try it out.  We add a remote Host and a Target to our Droid's Project
`droid.yml`:-

```yaml
# /path/to/projects/myproject/droid.yml
project:
    name: "A basic Droid Project"

targets:
    install_jotd:
        hosts: "myhost"
        tasks:
            -
              name: "Install Joke of the Day"
              command: "chuck-norris:jotd"
              sudo: true
            -
              name: "Display the installed Joke of the Day"
              command: "shell:exec"
              sudo: true
              arguments:
                  command-line: "head /etc/motd"

hosts:
    myhost:
        droid_ip: "198.51.100.1"
```

Our Droid Project now requires the `droid/droid-shell` package because we have
used the `shell:exec` Command in the second Task:-

```shell
$ composer require droid/droid-shell
```

We may execute the `install_jotd` Target:-

```shell
$ vendor/bin/droid install_jotd
```

We should see something like the following:-

```shell
Droid: Running target `install_jotd`
Task `Install Joke of the Day`: chuck-norris:jotd on myhost
Host myhost: file=/etc/motd
myhost Begin droid enablement.
myhost Finished droid enablement. Success.
myhost Successfully wrote to the <file>.
Task `Display the installed Joke of the Day`: shell:exec on myhost
Host myhost: command-line=head /etc/motd
myhost I have successfully executed the given command line.
myhost Someone once tried to tell Chuck Norris that roundhouse kicks aren't the best way to kick someone. This has been recorded by historians as the worst mistake anyone has ever made.
Result: 0
--------------------------------------------
```

Our new Command Plugin was installed, directly from GitHb, to the remote Host
and the `chuck-norris:jotd` Command was successfully executed.

[autotune]: <https://github.com/linkorb/autotune> "AutoTune for Composer - For PHP Library Developers"
[command-plugin-gen]: </command-reference/generate__plugin.html> "Command `generate:plugin`"
[composer]: <https://getcomposer.org/> "Composer: Dependency Manager for PHP"
[composer-repo]: <https://getcomposer.org/doc/05-repositories.md#loading-a-package-from-a-vcs-repository> "Loading a package from a VCS repository"
[github]: <https://github.com> "GitHub"
[github-make-repo]: <https://help.github.com/articles/create-a-repo/> "Create A Repo"
[icndb]: <http://www.icndb.com/> "The Internet Chuck Norris Database"
[icndb-api]: <http://www.icndb.com/api/> "API | The Internet Chuck Norris Database"
[packagist]: <https://packagist.org/> "The PHP Package Repository"
