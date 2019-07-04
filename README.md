# README

PoC for SCA (PSD/2) based upon Hydra and OpenShift.

## Bootstrap Openshift 3.10 with ISTIO on OSX

Follow the [setup instructions](./minishift-osx-bootstrap.md) to run MiniShift 3.10 locally; use the profile below to start MiniShift:

~~~bash
#!/bin/bash
set -e

# Create a profile called "servicemesh"
minishift profile set servicemesh

# Give the profile 8GB of memory
minishift config set memory 8GB

# Give the profile 4 CPU
minishift config set cpus 4

# Adding container image caching to allow faster profile setup
minishift config set image-caching true

# Pinning OpenShift version to be 3.10.0
minishift config set openshift-version v3.10.0

# Set DNS (due to dnsmasq on OSX)
minishift config set network-nameserver 8.8.8.8

# Add a user with cluster-admin role
minishift addon enable admin-user

# Allows to run containers with uid 0 on Openshift
minishift addon enable anyuid

# Start the profile
minishift start
~~~

You can use the [Istio add-on](https://github.com/minishift/minishift-addons/tree/master/add-ons/istio) from minishift to install and deploy Istio on to your OpenShift cluster. The add-on uses [Kubernetes Operators](https://coreos.com/operators/) that were built as part of [Maistra](http://maistra.io/) project.

Run the following command to have the add-on installed:

~~~bash
#!/bin/bash
set -e

git clone \
   https://github.com/minishift/minishift-addons

minishift addon install ./minishift-addons/add-ons/istio
~~~

In the final 3 step of your install you can install [Istio](https://istio.io/) by just enabling and applying the add-on on to your minishift profile created above:

~~~bash
#!/bin/bash
set -e

minishift addon enable istio
minishift addon apply istio
~~~

You might need to wait for few minutes for all the Istio pods to be up and running.

You can watch the status of the pods via command `oc get pods -n istio-system -w` — as system:admin.
