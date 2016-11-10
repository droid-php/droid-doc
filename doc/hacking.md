# Hacking

A short guide to contributing to the Droid PHP project.

## Droid repositories

Droid comprises a number of packages.  Development takes place in a number of
git repositories, hosted at GitHub.

- [droid/droid][droid] - The main Droid application.
- [droid/droid-model][droid-model] - Classes modelling Targets, Tasks, Hosts,
  Groups, etc.
- [droid/droid-libcommand][droid-libcommand] - A small number of helpers for
  Droid Command Plugins.
- [droid/droid-\*][droid-all] - Droid Command Plugins.
- [droid/droid-module-\*][droid-modules] - Droid Modules.
- [droid/droid-standard][droid-standard] - A meta package providing a standard
  set of Droid Command Plugins.
- [droid/droid-doc][droid-doc] - Documentation.

## Report a bug

Please consider reporting problems and defects in any of the Droid packages.
First, identify the repository corresponding to the defective package and
search the "Issues" to see if the particular problem has already been reported.
If an an existing Issue already covers the problem, please add any further
information that will help with diagnosis or correction.

When an existing Issue cannot be found for a particular problem, please create
a new one and observe the following recommendations:-

- Provide a concise, descriptive Issue Title.
- Describe the behaviour observed, including any relevant output, and how it
  differs from what was expected.
- Provide information that someone else can use to reproduce the problematic
  behaviour; this can be a list of steps to take or better, a test case.
- Provide details about the version of PHP being used locally and, where
  applicable, on remote Hosts.
- Provide the output of `composer show droid\*` and `composer show symfony\*`.

A patch to fix a defect is welcome; please see the next section for guidance on
patch submissions.

## Submit a patch

Making a Pull Request (see below) to provide a fix for a bug is preferred, but
a patch is always welcome too.  Attach a patch to an existing Issue if one
already describes the problem addressed by the patch; otherwise, please create
a new Issue (see above) and attach the patch to it.

To create a patch:-

- Clone the relevant repository, using `git clone`, for example:-

  ```shell
  $ git clone https://github.com/droid-php/droid.git
  ```

- Install any dependencies, including those meant for use during development,
  using `composer install`, for example:-

  ```shell
  $ cd droid && composer install
  ```

- Make the necessary fix. Please follow the recommendations of the
  [PSR-2 code style guide][psr-2].

- Run the PHPUnit test suite to ensure that the change doesn't introduce a
  further bug:-

  ```shell
  $ cd droid && phpunit
  ```

- Create the patch, for example:

  ```shell
  $ git diff > fix-something.patch
  ```

  or, when the change involves adding or removing files:-

  ```shell
  $ git add <files> # and/or git rm <files>
  $ git diff --cached > fix-something.patch
  ```

- Attach the patch to an Issue comment which describes the changes introduced

## Submit a Pull Request

Submitting a Pull Request (PR) is the best way to contribute a bug fix or
feature.  To obtain the code and work on the fix or feature:-

- Head to GitHub and fork the relevant repository and clone the fork:-

  ```shell
  $ git clone git@github.com:myname/droid
  ```

- Track the relevant repository by adding a [git-remote][], for example, to
  refer to the repository as "upstream":-

  ```shell
  $ cd droid && git remote add upstream git@github.com:droid-php/droid
  ```

- Create a topic branch on which to work on the fix or feature:-

  ```shell
  $ git checkout -b fix_something
  ```

- Install any dependencies, including those meant for use during development,
  using `composer install`:-

  ```shell
  $ composer install
  ```

- Run the test suite:-

  ```shell
  $ phpunit
  ```

- Work on the fix or feature, adding unit tests and integration tests as
  appropriate.

- Run the test suite and resolve any differences from the previous test result.

- Commit the fix or feature.  Please try to keep the scope of each commit as
  focused as possible and provide a commit message that includes sufficient
  detail that someone else will understand the change introduced.

- Fetch and integrate any changes from the official repository before pushing
  changes back to the forked repository:-

  ```shell
  $ git checkout master
  $ git fetch upstream master
  $ git merge --ff-only upstream/master
  $ git checkout fix_something
  $ git rebase master
  ```

- Push to the forked repository:-

  ```shell
  $ git push origin/fix_something
  ```

- Head to GitHub and open a PR against the master branch of the official
  repository.

Happy Hacking!

[droid]: <https://github.com/droid-php/droid> "Droid"
[droid-model]: <https://github.com/droid-php/droid-model> "Droid Model"
[droid-libcommand]: <https://github.com/droid-php/droid-libcommand> "Droid libcommand"
[droid-all]: <https://github.com/droid-php> "All Droid repositories"
[droid-modules]: <https://github.com/droid-php?query=droid-module> "Droid Modules"
[droid-doc]: <https://github.com/droid-php/droid-doc> "Droid Documentation"
[droid-standard]: <https://github.com/droid-php/droid-standard> "Droid Standard"
[git-remote]: <https://git-scm.com/docs/git-remote> "Git - git-remote Documentation"
[psr-2]: <http://www.php-fig.org/psr/psr-2/> "PSR-2: Coding Style Guide"
