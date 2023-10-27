---
layout: post
title: Security Context Constraints
subtitle: Manage the security context of your K8s/OCP workloads
tags: [Kubernetes, K8s, OpenShift, OCP, SCC, Security Context Constraints]
author: cmeissner
---

Usually, OpenShift prevents containers running in a cluster from accessing Linux features like shared file systems, root access, and some core capabilities, such as the KILL command, as such protected functions could affect other workloads running in a cluster.

Most workloads are fine with the default, but especially stateful workloads often require additional permissions to run.

Security Context Constraints define which protected functions are allowed and which are not allowed. (see [About security context constraints](https://docs.openshift.com/container-platform/4.11/authentication/managing-security-context-constraints.html#security-context-constraints-about_configuring-internal-oauth)) One of the defaults SCC is `restricted-v2` which is applied by default to any workload.

```shell
$ oc describe scc/restricted-v2
Name:                                           restricted-v2
Priority:                                       <none>
Access:
  Users:                                        <none>
  Groups:                                       <none>
Settings:
  Allow Privileged:                             false
  Allow Privilege Escalation:                   false
  Default Add Capabilities:                     <none>
  Required Drop Capabilities:                   ALL
  Allowed Capabilities:                         NET_BIND_SERVICE
  Allowed Seccomp Profiles:                     runtime/default
  Allowed Volume Types:                         configMap,csi,downwardAPI,emptyDir,ephemeral,persistentVolumeClaim,projected,secret
  Allowed Flexvolumes:                          <all>
  Allowed Unsafe Sysctls:                       <none>
  Forbidden Sysctls:                            <none>
  Allow Host Network:                           false
  Allow Host Ports:                             false
  Allow Host PID:                               false
  Allow Host IPC:                               false
  Read Only Root Filesystem:                    false
  Run As User Strategy: MustRunAsRange
    UID:                                        <none>
    UID Range Min:                              <none>
    UID Range Max:                              <none>
  SELinux Context Strategy: MustRunAs
    User:                                       <none>
    Role:                                       <none>
    Type:                                       <none>
    Level:                                      <none>
  FSGroup Strategy: MustRunAs
    Ranges:                                     <none>
  Supplemental Groups Strategy: RunAsAny
    Ranges:                                     <none>
```

This listing shows the permission set default (`restricted-v2`) SSC. These permissions can be grouped as followed:

- Privileges
- Access controls
- Capabilities

### Privileges

Privileges defines general authorities a pod has when it is deployed. These privileges start in SCC with `allow*` and are represented by boolean values, where `true` means `allowed` and `false` stands for `forbidden`.

```yaml
allowPrivilegeEscalation: false
allowPrivilegedContainer: false
```

To obtain a certain privilege, you need to include the following parameter in your deployment for a container or all containers in the pod, as follows.

```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: scc-test
spec:
 template:
    ...
    spec:
      containers:
        - name: webserver
          image: quay.io/redhattraining/hello-world-nginx
          ...
          securityContext:
            allowPrivilegedContainer: false
      serviceAccount: scc-flow-test-sa
 ...
```

### Access controls

This defines which specific UID and GID a pod can be run under. The control of access is governed by a set of permissible parameters

- runAsUser - the range of user IDs the container is allowed to run under
- supplementalGroups - the range of groups IDs the container is allowed to run under
- fsGroup - defines the range of groups IDs the pod will use for controlling pod storage volumes
- seLinuxContext - defines the allowed SELinux context

and by a strategy

- MustRunAs and MustRunAsRange - enforces the allowed range and sets the default value
- RunAsAny - indicates that no default is provided and allows any runAsUser to be specified
- MustRunAsNonRoot -  indicates that any non-root UID is allowed to use

```yaml
runAsUser:
  type: MustRunAsRange
  uidRequestMin: 4000
  uidRequestMax: 4999
```

To request access corresponding to the SCC, it is necessary to include the following section in your deployment.

```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: scc-test
spec:
 template:
    ...
    spec:
      containers:
        - name: webserver
          image: quay.io/redhattraining/hello-world-nginx
          ...
          securityContext:
            runAsUser: 4567
      serviceAccount: scc-flow-test-sa
 ...
```

### Capabilities

With these settings, you will be able to manage access to [Linux capabilities](https://man7.org/linux/man-pages/man7/capabilities.7.html). Starting with kernel 2.2, Linux divides the privileges traditionally associated with superuser into distinct units, known as capabilities, which can be independently enabled and disabled. 
As a default, all capabilities were dropped `and NET_BIND_SERVICE is` explicitly added in restricted-v2 SCC. You should always only add the capabilities you really need for your work 

Managing capabilities in SCCs is done by the following parameters:

- defaultAddCapabilities - list of default capabilities automatically added, set to `null` in `restricted-v2` SCC
- requiredDropCapabilities - list of capabilities that are dropped and thus forbidden for each container
- allowedCapabilities - list of capabilities that are allowed to be requested by a Deployment

For example, a deployment that requests capabilities could look like this:

```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: scc-test
spec:
 template:
    ...
    spec:
      containers:
        - name: webserver
          image: quay.io/redhattraining/hello-world-nginx
          ...
          securityContext:
            capabilities:
              add: ["NET_ADMIN", "SYS_TIME"]
      serviceAccount: scc-flow-test-sa
 ...
```

## Personas

It's important to know which personas have to be considered when running a workload with other permissions. The following three personas are discussed in this article:

- Developer
- Deployer
- Cluster Administrator

### Developer

The deployer creates a deployment for that piece of software and applies a Security Context, either for a specific container or for all the containers in a pod at once. You need to use a Service Account for that deployment. Since a deployer does not have broader permissions on an OpenShift cluster, it is necessary to request the used Service Account and a matching SCC that is applied to this SA.

### Deployer

The deployer creates a deployment for that piece of software and applies a Security Context, either for a specific container or for all containers in a pod. It is also necessary to use a Service Account for that deployment.
As a deployer has not broader permissions on a OpenShift cluster, it is necessary to request the used Service Account and a matching SCC that is applied to this SA.

### Cluster Administrator

The cluster administrator initiates the requested SA and assigns a compatible Security Context Constraint to it. This Security Context Constraint can either be a predefined or a custom one.

## Security Admission

The admission procedure compares the security context with the security context constraint assigned to the service account. The deployment is allowed if the SCC matches the requested privileges, otherwise it will be blocked.

![SCC Admission flow](/assets/img/scc_admission_flow.png){:.mx-auto.d-block :}

### SCC Ordering

The Admission Controller manages the ordering of all assigned SCCs, so you need to understand how they manage the ordering of all assigned SCCs.

1. List of SCCs that could be assigned to the pod based on the pods specs and the SCCs the User/ServiceAccount who created the pod can use.
2. SCCs in the list are ordered as follows:
    - If the SCCs have different priorities, higher priority first.
    - If priority is the same, the most restrictive first.
    - If priority and restriction level are the same, by alphabetical order first.

## Dos and don'ts

For the work with SCCs, there should be some rules that should be considered. Here are some common ones to give you a good starting point.

### Predefined SCCs

OpenShift is equipped with a comprehensive array of [predefined SCCs](https://docs.openshift.com/container-platform/4.11/authentication/managing-security-context-constraints.html#default-sccs_configuring-internal-oauth), which are designed to cater to the majority of prevalent usage scenarios.

These SCCs can be used as-is, for example, by applying `anyuid` it let a container run under any UID, including UID 0.

#### Default SCC

Starting with OpenShift 4.11+, [Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/) is enabled by default. With this version, new SCCs (`*-v2`) were established to conform to the [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/). If no SCCs is applied to the ServiceAccount of a Deployment, `restricted-v2` will be applied to this workload.

#### Adaptation

If it is necessary to modify an SCC, it is advisable not to do so within the SCCs that are provided with OpenShift. Modifying predefined SCCs can have a detrimental impact on the health of your cluster, as the SCCs included with OpenShift are extensively utilized by the cluster components, and it may result in unanticipated adverse effects on the cluster workloads.

Instead of modifying a predefined SCC, you can make a copy of the SCC and then adapt and make your changes there.

### ServiceAccount - SCC assignment

If you create a Project, OpenShift will create three default SAs

```shell
$ oc get sa -n <Project>
NAME       SECRETS   AGE
builder    1         21s
default    1         21s
deployer   1         21s
```

The `default` SA is used for each Deployment where no particularly `serviceAccount` parameter is set. It is best practice to have a seperate Service Account for each workload or groups of workloads with same needs.

#### Assign SCC to SA

If you need to use protected functions for your workload, you should select a SA specifically for this task and assign an SCC to this SA. This habit helps to implement the least privileges approach. Otherwise, all workloads will run with these privileges if you apply an SCC with broader permissions to the `default` SA.

#### Use of Roles

Assigning an SCC to an SA will work fine, but you should think about adding a Role to it. You can assign different SCCs to different SAs by changing the role, instead of handling each SCC-SA-assignment individually.
