# Make Network Observability Available on Day 0

## Introduction

This document shows you how to test this feature proposal.  It involves three OpenShift repositories, namely openshift/api, openshift/installer, and openshift/cluster-network-operator.
- [api](https://github.com/stleerh/openshift-api)
- [installer](https://github.com/stleerh/openshift-installer)
- [cluster-network-operator](https://github.com/stleerh/openshift-cluster-network-operator)

I prefix'ed the repository names with "openshift-" in my forked repos to show they are related.  The code changes are in the "cno-observability" branch of each repo.

## Proposal

This change makes Network Observability available on day 0 by default.  That is, Network Observability is up and running after you create an OpenShift cluster using `openshift-install`.  There is an option to turn this off.  By default, it installs Network Observability Operator and creates a basic FlowCollector instance.

## Background

Network Observability is an optional operator that provides insights into your network traffic, including troubleshooting features like packet drops, latencies, DNS tracking, and more.  There is an upstream and downstream version, but you get more features and benefits when used in OpenShift, and specifically OpenShift Networking.  If you are running OpenShift Networking, it should be expected to have observability, unless it falls into an exceptional case (e.g. very limited resources).  The two should be bound together.

## How Does It Work

`openshift-install` uses an install-config.yaml file to create manifests that are used to create an OpenShift cluster.  One of the manifest files is cluster-network-02-config.yml.  This is the configuration file, based on the Network CRD, that Cluster Network Operator (CNO) reads to bootstrap networking.  A new field, observabilityEnabled, when set to true, tells CNO to enable Network Observability.

```
spec:
  observabilityEnabled: true
```

If this field is false or does not exist, Network Observability will not be enabled.  See below on how this is enabled by default.

The steps to enable Network Observability are as follows.  It is done only once at startup.

1. Check if Network Observability Operator (NOO) is installed.  If yes, exit.
2. Create "openshift-netobserv-operator" namespace if it doesn't exist.
3. Install NOO using OLM's OperatorGroup and Subscription.
4. Wait for NOO to be ready and OpenShift web console to be available.
5. Create "netobserv" namespace if it doesn't exist.
6. Check if a FlowCollector instance exists.  If yes, exit.
7. Create a FlowCollector instance.

It is important to note that CNO does not manage the lifecycle of Network Observability.  It doesn't upgrade Network Observability or monitor it.  It simply *enables* it, and after that, it is on its own to be managed by its operator, NOO.  In fact, it doesn't even remove it, hence the field is called **observabilityEnabled** to indicate that it will just enable it and do nothing if this is set to false.

To make Network Observability enabled by default, the same field is introduced in install-config.yaml.  However, the behavior is slightly different.  If the **observabilityEnabled** field is not there, it is *enabled* by default, unlike the field in the Network CRD.  Therefore, you normally don't need to do anything, but if you don't want Network Observability, add this to install-config.yaml:

```
networking:
  observabilityEnabled: false
```

This setting or the lack of it will determine the setting in the Network CRD.


## Code Changes

To view the code diffs,
- [api](https://github.com/stleerh/openshift-api/commit/ffae1de2ce1feb73bde651540e0c3a276b62e8a7)
- [install](https://github.com/stleerh/openshift-installer/commit/1a42ecd4a4f60caf218d59f4482b2ab9c1796c6c)
- [cluster-network-operator](https://github.com/stleerh/openshift-cluster-network-operator/commit/387f905ff8bae0bc2c1d6de12adb2251d179536d)


## Test on Existing Cluster

Use this test setup if you simply want to run the code against your existing OpenShift cluster and don't want to build the binaries.  Because the changes have not been committed to the official OpenShift repositories, there are additional commands you need to issue, such as to stop Cluster Version Operator (CVO) from resetting your changes.

### Prerequisites

- Have an existing OpenShift 4.19+ cluster with cluster-admin access
- Access to `oc` and logged into your cluster

### Commands

Issue these commands to set up your cluster.  The file, AAA_ungated.yaml, is in this repository.  You can also download it [here](https://raw.githubusercontent.com/stleerh/cno-observability/refs/heads/main/manifests/AAA_ungated.yaml).

```
# Disable CVO from resetting the Network CR
oc scale deployment/cluster-version-operator -n openshift-cluster-version --replicas=0

# Use the private CNO image on quay.io
oc set image deployment/network-operator -n openshift-network-operator network-operator=quay.io/stlee/cluster-network-operator:dev

# Inform API server of the updated Network CRD
# Get the yaml file in this repo or at https://github.com/stleerh/cno-observability/blob/main/manifests/AAA_ungated.yaml
oc replace -f manifests/AAA_ungated.yaml
```

### Test cases

In OpenShift web console, go to **Workloads > Pods**.  Select project **openshift-network-operator**.

For each test case, enter `oc edit network` to edit the Network CR.  Make the change as indicated in the step.  In OpenShift web console, right click &vellip; for the **network-operator-*** pod, and select **Delete Pod**.  This removes the pod, but then it restarts.  You can optionally check the logs in the **Logs** tab and search for "observability_controller" to track what's going on.

Note: Due to a [console reload](https://issues.redhat.com/browse/OCPBUGS-58468) issue, you might get logged out when Network Observability is installed.

<br>

1. Test `observabilityEnabled: true`
```
spec:
  observabilityEnabled: true
```

**Expect:** Network Observability is installed.  FlowCollector instance is created.

2. Test `observabilityEnabled: false`
```
spec:
  observabilityEnabled: false
```

**Expect:** Nothing happens.  Network Observability is still enabled if you installed it.

3. Test with no `observabilityEnabled` statement.

Remove `observabilityEnabled` line.  Remove FlowCollector.

**Expect:** FlowCollector is created again.


## Test: Build the Binaries

Use this test setup if you want to build your own binaries.  You can then follow the instructions above to test on an existing cluster but using your binaries.

### Test cases for openshift-install

There is no simple way to have a private openshift-install binary use a private CNO to create an OpenShift Cluster.  Therefore, the tests for your openshift-install binary will simply show that it accepts the new field in install-config.yaml and creates the correct manifest files.

#### Get source code

```
mkdir cno
cd cno
git clone https://github.com/stleerh/openshift-api.git api
git clone https://github.com/stleerh/openshift-installer.git installer
```

Note: This feature is on the **cno-observability** branch for all the repositories.

#### Build openshift/api

The latest openshift/install repo is based on an older version of openshift/api.  There appears to be some work to upgrade it to use the v0.33 Kubernetes libraries, so it hasn't been done yet.  Therefore, we will cherrypick the change back to the older version of openshift/api.  In your **api** working directory, enter:
```
cd api
git checkout 6a7223edb2fc  # older commit version
git switch -c api-cno
git cherry-pick ffae1de2c

# No changes to protobuf files; only Go files.  Therefore, it doesn't need to compile protobuf files.
PROTO_OPTIONAL=1 make update
```

#### Build openshift/installer

Now, go to the **installer** working directory.  Enter:
```
cd installer
git checkout cno-observability

# Use our 'api' directory
echo "replace github.com/openshift/api => ../api" >> go.mod

go mod tidy
go mod vendor
hack/build.sh
```

This creates the binary at **bin/openshift-install**.


#### Test openshift-install binary

You can use the sample [**install-config.yaml**](https://raw.githubusercontent.com/stleerh/cno-observability/refs/heads/main/manifests/install-config.yaml) file in the manifests/ directory to test your openshift-install binary.
```
mkdir cluster
cd cluster
cp ../manifests/install-config.yaml .
```

In the install-config.yaml, set the following to true, false, a bad value, or remove the statement.
```
networking:
  observabilityEnabled: true
```

Run `../installer/bin/openshift-install create install-config`.  View the modified install-config.yaml file.  The true and false cases should remain unchanged.  If the statement is not there, it sets it to true.  If it's a bad value, it rejects it.  Be sure to copy the install-config.yaml file after each test as it modifies the file.

Run `../installer/bin/openshift-install create manifests`.  View the manifests/cluster-network-02-config.yml.  It should set the `observabilityEnabled` field as expected.


#### Build openshift/cluster-network-operator

Unlike openshift/installer, openshift/cluster-network-operator can and should use the latest branch of openshift/api.  It's probably the easiest to create another clone of api.

```
git clone https://github.com/stleerh/openshift-api.git api2
cd api2
git checkout cno-observability
PROTO_OPTIONAL=1 make update

cd ../cluster-network-operator
git checkout cno-observability

echo "replace github.com/openshift/api => ../api2" >> go.mod

# Use the latest openshift/client-go instead of a commit version.
go get github.com/openshift/client-go@master

go mod tidy
go mod vendor
make
```

This creates three binaries starting with **cluster-network** at the root level.

#### Build Docker image and Push to quay.io

The Dockerfile refers to an image at **registry.ci.openshift.org**, which requires authentication.  To work around this, change the **FROM** line to refer to **registry.access.redhat.com/ubi9/ubi:latest** instead.

Log into podman with `podman login`.  Then enter the following:
```
# This assumes your PC username is the same as your quay.io username.
export IMAGE_TAG="quay.io/$(whoami)/cluster-network-operator:dev"

podman build -f Dockerfile.sl -t $IMAGE_TAG .
podman push $IMAGE_TAG
```

You can now test with your CNO following the steps in "Test on Existing Cluster".  The only difference is when you set the image, refer to your CNO on quay.io.

```
oc set image deployment/network-operator -n openshift-network-operator network-operator=quay.io/$(whoami)/cluster-network-operator:dev
```
