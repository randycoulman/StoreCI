# StoreCI

Integrate Cincom Visualworks Smalltalk with a Continuous Integration server.

StoreCI is licensed under the MIT license.  See the Copyright tab in
the RB, the 'notice' property of this package, or the `License.txt` file
on GitHub.

StoreCI's primary home is the
[Cincom Public Store Repository](http://www.cincomsmalltalk.com/CincomSmalltalkWiki/Public+Store+Repository).
Check there for the latest version.  It is also on
[GitHub](https://github.com/randycoulman/StoreCI).

StoreCI was developed in VW 7.9.1, but is compatible with VW 7.7 and
later.  It uses the newer Glorp-based Store implementation, so will
not work in 7.6 or earlier.  If you find any incompatibilities with VW
7.7 or later, let me know (see below for contact information) or file
an issue on GitHub.

# Introduction

StoreCI consists of the following components.  These components are
designed to work together, but can be used independently if desired.
See the individual package comments for more details about each
component.

* StoreCI-Polling is a bridge between a Continuous Integration (CI)
  server and Visualworks Store.  It has been tested against Jenkins
  1.466 and later using the
  [visualworks-store-plugin](https://wiki.jenkins-ci.org/display/JENKINS/Visualworks+Smalltalk+Store+Plugin).

* StoreCI-Building loads a set of pundles from a Store repository into
  an image, and optionally deploys them as parcels.  It is intended to
  be used as part of an automated build, perhaps driven from a
  Continuous Integration system such as Jenkins or CruiseControl.

* StoreCI-Testing runs a set of SUnitToo test cases and outputs the
  results in a JUnit-compatible XML format.  It is intended to be used
  as part of an automated build, perhaps driven from a Continuous
  Integration system such as Jenkins or CruiseControl.

A common workflow is to use StoreCI-Polling to watch a Store
repository for changes of interest, having it generate a "load-order"
file.  Then, StoreCI-Building reads the load-order file, loads the
parcels into an image, and then either saves the image or deploys
parcels.  Finally, StoreCI-Testing runs all of the SUnitToo tests in
the image or deployed parcels.

# StoreCI-Polling

StoreCI-Polling is a bridge between a Continuous Integration (CI)
server and Visualworks Store.  It has been tested against Jenkins
1.466 and later using the
[visualworks-store-plugin](https://wiki.jenkins-ci.org/display/JENKINS/Visualworks+Smalltalk+Store+Plugin).

## Usage

The Jenkins visualworks-store-plugin expects to run StoreCI-Polling by
invoking a shell script (or batch file on Windows).  This script
should run Visualworks with an image that has StoreCI-Polling already
loaded, or with a base image and command-line options to load
StoreCI-Polling from a parcel.  The image also needs to contain Store
repository definitions for the repositories that you want to monitor,
or you can use the `-repositoriesFrom` argument (see
`ImportRepositoriesSubsystem` in StoreCI-Support).

The script should pass any command-line arguments that it receives on
to the image.  An example Unix/Linux/MacOS shell script might run
StoreCI-Polling this way:

	/path/to/VM /path/to/image "$@"

or:

	/path/to/VM /path/to/image -pcl /path/to/StoreCI-Polling.pcl "$@"

## Configuring the Jenkins Plugin

visualworks-store-plugin contains a global configuration section
(under the Manage Jenkins link) where you can define one or more
scripts as described above.  Simply specify the full path to the
desired script.  You might have multiple scripts if you want to run
builds for multiple versions of Visualworks in parallel.

When creating or editing a Job, you can choose Store as the SCM type.
Then:

* Choose one of the scripts you configured in the global settings.

* Specify the name of the Store repository to monitor.

* Choose one or more pundles to monitor.  StoreCI-Polling will watch
  for changes to those pundles and any of their (recursive)
  prerequisites in the same repository.  Thus, it is only necessary to
  specify "root" pundles here.

* Specify a regular expression that indicates what versions you would
  like to consider.  By default, all versions are considered.  The
  regular expression must be in Regex11 format.  For example,
  integer-only version numbers can be matched using `\d+`.

* Specify a minimum blessing level (default: Development).  Pundles
  with a lower blessing level will not be considered.

* If desired, check the box to enable generation of an input file for
  ParcelBuilder (now called StoreCI-Building and then specify a
  filename.  This file will contain the list of most-recent pundle
  versions that were found by StoreCI-Polling as of the time the build
  started.  This file can be used by StoreCI-Building or similar tool
  to build an image or deploy parcels as part of the build.  If no
  path is specified, the file will be created in the root of the
  Jenkins workspace directory for your project.  This file will
  contain the pundles in "load order".  That is, all of a pundle's
  prerequisites will be earlier in the file than itself.  That way,
  when loading the pundles into an image, no automatic prerequisite
  resolution will need to be done.

The visualworks-store-plugin is designed to work with the Multiple
SCMs plugin.  This is useful if you need to monitor more than one
repository for the same build, or if you need to monitor both a Store
repository and a git or Subversion repository.

## Command-line Usage

If you'd like to use StoreCI-Polling with a different CI system, you'd
need to configure that system to invoke StoreCI-Polling with the
correct command-line arguments.

StoreCI-Polling has two basic modes of operation:

1. Compute the current revision state.  This is what a CI system uses
   to determine whether there are any source code changes that should
   trigger a build.

2. Compute a changelog.  This is what a CI system uses to report the
   details of any changes.

They both write the current revision state to Stdout to be picked up
by the CI system.  That allows some optimizations in the CI system,
and also provides for CI systems that only invoke a single operation -
they can use the changelog-computation operation.  See the
RevisionState class comment for details on the output format.

Both operations accept the following command-line arguments:

* (Optional) `-repositoriesFrom <Filename>`: The name of a file
  containing the necessary Store repository definitions.  See
  `ImportRepositoriesSubsystem` in StoreCI-Support for more
  information.

* (Required) `-repository <Name>`: The name of the Store repository to
  monitor.  The image containing StoreCI-Polling must have a
  repository defined with this name.

* (Required) `-package <Package name>` / `-packages <list of Package
  Names>` / `-bundle <Bundle name>` / `-bundles <list of Bundle
  names>`: The "root" pundles to monitor.  StoreCI-Polling monitors
  these pundles and their recursive prerequisites for changes.

* (Optional) `-blessedAtLeast <Blessing Level>`: The minimum blessing
  level that will be considered.  Pundles with a lower blessing level
  will be ignored.

* (Optional) `-versionRegex <Regex>`: A Regex11-style regular
  expression that can be used to specify which versions of pundles
  should be considered.

* (Optional) `-parcelBuilderFile <Filename>`: The name of a file that
  StoreCI-Polling will populate with the current versions of all of
  the pundles considered.  This file will contain the pundles in "load
  order".  That is, all of a pundle's prerequisites will be earlier in
  the file than itself.  That way, when loading the pundles into an
  image, no automatic prerequisite resolution will need to be done.
  See the `LoadOrder` class comment for details on the file format.

* (Optional) `-debug`: When specified, exceptions will not be trapped
  and the image will not exit upon completion.

The changelog computation process also takes:

* (Required) `-changelog <Filename>`: The name of the file to which
  the changelog should be written.  See the `Changelog` class comment
  for details on the file format.

* (Required) `-since <Timestamp>`: The time of the previous build.
  The timestamp must be formatted in the C-locale timestamp format
  compatible with `Timestamp class>>readFrom:`.

* (Required) `-now <Timestamp>`: The time of the current build.  The
  timestamp must be formatted in the C-locale timestamp format
  compatible with `Timestamp class>>readFrom:`.  By specifying this
  timestamp, any changes that are published while the current build is
  running will be ignored and then picked up on the next build cycle.

Revision state is computed without making any changes to the local
filesystem.  Thus, it can be run from any working directory.

Changelog computation, however, maintains a cache of the last-seen
pundle versions in a .store subdirectory of the working directory.
This allows it to determine which packages are new, deleted, or just
modified.

## Understanding the Code

`PollingSubsystem` is the main entry point for the command-line
application.  It processes command-line arguments, configures an
instance of `StoreSCM`, and then invokes the desired operation on it.
See the class and method comments for more information.

## Related Work

StoreCI-Polling could also work with CruiseControl, but would require
some changes to the CruiseControl Store plugin to support bundles and
the new command-line API.  See my older CruiseControl package in the
Public Store Repository.  The CruiseControl package supports the
command-line API used by the CruiseControl Store plugin, but only
supports packages and not bundles at this time.

# StoreCI-Building

StoreCI-Building loads a set of pundles from a Store repository into
an image, and optionally deploys them as parcels.  It is intended to
be used as part of an automated build, perhaps driven from a
Continuous Integration system such as Jenkins or CruiseControl.

When deploying parcels, StoreCI-Building also copies in any "stock"
parcels that are used by the loaded pundles.  Stock parcels are those
that are shipped with Visualworks.  This is done so that there is a
complete, standalone set of parcels available that will load cleanly
into a base runtime image.

## Usage

StoreCI-Building is intended to be run from a command-line or
automated build script.  It requires an image that has
StoreCI-Building already loaded, or with a base image and command-line
options to load StoreCI-Building from a parcel.  The image also needs
to contain any necessary Store database interface pundles and Store
repository definitions for the repositories that you want to monitor.
As an alternative, you can use the `-repositoriesFrom` argument (see
`ImportRepositoriesSubsystem` in StoreCI-Support).

Run the image as follows:

	/path/to/VM /path/to/image <command-line options>

or:

	/path/to/VM /path/to/image -pcl /path/to/StoreCI-Building <command-line options>

The allowable command line options are:

* (Optional) `-repositoriesFrom <Filename>`: The name of a file
  containing the necessary Store repository definitions.  See
  `ImportRepositoriesSubsystem` in `StoreCI-Support` for more
  information.

* (Required) `-repository <Name>`: The name of the Store repository to
  build from.  The image containing StoreCI-Building must have a
  repository defined with this name.

* (Required) `-loadPundlesIn <Filename>`: The name of a file
  containing the list of pundles to load.  See below.

* (Optional) `-writeParcelsTo <Directory>`: The name of the directory
  where parcels will be created.  If it does not exist, it will be
  created.

* (Optional) `-debug`: When specified, exceptions will not be trapped
  and the image will not exit upon completion.

The input file must be in the same format as that produced by the
`-parcelBuilderFile` option of StoreCI-Polling.  That is, each pundle
to be loaded should be on its own line in the file.  Each line has the
format:

	PundleClassName<tab>"Name in Double Quotes"<tab>PundleVersion

where `PundleClassName` is either `StorePackage` or `StoreBundle`.

## Understanding the Code

`BuildingSubsystem` is the main entry point for the command-line
application.  It processes command-line arguments, configures a
`PundleLoader` and optionally a `ParcelWriter`, and then invokes the
desired operations See the class and method comments for more
information.

# StoreCI-Testing

StoreCI-Testing runs a set of SUnitToo test cases and outputs the
results in a JUnit-compatible XML format.  It is intended to be used
as part of an automated build, perhaps driven from a Continuous
Integration system such as Jenkins or CruiseControl.

## Usage

StoreCI-Testing is intended to be run from a command-line or automated
build script.  It requires an image that has the tests to be run and
StoreCI-Testing already loaded, or with a base image and command-line
options to load all of the code from parcels.  A common way to
generate the parcels is by using StoreCI-Building to deploy them in an
earlier part of the build process.

Run the image as follows:

	/path/to/VM /path/to/image <command-line options>

or:

	/path/to/VM /path/to/image -pcl <Top-level parcel> -pcl StoreCI-Testing <command-line options>

The allowable command line options are:

* (Required) `-testResultsFile <Filename>`: The name of the file to
  which the JUnit-format XML results will be written.

* (Optional) `-traceFile <Filename>`: The name of a file to which
  trace information will be written while tests are running.  This is
  useful for debugging purposes if tests start hanging in the
  automated build, or if they randomly fail due to test ordering
  issues.  A line is added to this file when a test is started, and
  another line is added when it completes.

* (Optional) `-testPackages <list of Package names>` / `-testBundles
  <list of Bundle names>` / `-testClasses <list of Class names>`: By
  default, StoreCI-Testing runs all of the tests in the image.  These
  options cause StoreCI-Testing to restrict itself to only those tests
  in the specified packages or bundles, or only the test classes
  explicitly listed.

* (Optional) `-debug`: When specified, exceptions will not be trapped
  and the image will not exit upon completion.

## Understanding the Code

`TestingSubsystem` is the main entry point for the command-line
application.  It processes command-line arguments, asks `SuiteBuilder`
to build a test suite to be run by `SuiteRunner`.  Results are
captured into `SuiteResults` and are wriitten to the output file by
`XMLResultWriter`.  See the class and method comments for more
information.

# Contributing

I'm happy to receive bug fixes and improvements to this package.  If
you'd like to contribute, please publish your changes as a "branch"
(non-integer) version in the Public Store Repository and contact me as
outlined below to let me know.  I will merge your changes back into
the "trunk" as soon as I can review them.

# Contact Information

If you have any questions about StoreCI and how to use it, feel
free to contact me.

* Web site: http://randycoulman.com
* Blog: Courageous Software (http://randycoulman.com/blog)
* E-mail: randy _at_ randycoulman _dot_ com
* Twitter: @randycoulman
* GitHub: randycoulman
