## Overview
`kube-bench` runs YAML specifications of the CIS Kubernetes Benchmarks against
a kubernetes cluster.

## Architecture
benchmark spec  --> kube-bench <-- configuration files
                        |
                        v
                     results 


## Benchmark Specification
Benchmark specifications are YAML representation of the CIS Kubernetes Benchmark.

### Structure
The `kube-bench` benchmark specification is a collection checks in a YAML file
that are run against a specific type of node in a kubernetes cluster based on
recommendations in the CIS Kubernetes Benchmark.

The kubernetes node types are the `master` which runs the kubernetes contol 
plane components such as the apiserver, controller-manager and scheduler, and 
the `node` which workloads scheduled on.

The following is a basic `kube-bench` benchmark spec:
```yaml
---
controls:
id: 1
text: "Master Node Security Configuration"
type: "master"
groups:
- id: 1.1
  text: API Server
  checks:
    - id: 1.1.1
      text: "Ensure that the --allow-privileged argument is set (Scored)"
      audit: "ps -ef | grep kube-apiserver | grep -v grep"
      tests:
      bin_op: or
      test_items:
      - flag: "--allow-privileged"
        set: true
      - flag: "--some-other-flag"
        set: false
      remediation: "Edit the /etc/kubernetes/config file on the master node and
        set the KUBE_ALLOW_PRIV parameter to '--allow-privileged=false'"
      scored: true
```

From the basic benchmark spec above, you can see that the YAML document has an
id, a text, a type and a groups field.

The id field is the top-level group for all checks in this spec. This id is
serves an information and organization purpose. It is displayed in the output
of kube-bench.

The text field describes the purpose of the benchmark and all checks in it.
This field is also displayed in the kube-bench output.

The type field 

