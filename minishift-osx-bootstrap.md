# OpenShift (minishift) on OSX

The following section describes how to [install Minishift](https://docs.openshift.org/latest/minishift/getting-started/installing.html) and the required dependencies.

Overview steps:

* Install hypervisor and Docker driver
* Install Minishift CLI
* Install OpenShift CLI
* Start MiniShift (optionally to test)

## Install hypervisor and Docker driver

The xhyve hypervisor is a port of bhyve to OS X. It is built on top of Hypervisor.framework in OS X 10.10 Yosemite and higher, runs entirely in userspace, and has no other dependencies. It can run FreeBSD and vanilla Linux distributions and may gain support for other guest operating systems in the future.

~~~bash
brew update
brew install --HEAD xhyve
~~~

### Setup the xhyve driver

Minishift uses Docker Machine and its driver plug-in architecture to provide a consistent way to manage the OpenShift VM. Minishift embeds VirtualBox and VMware Fusion drivers so no additional steps are required to use them.

We are using the xhyve driver as documented [here](https://docs.openshift.org/latest/minishift/getting-started/setting-up-driver-plugin.html#xhyve-driver-install). You can verify the existing version of the xhyve driver on your system using:

~~~bash
brew info docker-machine-driver-xhyve

docker-machine-driver-xhyve: stable 0.3.1 (bottled), HEAD
Docker Machine driver for xhyve
https://github.com/zchee/docker-machine-driver-xhyve
/usr/local/Cellar/docker-machine-driver-xhyve/0.3.1 (3 files, 13.2M) *
  Poured from bottle on 2016-12-20 at 17:44:35
From: https://github.com/Homebrew/homebrew-core/blob/master/Formula/docker-machine-driver-xhyve.rb
~~~

To install the stable version (0.3.3_1) of the driver with brew:

~~~bash
brew install https://raw.githubusercontent.com/Homebrew/homebrew-core/7310c563d662ddbe094f46f9600cad30ad3551a6/Formula/docker-machine-driver-xhyve.rb

# docker-machine-driver-xhyve need root owner and uid
sudo chown root:wheel $(brew --prefix)/opt/docker-machine-driver-xhyve/bin/docker-machine-driver-xhyve
sudo chmod u+s $(brew --prefix)/opt/docker-machine-driver-xhyve/bin/docker-machine-driver-xhyve
~~~

## Install Minishift CLI

Single command does the magix.

~~~bash
brew cask install minishift
~~~

## Install OpenShift CLI

It basically drills down to a single command, but the main [instructions and getting started](https://docs.openshift.org/latest/cli_reference/get_started_cli.html#cli-mac) might give you better background.

~~~bash
brew install openshift-cli
~~~

## Start MiniShift (test installation)

Run the minishift start command.

~~~bash
minishift start
-- Installing default add-ons ... OK
-- Checking if xhyve driver is installed ...
   Driver is available at /usr/local/bin/docker-machine-driver-xhyve
   Checking for setuid bit ... OK
-- Starting local OpenShift cluster using 'xhyve' hypervisor ...
-- Minishift VM will be configured with ...
   Memory:    2 GB
   vCPUs :    2
   Disk size: 20 GB
-- Starting Minishift VM ...

 (...)

OpenShift server started.

The server is accessible via web console at:
    https://192.168.64.2:8443

You are logged in as:
    User:     developer
    Password: <any value>

To login as administrator:
    oc login -u system:admin
~~~

> **Note**: if the VM fails to start due to error:
> ~~~text
> -- Checking HTTP connectivity from the VM ...
>  Retrieving http://minishift.io/index.html ... FAIL
> ~~~
> It is likely the `dnsmasq` deamon on OSX is interfering with name resolving... just stop the deamon before starting the VM, you can safely start the deamon afterwards

## Tear down MiniShift

Stop MiniShift and remove the VM.

~~~bash
# Stop
minishift stop

# Delete VM to free up diskspace
minishift delete
~~~