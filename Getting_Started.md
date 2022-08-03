# Getting Started with Bunsen

This guide is intended to help get your Bunsen environment up and running.

# Expectations

This project is an ansible playbook with specific requirements that need to be satisfied before it can be used to provision an environment.

## Initial requirements

Before you can start provisioning your environment, the following criteria must be satisfied:

1. At least 3 individual systems, physical and/or virtual.
    - It is recommended to have at least 5 systems.
    - The function of the separate systems are described below.
    - At least one of the 3 systems must have a minimum of 280 GiB of additional storage in a separate block device.
        - Generally this is /dev/sdb.
        - If the system is virtualized using 'kvm', it can be /dev/vdb.
2. Each system must be able to communicate with the others in the environment both by IP address and by name (e.g. ossbunsen-resource [10.2.0.5] needs to be able to ssh to ossbunsen-farm-f36-1 [10.2.0.6])
3. The provisioning user must have an ssh-key generated and located under `~/.ssh`.
4. A `controller` system that has ansible installed on it.

You can find an example environment that utilizes vagrant and libvirt at [the VDO vagrant example repository](https://github.com/dm-vdo/vagrant)

## System Functions

A provisioned Bunsen Environment consists of three different classes of system.

- Infrastructure
    - This system provides necessary services for the other systems to utilize.
        - Such as NFS, [RSVP](https://github.com/dm-vdo/permabit-rsvpd) (resource management and coordination), iSCSI backing storage (if applcable)
- Resource
    - General development and compiling will be done on these systems.
    - If there are multiple incompatible targets (e.g. Fedora and RHEL), then one of each possible target should be available to ensure that the compilations are done according to their environment.
- Farm
    - At least one for each applicable targets (e.g. Fedora or RHEL).
        - With Fedora, we typically utilize at least one of each supported version of Fedora in case there is a change applied to one and not the other.
    - Each farm must have an additional disk (see [Initial Requirements](#initial-requirements) for size and naming requirements) to use as scratch space for testing.
- Controller
    - This system can be the hypervisor that is running the various VMs.  Or it could be an additional VM living next to the others.

# Deploying your first environment

Assuming that the dependencies above have been satisfied, you can now provision a Bunsen environment.  In order to deploy a Bunsen environment, you need the following:

1. A copy of this repository local to a `controller` system.
    - This system is not part of the minimal environment specified above.
2. An inventory file for ansible to utilize when provisioning the hosts.

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

    NOTE: If using vagrant, you can replace the use of `ansible_ssh_pass=vagrant` with `ansible_ssh_private_key_file='/path/to/vagrantdir/.vagrant/machines/<name>/libvirt/private_key'`. If not using vagrant, or wanting to use a different private key, `ansible_ssh_private_key` should point to the appropriate private key.
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
        - `--extra-vars="install_bunsen_support=true"` instructs the ansible playbook to install the necessary COPR repositories on the `controller` system and install the python dependencies (python3-resource-discovery, and python3-utility-mill).  It is required the first time you run the Bunsen playbook.  After that, you may exclude using it unless you want to be certain to be using the latest version available; i.e., with it specified the most up-to-date version of the python dependencies will be installed.
        - `--ask-become-pass` instructs the `ansible-playbook` command to prompt for the sudoers password on the `controller` to allow for you to install the packages.
        - If you would prefer NOT letting ansible provision your `controller`, please see the [Manually Installing Bunsen Support Libs](#installing-bunsen-support-libraries)
        - If you generally keep your system up to date, updates may come through the normal update process, which is separate from provisioning your Bunsen environment.
      
# Using the Bunsen environment

While the environment is yours and you can make it work as you wish, the most effective workflow involves doing general development on the `resource` machine and then doing the compilations and testing from that machine.

The home directory for the `target_account` specified above is mounted via NFS, so if you are doing work that requires multiple distributions, you don't have to unnecessarily duplicate things.

To perform general development and testing for VDO, please see the VDO Getting Started document (This does not exist yet, contents TBD)

# Installing Bunsen Support Libraries

There are two ways get the necessary Bunsen support software.  The simplest
is to let Bunsen install it for you.  This is accomplished by overriding the
default setting of the `install_bunsen_support` variable when invoking the
Bunsen playbook by providing the following additional argument to
`ansible-playbook`:
* `--extra-vars="install_bunsen_support=true"`  

You need only provide this additional argument the first time you
run the Bunsen playbook on an `ansible` controller or if you want Bunsen to
install available updates to the support software.

The second way to get the Bunsen support software is to install it yourself.
For this you will want to execute the following commands:
* `dnf copr enable @dm-vdo/BunsenSupport`
* `dnf install python3-resource-discovery`

Regardless of the method of installation you choose the following software will
be installed to satisfy dependencies of `python3-resource-discovery`:
* Bunsen support
  * `python3-utility-mill`
* system supplied (if not already installed)
  * `pyyaml` 