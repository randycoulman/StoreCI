# StoreCI

Integrate Cincom Visualworks Smalltalk with Continuous Integration
servers.

StoreCI is licensed under the MIT license.  See License.txt for
details.

StoreCI is hosted in the
[Cincom Public Store Repository](http://www.cincomsmalltalk.com/CincomSmalltalkWiki/Public+Store+Repository).
I've included a snapshot of the StoreCI parcel here, but you should
check for the latest version in the public repository.

StoreCI is intended to be used as a bridge between a Continuous
Integration (CI) server and Visualworks Store.  It has been tested
against Jenkins 1.466 and later using the visualworks-store-plugin
(https://wiki.jenkins-ci.org/display/JENKINS/Visualworks+Smalltalk+Store+Plugin).

StoreCI has been developed in VW 7.9.1, but should be compatible with
VW 7.7 and later.  It uses the newer Glorp-based Store implementation,
so will not work in 7.6 or earlier.  If you find any incompatibilities
with VW 7.7 or later, please file an issue.  I have not yet tested
this with the VW 7.10 development builds.

# Usage

The Jenkins visualworks-store-plugin expects to run StoreCI by
invoking a shell script (or batch file on Windows).  This script
should run Visualworks with an image that has StoreCI already loaded,
or with a base image and command-line options to load StoreCI from a
parcel.  The image also needs to contain Store repository definitions
for the repositories that you want to monitor.

The script should pass any command-line arguments that it receives on
to the image.  An example Unix/Linux/MacOS shell script might run
StoreCI this way:

    /path/to/VM /path/to/image "$@"

or:

    /path/to/VM /path/to/image -pcl /path/to/StoreCI.pcl "$@"

# Configuring the Jenkins Plugin

visualworks-store-plugin contains a global configuration section
(under the Manage Jenkins link) where you can define one or more
scripts as described above.  Simply specify the full path to the
desired script.  You might have multiple scripts if you want to run
builds for multiple versions of Visualworks in parallel.

When creating or editing a Job, you can choose Store as the SCM type.
Then:

* Choose one of the scripts you configured in the global settings.

* Specify the name of the Store repository to monitor.

* Choose one or more pundles to monitor.  StoreCI will watch for
  changes to those pundles and any of their (recursive) prerequisites
  in the same repository.  Thus, it is only necessary to specify
  "root" pundles here.

* Specify a regular expression that indicates what versions you would
  like to consider.  By default, all versions are considered.  The
  regular expression must be in Regex11 format.  For example,
  integer-only version numbers can be matched using "\d+".

* Specify a minimum blessing level (default: Development).  Pundles
  with a lower blessing level will not be considered.

* If desired, check the box to enable generation of an input file for
  ParcelBuilder and then specify a filename.  This file will contain
  the list of most-recent pundle versions that were found by StoreCI
  as of the time the build started.  This file can be used by
  ParcelBuilder (available in the Public Store Repository) or similar
  tool to build an image or deploy parcels as part of the build.  If
  no path is specified, the file will be created in the root of the
  Jenkins workspace directory for your project.  This file will
  contain the pundles in "load order".  That is, all of a pundle's
  prerequisites will be earlier in the file than itself.  That way,
  when loading the pundles into an image, no automatic prerequisite
  resolution will need to be done.

StoreCI is designed to work with the Multiple SCMs plugin.  This is
useful if you need to monitor more than one repository for the same
build, or if you need to monitor both a Store repository and a git or
Subversion repository.

# Command-line Usage

If you'd like to use StoreCI with a different CI system, you'd need to
configure that system to invoke StoreCI with the correct command-line
arguments.

StoreCI has two basic modes of operation:

1. Compute the current revision state.  This is what a CI system uses
to determine whether there are any source code changes that should
trigger a build.

2. Compute a changelog.  This is what a CI system uses to report the
details of any changes.

They both write the current revision state to `Stdout` to be picked up
by the CI system.  That allows some optimizations in the CI system,
and also provides for CI systems that only invoke a single operation -
they can use the changelog-computation operation.  See the
RevisionState class comment for details on the output format.

Both operations accept the following command-line arguments:

* (Required) `-repository <Name>`: The name of the Store repository to
  monitor.  The image containing StoreCI must have a repository
  defined with this name.

* (Required) `-package <Package name>` / `-packages <list of Package
  Names>` / `-bundle <Bundle name>` / `-bundles <list of Bundle names>`:
  The "root" pundles to monitor.  StoreCI monitors these pundles and
  their recursive prerequisites for changes.

* (Optional) `-blessedAtLeast <Blessing Level>`: The minimum blessing
  level that will be considered.  Pundles with a lower blessing level
  will be ignored.

* (Optional) `-versionRegex <Regex>`: A Regex11-style regular
  expression that can be used to specify which versions of pundles
  should be considered.

* (Optional) `-parcelBuilderFile <Filename>`: The name of a file that
  StoreCI will populate with the current versions of all of the
  pundles considered.  This file will contain the pundles in "load
  order".  That is, all of a pundle's prerequisites will be earlier in
  the file than itself.  That way, when loading the pundles into an
  image, no automatic prerequisite resolution will need to be done.
  See the LoadOrder class comment for details on the file format.

* (Optional) `-debug`: When specified, exceptions will not be trapped
  and the image will not exit upon completion.

The changelog computation process also takes:

* (Required) `-changelog <Filename>`: The name of the file to which
  the changelog should be written.  See the Changelog class comment
  for details on the file format.

* (Required) `-since <Timestamp>`: The time of the previous build.
  The timestamp must be formatted in the C-locale timestamp format
  compatible with Timestamp class>>readFrom:

* (Required) `-now <Timestamp>`: The time of the current build.  The
  timestamp must be formatted in the C-locale timestamp format
  compatible with Timestamp class>>readFrom:.  By specifying this
  timestamp, any changes that are published while the current build is
  running will be ignored and then picked up on the next build cycle.

Revision state is computed without making any changes to the local
filesystem.  Thus, it can be run from any working directory.

Changelog computation, however, maintains a cache of the last-seen
pundle versions in a .store subdirectory of the working directory.
This allows it to determine which packages are new, deleted, or just
modified.

# Understanding the Code

StoreCI is the main entry point for the command-line application.  It
processes command-line arguments, configures an instance of StoreSCM,
and then invokes the desired operation on it.  See the class and
method comments for more information.

# Contributing

I'm happy to receive bug fixes and improvements to this package.  If
you'd like to contribute, please publish your changes as a "branch"
(non-integer) version in the Public Store Repository and contact me to
let me know.  I will merge your changes back into the "trunk" as soon
as I can review them.

# Related Work

ParcelBuilder can be used to load the list of pundle versions created
by StoreCI and save them as parcels on disk.  This is ideal for a
"base image + parcels" deployment approach.

StoreCI could also work with
[CruiseControl](http://cruisecontrol.sourceforge.net/), but would
require some changes to the CruiseControl Store plugin to support
bundles and the new command-line API.  See my older CruiseControl
package in the Public Store Repository.  The CruiseControl package
supports the command-line API used by the CruiseControl Store plugin,
but only supports packages and not bundles at this time.
