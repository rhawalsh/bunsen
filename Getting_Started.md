# Getting Started with Bunsen

This guide is intended to help you get your first Bunsen environment up and running.

# Expectations

This project is simply an ansible playbook that has a certain set of requirements that need to be satisfied moving in.

## Initial requirements

Before you can start provisioning your environment, the following items must be satisfied:

1. An environment must consist of at least 3 different systems (physical or virtual).  It is recommended that you have at least 5 machines, but minimally 3 machines can work.  More information on what the different systems will be doing will be provided later.
    - Of those 3 machines, one must have at least 280GiB of additional storage in a separate block device (i.e. sdb)
2. Each machine must be able to communicate with the others in the environment both by IP address and by name (i.e. ossbunsen-resource [10.2.0.5] needs to be able to ssh to ossbunsen-farm-f36-1 [10.2.0.6])
3. The provisioning user must have an ssh-key generated and located under `~/.ssh`.
4. A `controller` system that has ansible installed on it.

You can find an example environment that utilizes vagrant and libvirt at [the VDO vagrant example repository](https://github.com/dm-vdo/vagrant)

## System Functions

A provisioned Bunsen Environment consists of three different types of machine.

- Infrastructure
    - This machine will provide necessary infrastructure services for the other systems to utilize.
        - Such as NFS, [RSVP](https://github.com/dm-vdo/permabit-rsvpd) (resource management and coordination), iSCSI backing storage (if applcable)
- Resource
    - General development and compiling will be done on this machine.
    - If there are multiple incompatible targets (i.e. Fedora, RHEL), then one of each possible target should be available to ensure that the compilations are done according to their environment.
- Farm
    - At least one of each applicable targets (i.e. Fedora, RHEL).
        - With Fedora, we typically utilize at least one of each supported version of Fedora in case there is a change applied to one and not the other.
    - Each farm must have an additional disk (i.e. sdb) with 280 GiB or more of storage to use as scratch space for testing.
- Controller
    - This system can be the hypervisor that is running the various VMs.  Or it could be an additional VM living next to the others.

# Deploying your first environment

Assuming that the dependencies above have been satisfied, you can now provision a Bunsen environment.  In order to deploy a bunsen environment, you need a few things first.

1. First, we need a copy of this repository placed on a `controller` somewhere.
    - This system is not part of the minimal environment specified above.
2. Next, we need to generate an inventory file for ansible to utilize when provisioning the hosts.

    An example inventory file
    ```
    ossbunsen-infra      ansible_user=vagrant ansible_python_interpreter=/usr/bin/python3 ansible_ssh_pass=vagrant
    ossbunsen-resource   ansible_user=vagrant ansible_python_interpreter=/usr/bin/python3 ansible_ssh_pass=vagrant
    ossbunsen-farm-f36-1 ansible_user=vagrant ansible_python_interpreter=/usr/bin/python3 ansible_ssh_pass=vagrant
    
    [infrastructure]
    ossbunsen-infra
    
    [resources]
    ossbunsen-resource
    
    [farms]
    ossbunsen-farm-f36-1
    
    [all:vars]
    deploying_account = admin
    target_account = admin
    ```

    The `admin` account should be replaced with the account that you intend to do your development with.

    NOTE: In place of `ansible_ssh_pass=vagrant`, you can use `ansible_ssh_private_key_file='/path/to/vagrantdir/.vagrant/machines/<name>/libvirt/private_key'` (of course if you're not using vagrant or you have another private key granted, then you should point at that instead)
        - `/path/to/vagrantdir` would be where you cloned [the VDO vagrant example repository](https://github.com/dm-vdo/vagrant)

    NOTE: You can also leave out `ansible_ssh_pass` and `ansible_ssh_private_key_file` and insted use `--ask-pass` when running the `ansible-playbook` commands below to be prompted for the passwords instead.

3. Now we can run the provisioning for the first time.  It is generally recommended to provision the roles serially to ensure that any dependencies across roles are satisfied without the chance of a timing issue being introduced.

    Example commands for each role.
    ```
    ansible-playbook -i inventory \
                     -l infrastructure \
                     provisioning/playbook.yml \
                     --extra-vars="install_bunsen_support=true" \
                     --ask-become-pass

    ansible-playbook -i inventory \
                     -l resources \
                     provisioning/playbook.yml

    ansible-playbook -i inventory \
                     -l farms \
                     provisioning/playbook.yml
    ```

    - Note the added options `--extra-vars="install_bunsen_support=true" --ask-become-pass` on the infrastructure role.
        - `--extra-vars="install_bunsen_support=true"` instructs the ansible playbook to install the necessary COPR repositories on the `controller` system and install the python dependencies (python3-resource-discovery, and python3-utility-mill).  It is only needed to be specified one time.
        - `--ask-become-pass` instructs the `ansible-playbook` command to prompr for the sudoers password on the `controller` to allow for you to install the packages.
        - If you would prefer NOT letting ansible provision your `controller`, please see the [Manually Installing Bunsen Support Libs](#installing-bunsen-support-libraries)
      
# Using the Bunsen environment

While the environment is yours and you can make it work as you wish, the most effective workflow involves doing general development on the `resource` machine and then doing the compilations and testing from that machine.

The home directory for the `target_account` specified above is mounted via NFS, so if you are doing work that requires multiple distributions, you don't have to unnecessarily duplicate things.

To perform general development and testing for VDO, please see the VDO Getting Started document (This does not exist yet, contents TBD)

# Installing Bunsen Support Libraries

There are two ways get the necessary `bunsen` support software.  The simplest
is to let `bunsen` install it for you.  This is accomplished by overriding the
default setting of the `install_bunsen_support` variable when invoking the
`bunsen` playbook by providing the following additional argument to 
`ansible-playbook`:
* `--extra-vars="install_bunsen_support=true"`  

You need only provide this additional argument the first time you
run the `bunsen` playbook on an `ansible` controller or if you want `bunsen` to
install available updates to the support software.

The second way to get the `bunsen` support software is to install it yourself.
For this you will want to execute the following commands:
* `dnf copr enable @dm-vdo/BunsenSupport`
* `dnf install python3-resource-discovery`

Regardless of the method of installation you choose the following software will
be installed to satisfy dependencies of `python3-resource-discovery`:
* bunsen support
  * `python3-utility-mill`
* system supplied (if not already installed)
  * `pyyaml` 