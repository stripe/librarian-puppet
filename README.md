# !

Take note: this repo has been forked from
[rodjek/librarian-puppet](https://github.com/rodjek/librarian-puppet) for feature development by
Stripe's Systems team.  The branch `master` should track upstream `master`, and the branch `stripe` should be used as the stable changes from Stripe.

# Librarian-puppet

[![Build Status](https://travis-ci.org/rodjek/librarian-puppet.png?branch=master)](https://travis-ci.org/rodjek/librarian-puppet)

## Introduction

Librarian-puppet is a bundler for your puppet infrastructure.  You can use
librarian-puppet to manage the puppet modules your infrastructure depends on,
whether the modules come from the [Puppet Forge](https://forge.puppetlabs.com/),
Git repositories or just a path.

* Librarian-puppet can reuse the dependencies listed in your `Modulefile` or `metadata.json`
* Forge modules can be installed from [Puppetlabs Forge](https://forge.puppetlabs.com/) or an internal Forge such as [Pulp](http://www.pulpproject.org/)
* Git modules can be installed from a branch, tag or specific commit, optionally using a path inside the repository
* Modules can be installed from GitHub using tarballs, without needing Git installed
* Modules can be installed from a filesystem path
* Module dependencies are resolved transitively without needing to list all the modules explicitly


Librarian-puppet manages your `modules/` directory for you based on your
`Puppetfile`.  Your `Puppetfile` becomes the authoritative source for what
modules you require and at what version, tag or branch.

Once using Librarian-puppet you should not modify the contents of your `modules`
directory.  The individual modules' repos should be updated, tagged with a new
release and the version bumped in your Puppetfile.

It is based on [Librarian](https://github.com/applicationsonline/librarian), a
framework for writing bundlers, which are tools that resolve, fetch, install,
and isolate a project's dependencies.

## Versions

Librarian-puppet >= 2.0 (as well as 1.1, 1.2 and 1.3) requires Ruby 1.9 and uses the Puppet Forge API v3.
Versions < 2.0 work on Ruby 1.8.

See the [Changelog](Changelog.md) for more details.

## The Puppetfile

Every Puppet repository that uses Librarian-puppet may have a file named
`Puppetfile`, `metadata.json` or `Modulefile` in the root directory of that repository.
The full specification
for which modules your puppet infrastructure repository depends goes in here.

### Simple usage

If no Puppetfile is present, `librarian-puppet` will download all the dependencies
listed in your `metadata.json` or `Modulefile` from the Puppet Forge,
as if the Puppetfile contained

    forge "https://forgeapi.puppetlabs.com"

    metadata


### Example Puppetfile

    forge "https://forgeapi.puppetlabs.com"

    mod 'puppetlabs-razor'
    mod 'puppetlabs-ntp', "0.0.3"

    mod 'puppetlabs-apt',
      :git => "git://github.com/puppetlabs/puppetlabs-apt.git"

    mod 'puppetlabs-stdlib',
      :git => "git://github.com/puppetlabs/puppetlabs-stdlib.git"

    mod 'puppetlabs-apache', '0.6.0',
      :github_tarball => 'puppetlabs/puppetlabs-apache'

    mod 'acme-mymodule', :path => './some_folder'

    exclusion 'acme-bad_module'


### Recursive module dependency resolution

When fetching a module all dependencies specified in its
`Modulefile`, `metadata.json` and `Puppetfile` will be resolved and installed.

### Puppetfile Breakdown

    forge "https://forgeapi.puppetlabs.com"

This declares that we want to use the official Puppet Labs Forge as our default
source when pulling down modules.  If you run your own local forge, you may
want to change this.

    metadata

Download all the dependencies listed in your `metadata.json` or `Modulefile` from the Puppet Forge.

    mod 'puppetlabs-razor'

Pull in the latest version of the Puppet Labs Razor module from the default
source.

    mod 'puppetlabs-ntp', "0.0.3"

Pull in version 0.0.3 of the Puppet Labs NTP module from the default source.

    mod 'puppetlabs-apt',
      :git => "git://github.com/puppetlabs/puppetlabs-apt.git"

Our puppet infrastructure repository depends on the `apt` module from the
Puppet Labs GitHub repos and checks out the `master` branch.

    mod 'puppetlabs-apt',
      :git => "git://github.com/puppetlabs/puppetlabs-apt.git",
      :ref => '0.0.3'

Our puppet infrastructure repository depends on the `apt` module from the
Puppet Labs GitHub repos and checks out a tag of `0.0.3`.

    mod 'puppetlabs-apt',
      :git => "git://github.com/puppetlabs/puppetlabs-apt.git",
      :ref => 'feature/master/dans_refactor'

Our puppet infrastructure repository depends on the `apt` module from the
Puppet Labs GitHub repos and checks out the `dans_refactor` branch.

When using a Git source, we do not have to use a `:ref =>`.
If we do not, then librarian-puppet will assume we meant the `master` branch.

If we use a `:ref =>`, we can use anything that Git will recognize as a ref.
This includes any branch name, tag name, SHA, or SHA unique prefix. If we use a
branch, we can later ask Librarian-puppet to update the module by fetching the
most recent version of the module from that same branch.

The Git source also supports a `:path =>` option. If we use the path option,
Librarian-puppet will navigate down into the Git repository and only use the
specified subdirectory. Some people have the habit of having a single repository
with many modules in it. If we need a module from such a repository, we can
use the `:path =>` option here to help Librarian-puppet drill down and find the
module subdirectory.

    mod 'puppetlabs-apt',
      :git => "git://github.com/fake/puppet-modules.git",
      :path => "modules/apt"

Our puppet infrastructure repository depends on the `apt` module, which we have
stored as a directory under our `puppet-modules` git repos.

    mod 'puppetlabs-apache', '0.6.0',
      :github_tarball => 'puppetlabs/puppetlabs-apache'

Our puppet infrastructure repository depends on the `puppetlabs-apache` module,
to be downloaded from GitHub tarball.

    mod 'acme-mymodule', :path => './some_folder'

Our puppet infrastructure repository depends on the `acme-mymodule` module,
which is already in the filesystem.

    exclusion 'acme-bad_module'

Exclude the module `acme-bad_module` from resolution and installation.

## How to Use

Install librarian-puppet:

    $ gem install librarian-puppet

Prepare your puppet infrastructure repository:

    $ cd ~/path/to/puppet-inf-repos
    $ (git) rm -rf modules
    $ librarian-puppet init

Librarian-puppet takes over your `modules/` directory, and will always
reinstall (if missing) the modules listed the `Puppetfile.lock` into your
`modules/` directory, therefore you do not need your `modules/` directory to be
tracked in Git.

Librarian-puppet uses a `.tmp/` directory for tempfiles and caches. You should
not track this directory in Git.

Running `librarian-puppet init` will create a skeleton Puppetfile for you as
well as adding `tmp/` and `modules/` to your `.gitignore`.

    $ librarian-puppet install [--clean] [--verbose]

This command looks at each `mod` declaration and fetches the module from the
source specified.  This command writes the complete resolution into
`Puppetfile.lock` and then copies all of the fetched modules into your
`modules/` directory, overwriting whatever was there before.

Librarian-puppet support both v1 and v3 of the Puppet Forge API.
Specify a specific API version when installing modules:

    $ librarian-puppet install --use-v1-api # this is default; ignored for official Puppet Forge
    $ librarian-puppet install --no-use-v1-api # use the v3 API; default for official Puppet Forge

Please note that this does not apply for the official Puppet Forge, where v3 is used by default.

Get an overview of your `Puppetfile.lock` with:

    $ librarian-puppet show

Inspect the details of specific resolved dependencies with:

    $ librarian-puppet show NAME1 [NAME2, ...]

Find out which dependencies are outdated and may be updated:

    $ librarian-puppet outdated [--verbose]

Update the version of a dependency:

    $ librarian-puppet update apt [--verbose]
    $ git diff Puppetfile.lock
    $ git add Puppetfile.lock
    $ git commit -m "bumped the version of apt up to 0.0.4."

## Configuration

Configuration comes from three sources with the following highest-to-lowest
precedence:

* The local config (`./.librarian/puppet/config`)
* The environment
* The global config (`~/.librarian/puppet/config`)

You can inspect the final configuration with:

    $ librarian-puppet config

You can find out where a particular key is set with:

    $ librarian-puppet config KEY

You can set a key at the global level with:

    $ librarian-puppet config KEY VALUE --global

And remove it with:

    $ librarian-puppet config KEY --global --delete

You can set a key at the local level with:

    $ librarian-puppet config KEY VALUE --local

And remove it with:

    $ librarian-puppet config KEY --local --delete

You cannot set or delete environment-level config keys with the CLI.

Configuration set at either the global or local level will affect subsequent
invocations of `librarian-puppet`. Configurations set at the environment level are
not saved and will not affect subsequent invocations of `librarian-puppet`.

You can pass a config at the environment level by taking the original config key
and transforming it: replace hyphens (`-`) with underscores (`_`) and periods
(`.`) with doubled underscores (`__`), uppercase, and finally prefix with
`LIBRARIAN_PUPPET_`. For example, to pass a config in the environment for the key
`part-one.part-two`, set the environment variable
`LIBRARIAN_PUPPET_PART_ONE__PART_TWO`.

Configuration affects how various commands operate.

* The `path` config sets the directory to install to. If a relative
  path, it is relative to the directory containing the `Puppetfile`. The
  equivalent environment variable is `LIBRARIAN_PUPPET_PATH`.

* The `tmp` config sets the cache directory for librarian. If a relative
  path, it is relative to the directory containing the `Puppetfile`. The
  equivalent environment variable is `LIBRARIAN_PUPPET_TMP`.

Configuration can be set by passing specific options to other commands.

* The `path` config can be set at the local level by passing the `--path` option
  to the `install` command. It can be unset at the local level by passing the
  `--no-path` option to the `install` command. Note that if this is set at the
  environment or global level then, even if `--no-path` is given as an option,
  the environment or global config will be used.


## Rsync Option

The default convergence strategy between the cache and the module directory is
to execute an `rm -r` on the module directory and just `cp -r` from the cache.
This causes the module to be removed from the module path every time librarian
puppet updates, regardless of whether the content has changed. This can cause
some problems in environments with lots of change. The problem arises when the
module directory gets removed while Puppet is trying to read files inside it.
The `puppet master` process will lose its CWD and the catalog will fail to
compile. To avoid this, you can use `rsync` to implement a more conservative
convergence strategy. This will use `rsync` with the `-avz` and `--delete`
flags instead of a `rm -r` and `cp -r`. To use this feature, just set the
`rsync` configuration setting to `true`.

    $ librarian-puppet config rsync true --global

Alternatively, using an environment variable:

    LIBRARIAN_PUPPET_RSYNC='true'

Note that the directories will still be purged if you run librarian-puppet with
the --clean or --destructive flags.

## How to Contribute

 * Pull requests please.
 * Bonus points for feature branches.

## Reporting Issues

Bug reports to the github issue tracker please.
Please include:

 * Relevant `Puppetfile` and `Puppetfile.lock` files
 * Version of ruby, librarian-puppet, and puppet
 * What distro
 * Please run the `librarian-puppet` commands in verbose mode by using the
  `--verbose` flag, and include the verbose output in the bug report as well.


## License
Please see the [LICENSE](https://github.com/rodjek/librarian-puppet/blob/master/LICENSE)
file.