A `check` (recommendation in the CIS Kubernetes Benchmark) is made up of an id,
description, commands to audit the cluster , tests against the output of these 
commands to determine if a cluster passed or failed the check, remediation, 
steps to remediate a failed check and finally a scored field that indicates if a
check is used in calculating a [CIS Benchmark Score](http://citation.needed).

This an example `check` object:
```
id: 1.1.1
text: "Ensure that the --anonymous-auth argument is set to false (Not Scored)"
audit: "ps -ef | grep kube-apiserver | grep -v grep"
tests:
  test_items:
    - flag: "--anonymous-auth"
      compare:
        op: eq
        value: false
      set: true
      remediation: |
        Edit the API server pod specification file $apiserverconf
        on the master node and set the below parameter.
        --anonymous-auth=false
      scored: false
```

A `group` comprises an id, a description and a list of checks which are run 
against the same kubernetes node type and target a specific component which runs
on that node type.

```yaml
id: 1.1
text: API Server
checks:
  - id: 1.1.1
    text: "Ensure that the --allow-privileged argument is set (Scored)"
    audit: "ps -ef | grep kube-apiserver | grep -v grep"
    tests:
    ...
  - id: 1.1.2
    text: "Ensure that the --anonymous-auth argument is set to false (Not Scored)"
    audit: "ps -ef | grep kube-apiserver | grep -v grep"
    tests:
    ...
    
```

For example, the group 1.1 is a collection of checks for the kube-apiserver that
runs on the kubernetes master. Also the group 1.2 is a collection of checks for
scheduler which runs on the kubernetes master as well.

In `kube-bench` all `group`s which run `check`s against the same kubernetes node
type, for example the kubernetes master are grouped into the same file.
This file is the benchmark spec and is named for the node type

For example all kubernetes 1.13 master checks for are in the directory 
`cfg/1.13/master.yaml` and all the node checks for the same version are in the
file `cfg/1.13/node.yaml`.

### Checks and tests
Tests are the items we actually look for to determine if a check is successful or not. Checks can have multiple tests, which must all be successful for the check to pass.

The syntax for tests:
```
tests:
- flag:
  set:
  compare:
    op:
    value:
...
```
Tests have various `operations` which are used to compare the output of audit commands for success.
These operations are:

- `eq`: tests if the flag value is equal to the compared value.
- `noteq`: tests if the flag value is unequal to the compared value.
- `gt`: tests if the flag value is greater than the compared value.
- `gte`: tests if the flag value is greater than or equal to the compared value.
- `lt`: tests if the flag value is less than the compared value.
- `lte`: tests if the flag value is less than or equal to the compared value.
- `has`: tests if the flag value contains the compared value.
- `nothave`: tests if the flag value does not contain the compared value.


### Variables

Kubernetes config and binary file locations and names can vary from installation to installation, so these are configurable in the `cfg/config.yaml` file.
For each type of node (*master*, *node* or *federated*) there is a list of components, and for each component there is a set of binaries (*bins*) and config files (*confs*) that kube-bench will look for (in the order they are listed). If your installation uses a different binary name or config file location for a Kubernetes component, you can add it to `cfg/config.yaml`.

* **bins** - If there is a *bins* list for a component, at least one of these binaries must be running. The tests will consider the parameters for the first binary in the list found to be running.
* **podspecs** - From version 1.2.0 of the benchmark (tests for Kubernetes 1.8), the remediation instructions were updated to assume that the configuration for several kubernetes components is defined in a pod YAML file, and podspec settings define where to look for that configuration.
* **confs** - If one of the listed config files is found, this will be considered for the test. Tests can continue even if no config file is found. If no file is found at any of the listed locations, and a *defaultconf* location is given for the component, the test will give remediation advice using the *defaultconf* location.
* **unitfiles** - From version 1.2.0 of the benchmark  (tests for Kubernetes 1.8), the remediation instructions were updated to assume that kubelet configuration is defined in a service file, and this setting defines where to look for that configuration.


### Versions and distributions
`kube-bench` supports benchmark specs for multiple Kubernetes versions and distributions.
The supported versions and distributions can be found under the `cfg` directory of the
root directory.

To see a list of supported kubernetes versions and distributions, run
```shell
cd $GOPATH/src/github.com/aquasecurity/kube-bench
ls -1 cfg
1.11
1.13
1.6
1.7
1.8
config.yaml
ocp-3.10
```

The versions listed in `cfg` are specifically kubernetes versions not CIS
Kubernetes Benchmark versions and they are not the same. Please refer to the version
matrix below to see how kubernetes versions and release map to CIS Kubernete Benchmarks.


### Test config YAML representation
The tests are represented as YAML documents (installed by default into ./cfg).

An example is as listed below:
```
---
controls:
id: 1
text: "Master Checks"
type: "master"
groups:
- id: 1.1
  text: "Kube-apiserver"
  checks:
    - id: 1.1.1
      text: "Ensure that the --allow-privileged argument is set (Scored)"
      audit: "ps -ef | grep kube-apiserver | grep -v grep"
      tests:
      bin_op: or
      test_items:
      - flag: "--allow-privileged"
        set: true
      - flag: "--some-other-flag"
        set: false
      remediation: "Edit the /etc/kubernetes/config file on the master node and set the KUBE_ALLOW_PRIV parameter to '--allow-privileged=false'"
      scored: true
```

Recommendations (called `checks` in this document) can run on Kubernetes Master, Node or Federated API Servers.
Checks are organized into `groups` which share similar controls (things to check for) and are grouped together in the section of the CIS Kubernetes document.
These groups are further organized under `controls` which can be of the type `master`, `node` or `federated apiserver` to reflect the various Kubernetes node types.


# Roadmap
Going forward we plan to release updates to kube-bench to add support for new releases of the Benchmark, which in turn we can anticipate being made for each new Kubernetes release.

We welcome PRs and issue reports.

# Testing locally with kind

Our makefile contains targets to test your current version of kube-bench inside a [Kind](https://kind.sigs.k8s.io/) cluster. This can be very handy if you don't want to run a real kubernetes cluster for development purpose.

First you'll need to create the cluster using `make kind-test-cluster` this will create a new cluster if it cannot be found on your machine. By default the cluster is named `kube-bench` but you can change the name by using the environment variable `KIND_PROFILE`.

*If kind cannot be found on your system the target will try to install it using `go get`*

Next you'll have to build the kube-bench docker image using `make build-docker`, then we will be able to push the docker image to the cluster using `make kind-push`.

Finally we can use the `make kind-run` target to run the current version of kube-bench in the cluster and follow the logs of pods created. (Ctrl+C to exit)

Everytime you want to test a change, you'll need to rebuild the docker image and push it to cluster before running it again. ( `make build-docker kind-push kind-run` )

