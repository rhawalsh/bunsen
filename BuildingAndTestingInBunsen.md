# Building and Testing the VDO software in a Bunsen Environment

This guide is intended to help get you started with developing, building, and testing your contributions to VDO in a Bunsen environment.

# Prerequisites

It is expected that you have satisfied the following criteria before getting started:

* [Set up a Bunsen Environment](Getting_Started.md)

# Expectations of this Guide

This guide is currently intended to provide enough information to get minimally functional.  A more detailed guide will be generated later, but in the meantime this document should help you get started if you're interested in contributing to the VDO projects.

# Find the appropriate code repository

Identify which repository you want to test.

Currently the following VDO repositories are set up to allow for local testing:
* [vdo-devel](https://github.com/dm-vdo/vdo-devel) - This is where the VDO product code resides as well as any supporting test code that is only needed for VDO itself.
* [common](https://github.com/dm-vdo/common) - This is where the library code resides that supports both the test environment as well as VDO.

* How do I tell what's the difference between the two?
  * The first thing to figure out is whether you're modifying the VDO code itself or the tests that directly test VDO.
    * If that's the case, then you want to use the [vdo-devel](https://github.com/dm-vdo/vdo-devel) tree.
  * Otherwise, you'll need to look for the code in either repository.  If you're not sure, then you should likely start on the [vdo-devel](https://github.com/dm-vdo/vdo-devel) repository and move to the [common](https://github.com/dm-vdo/common) repository if you still couldn't find what you were looking for.

# Find the code

For the purposes of this guide, all of the work you will be doing is going to take place on the Bunsen `resource` system.

SSH to the `resource` system (typically named `ossbunsen-resource`) as the user that you have provisioned it with.


For changes to the [vdo-devel](https://github.com/dm-vdo/vdo-devel) repository:

  * Starting in the home directory, clone the repository and cd into it
    ```
    git clone https://github.com/dm-vdo/vdo-devel.git && cd vdo-devel
    ```
  * The files under src from here can be generally distinguished by language-type (though `c++` isn't accurate).
    * Core VDO code can be found under src/c++/vdo/
    * Perl test library code can be found under src/perl/Permabit/
    * Core perl test code can be found under src/perl/vdotest/

For changes to the [common](https://github.com/dm-vdo/common) repository:

  * Starting in the home directory, clone the repository and cd into it
    ```
    git clone https://github.com/dm-vdo/common.git && cd common
    ```
  * This layout is similar to vdo-devel, except there is no src directory at the top level.
    * Perl code which is used for testing and other support functions can be found under perl/
    * Perl Test library code can be found under perl/Permabit/
      * Test definitions can be found under testcases/
    * Packaging code to generate the various RPMs needed for Bunsen can be found under packaging/

# Editing

Once you've cloned your tree to the resource machine, then you can go in and start making changes.

  * If you are intending to post these changes to another git repository (e.g., your fork of the repository on GitHub), you will want to add the relevant repositories as a remote in the repository.
  ```
  git remote add <name> <url>
  ```
  NOTE: Additional `git remote` operations can be studied through the usage of `git remote -h`.

# Building

The best way to build is to get into the base level of the repository and run `make`.

Assuming the repository was cloned to your homedir:

  * For [vdo-devel](https://github.com/dm-vdo/vdo-devel)

    ```
    cd ~/vdo-devel
    make
    ```

  * For [common](https://github.com/dm-vdo/common)

    ```
    cd ~/common
    make
    ```

# Automated Testing

## make jenkins

As part of the checkin process, VDO's projects always undergo testing via Jenkins.  This testing is initiated by calling the `make jenkins` target.

The `make jenkins` target is implemented in both the [common](https://github.com/dm-vdo/common) and [vdo-devel](https://github.com/dm-vdo/vdo-devel) projects.

IMPORTANT: All contributions made to these projects will run through the `make jenkins` target, so it is a good idea to run this before submitting a pull request to avoid any surprises.  It is generally the contributor's responsibility to ensure that all tests pass.

## make checkin

A quicker, but less complete, alternative to `make jenkins` in the repositories is the `make checkin` target which will run a less extensive suite of tests set of tests against UDS and VDO.

# Testing with Perl

## Running specific perl tests

If you want to run specific tests, then you should find the relevant directory to start:
* vdo-devel:
  * VDO tests - src/perl/vdotest/
  * UDS tests - src/c++/uds/src/tests
    * These tests are built and run in a kernel module via the perl test infrastructure.
    * Test files can be identified by their letter and number suffixes following the underscore (e.g. _t1, _p1, _x1, etc.).
* common - perl/Permabit/

From these locations, you will either find a script named `vdotests.pl` or `runtests.pl` (they are symlinks that point to the same base script) which will be used to start a test.

  * If you want to run a specific UDS test against the vdo-devel project, such as our KernelCheckin test:
  ```
  cd ~/vdo-devel/src/perl/udstest
  ./udstests.pl UDSTest::KernelCheckin
  ```

  * If you want to run a specific VDO test against the vdo-devel project, such as our Basic01 test:
  ```
  cd ~/vdo-devel/src/perl/vdotest
  ./vdotests.pl VDOTest::Basic01
  ```

  * If you want to run a specific test against the common project, such as our Assertions_t1 test:
  ```
  cd ~/common/perl/Permabit
  ./runtests.pl testcases::Assertions_t1
  ```

## Finding the results for the perl tests

By default, running `runtests.pl` or `vdotests.pl` with no additional options will log output to stdout.

If you want the results to go to a logfile, add the `--log` option to the command.  When running via `make jenkins` the logfiles are relocated to the base of the repository under the logfiles directory (e.g. ~/vdo-devel/logfiles or ~/common/logfiles)

# C Unit tests

## Locating the C unit tests

C Unit tests can only be found on the [vdo-devel](https://github.com/dm-vdo/vdo-devel) repository.

* VDO tests - src/c++/vdo/tests
  * These tests are built and run in userspace on the builder system.
  * Test files can be identified by their letter and number suffixes following the underscore (e.g. _t1, _p1, x1, etc.).

## Running the C unit tests for VDO

The VDO unit tests are typically run any time `make checkin` or `make jenkins` is run and it is done locally on the builder machine (in a bunsen environment, the `resource` system)

* Here is an example that runs the C unit tests for VDO
  ```
  cd ~/vdo-devel/src/c++/vdo/tests
  make checkin
  ```

  NOTE: We're using `make checkin` to launch the tests.  This is because the tests are typically always run all at once via a compiled program named `vdotest` in the same directory.
