---
layout: post
title: Security Context Constraints
subtitle: Manage the security context of your K8s/OCP workloads
tags: [Kubernetes, K8s, OpenShift, OCP, SCC, Security Context Constraints]
author: cmeissner
---

Usually, OpenShift prevents containers running in a cluster from accessing Linux features like shared file systems, root access and some core capabilities e.g. the KILL command as such protected functions could affect other workloads running in the same kernel.

Most workloads are fine with the default, but especially stateful workloads often need more permissions to run.

Security Context Constraints define which protected functions are allowed and which are not allowed. (see [About security context constraints](https://docs.openshift.com/container-platform/4.11/authentication/managing-security-context-constraints.html#security-context-constraints-about_configuring-internal-oauth)) One of the defaults SCC is `restricted-v2` which is applied by default to any workload.

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

To request a given privilege, you need to pass the following parameter in your Deployment for a container or all containers in the pod as followed:

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

Defines under which specific UID and GID a pod can be run. Access controls are controlled by an allowable set

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

To request access corresponding to the SCC, you need to put the following section into your Deployment.

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

With these settings, you will be able to manage the access to [Linux capabilities](https://man7.org/linux/man-pages/man7/capabilities.7.html). Starting with kernel 2.2, Linux divides the privileges traditionally associated with superuser into distinct units, known as capabilities, which can be independently enabled and disabled.
Per default, all capabilities were dropped and `NET_BIND_SERVICE` is explicitly added in `restricted-v2` SCC, and you should always only add these capabilities you really need for your workload.

Managing capabilities in SCCs is done by the following parameters:

- defaultAddCapabilities - list of default capabilities automatically added, set to `null` in `restricted-v2` SCC
- requiredDropCapabilities - list of capabilities that are dropped and thus forbidden for each container
- allowedCapabilities - list of capabilities that are allowed to be requested by a Deployment

A Deployment which requests capabilities could look like this:

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

To run a workload with other permissions as the default one, you need to know which personas have to consider.
We discuss here the following 3 personas:

- Developer
- Deployer
- Cluster Administrator

### Developer

The developer creates a software which needs some protected functions to run. But a developer often does not know about Security Contexts nor Security Context Constraints.

### Deployer

The deployer creates a deployment for that piece of software and applies a Security Context, either for a specific container or for all containers in a pod. It is also necessary to use a Service Account for that deployment.
As a deployer has not broader permissions on a OpenShift cluster, it is necessary to request the used Service Account and a matching SCC that is applied to this SA.

### Cluster Administrator

The cluster administrator creates the requested SA and assigns a matching Security Context Constraint to it. This SCC can either be a predefined or a custom Security Context Constraint.

## Security Admission

The admission process compares the Security Context with the Security Context Constraint assigned to the Service Account. If the SCC matches the requested privileges the deployment is allowed, if not, it will be blocked.

![SCC Admission flow](/assets/img/scc_admission_flow.png){:.mx-auto.d-block :}

### SCC Ordering

As you can assign more than one SCC to a Role or a ServiceAccount you need to understand how the Admission Controller manages the ordering of all assigned SCCs.

1. A list of potential SCCs to be assigned to the pod based on the pod spec and the SCCs the User/ServiceAccount creating the pod can use.
2. SCCs in the list are ordered as follows:
    - If the SCCs have different priorities, higher priority first.
    - If priority is the same, the most restrictive first.
    - If priority and restriction level are the same, by alphabetical order first.

## Dos and don'ts

For the work with SCCs, some rules should be considered. We will list here some of the common ones to give you a good starting point.

### Predefined SCCs

OpenShift comes with a rich set of [predefined SCCs](https://docs.openshift.com/container-platform/4.11/authentication/managing-security-context-constraints.html#default-sccs_configuring-internal-oauth), which should cover most of the common use cases.

These SCCs can be used as-is, e.g. by applying `anyuid` to let a container run under any UID, including UID 0.

#### Default SCC

Starting with OpenShift 4.11+ (Kubernetes 1.25) [Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/) become stable and is enabled by default.  With this version, new SCCs `*-v2`were established to meet the [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/). If no SCCs is applied to the ServiceAccount of a Deployment, `restricted-v2` will be applied to this workload.

#### Adaptation

If you need to adapt an SCC, **never** do this within the SCCs coming with OpenShift. Modifying predefined SCCs can harm your cluster health as the SCCs included with OpenShift are heavily used by the cluster components, and it can cause unexpected side effects on cluster workloads.

Instead of modifying a predefined SCC, make a copy of the SCC, you need to adapt and make your changes there.

### ServiceAccount - SCC assignment

If you create a Project, OpenShift will create three default SAs

```shell
$ oc get sa -n <Project>
NAME       SECRETS   AGE
builder    1         21s
default    1         21s
deployer   1         21s
```

The `default` SA is used for each Deployment where no particularly `serviceAccount` parameter set.

#### Assign SCC to SA

If you have to use protected functions for your workload, you **should use a SA especially** for this workload and **assign an SCC to this SA only**.
This habit helps to implement the least privileges approach. Otherwise, if you apply an SCC with broader permissions to the `default` SA, all workloads will run with these privileges.

#### Use of Roles

Assigning an SCC directly to an SA will work fine, but you should consider putting a Role into it. So you will be able to apply different SCCs to different SAs by changing the role and not to handle each SCC-SA-assignment individually.
