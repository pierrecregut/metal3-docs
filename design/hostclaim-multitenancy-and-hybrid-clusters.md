<!--
 This work is licensed under a Creative Commons Attribution 3.0
 Unported License.

 http://creativecommons.org/licenses/by/3.0/legalcode
-->

<!-- cSpell:ignore Sylva Schiff Kanod argocd GitOps -->
# HostClaim: multi-tenancy and hybrid clusters

## Status

provisional

## Summary

We introduce a new Custom Resource (named HostClaim) which will facilitate
the creation of multiple clusters for different tenants. It also provides
a framework for building clusters with different kind of compute resource:
baremetal servers but also virtual machines hosted in private or public cloud.

A HostClaim decouples the client need from the actual implementation of the
compute resource: it establishes a security boundary and provides a way to
migrate nodes between different kind of compute resources.

A HostClaim expresses that one wants to start a given
OS image with an initial configuration (typically cloud-init or ignition
configuration files) on a compute resource that meets a set of requirements
(host selectors). These requirements can be interpreted either as labels
for a recyclable resource (such as a bare-metal server) or as characteristics
for a disposable resource (such as a virtual machine created for the workload).
The status and meta-data of the HostClaim provide the necessary information
for the end user to define and manage his workload on the compute resource,
but they do not grant full control over the resource (typically, BMC
credentials of servers are not exposed to the tenant).

## Motivation

So far, the primary use case of cluster-api-baremetal is the creation of a
single target cluster from a temporary management cluster. The pivot process
transfers the resources describing the target cluster from the management
cluster to the target cluster. Once the pivot process is complete, the target
cluster takes over all the servers. It can scale based on its workload but it
cannot share its servers with other clusters.

There is another model where a single management cluster is used to create and
manage several clusters across a set of bare-metal servers. This is the focus
of the [Sylva Project](https://sylvaproject.org/) of the Linux foundation.
Another example is [Das Schiff](https://github.com/telekom/das-schiff).

One of the issue encountered today is that the compute resources
(BareMetalHost) and the cluster definition (Cluster, MachineDeployment,
Machines, Metal3Machines, etc.) must be in the same namespace. Since the goal
is to share the compute resources, this means that a single namespace is used
for all resources. Consequently, unless very complex access control
rules are defined, cluster administrators have visibility over all clusters
and full control over the servers as the credentials are stored in the same
namespace.

The solution so far is to completely proxy the access to the Kubernetes
resources that define the clusters.

Another unrelated problem is that Cluster-API has been designed
to define clusters using homogeneous compute resources: it is challenging to
define a cluster with both bare-metal servers and virtual machines in a private
or public cloud.
This [blog post](https://metal3.io/blog/2022/07/08/One_cluster_multiple_providers.html)
proposes several approaches but none is entirely satisfactory.

On the other hand, workloads can be easily defined in terms of OS images and
initial configurations and standards such as qcow or cloud-init have emerged
and are used by various infrastructure technologies.
Due to the unique challenges of managing bare-metal, the Metal3 project has
developed a set of abstractions and tools that could be used in different
settings. The main mechanism is the selection process that occurs between
the Metal3Machine and the BareMetalHost which assigns a disposable workload
(being a Kubernetes node) to a recyclable compute resource (a
server).

This proposal introduces a new resource called HostClaim that solves
both problems by decoupling the definition of the workload performed by
the Metal3 machine controller from the actual compute resource.
This resource acts as both a security boundary and a way to hide the
implementation details of the compute resource.

### Goals

* Split responsibilities between infrastructure teams, who manage servers, and
  cluster administrators, who create/update/scale baremetal clusters deployed
  on those servers, using traditional Kubernetes RBAC to ensure isolation.
* Provide a framework where cluster administrators can consume compute
  resources that are not baremetal servers, as long as they offer similar APIs,
  using the cluster-api-provider-metal3 to manage the life-cycle of those
  resources.
* Define a resource where a user can request a compute resource to execute
  an arbitrary workload described by an OS image and an initial configuration.
  The user does not need to know exactly which resource is used and may not
  have full control over this resource (typically no BMC access).

### Non-Goals

* How to implement HostClaim for specific compute resources that are not
  BareMetalHost.
* Discovery of which capabilities are exposed by the cluster.
  Which kind of compute resources are available and the semantics of the
  selectors are not handled.
* Compute resource quotas. The HostClaim resource should make it possible to
  develop a framework to limit the number/size of compute resources allocated
  to a tenant, similar to how quotas work for pods. However, the specification
  of such a framework will be addressed in another design document.
* Pivoting client clusters resources (managed clusters that are not the
  initial cluster).

## Proposal

### User Stories

#### Deployment of Simple Workloads

As a user I would like to execute a workload on an arbitrary server.

The OS image is available in qcow format on a remote server at ``url_image``.
It supports cloud-init and a script can launch the workload at boot time
(e.g., a systemd service).

The cluster offers bare-metal as a service using Metal3 baremetal-operator.
However, as a regular user, I am not allowed to directly access the definitions
of the servers. All servers are labeled with an ``infra-kind`` label whose
value depends on the characteristics of the computer.

* I create a resource with the following content:

  ```yaml
  apiVersion: metal3.io/v1alpha1
  kind: HostClaim
  metadata:
    name: my-host
  spec:
    online: false
    kind: baremetal

    hostSelector:
      matchLabels:
        infra-kind: medium
  ```

* After a while, the system associates the claim with a real server, and
  the resource's status is populated with the following information:

  ```yaml
  status:
    addresses:
      - address: 192.168.133.33
        type: InternalIP
      - address: fe80::6be8:1f93:7f65:59cf%ens3
        type: InternalIP
      - address: localhost.localdomain
        type: Hostname
      - address: localhost.localdomain
        type: InternalDNS
      bootMACAddress: "52:54:00:01:00:05"
      conditions:
      - lastTransitionTime: "2024-03-29T14:33:19Z"
        status: "True"
        type: Ready
      - lastTransitionTime: "2024-03-29T14:33:19Z"
        status: "True"
        type: AssociateBMH
      lastUpdated: "2024-03-29T14:33:19Z"
      nics:
      - MAC: "52:54:00:01:00:05"
        ip: 192.168.133.33
        name: ens3
  ```

* I also examine the annotations and labels of the HostClaim resource. They
  have been enriched with information from the BareMetalHost resource.
* I create three secrets in the same namespace ``my-user-data``,
  ``my-meta-data``, and ``my-network-data``. I use the information from the
  status and meta data to customize the scripts they contain.
* I modify the HostClaim to point to those secrets and start the server:

  ```yaml
  apiVersion: metal3.io/v1alpha1
  kind: HostClaim
  metadata:
    name: my-host
  spec:
    online: true
    image:
      checksum: https://url_image.qcow2.md5
      url: https://url_image.qcow2
      format: qcow2
    userData:
      name: my-user-data
    networkData:
      name: my-network-data
    kind: baremetal
    hostSelector:
      matchLabels:
        infra-kind: medium
  ```

* The workload is launched. When the machine is fully provisioned, the boolean
  field ready in the status becomes true. I can stop the server by changing
  the online status. I can also perform a reboot by targeting specific
  annotations in the reserved ``host.metal3.io`` domain.
* When I destroy the host, the association is broken and another user can take
  over the server.

#### Multi-tenancy

As an infrastructure administrator I would like to host several isolated clusters

All the servers in the data-center are registered as BareMetalHost in one or
several namespaces under the control of the infrastructure manager. Namespaces
are created for each tenant of the infrastructure. They create
standard cluster definitions in those namespaces. The only difference with
standard baremetal cluster definitions is the presence of a ``kind`` field in
the Metal3Machine templates. The value of this field is set to ``baremetal``.

When the cluster is started, a HostClaim is created for each Metal3Machine
associated to the cluster. The ``hostSelector`` and ``kind`` fields are
inherited from the Metal3Machine. They are used to define the BareMetalHost
associated with the cluster. The associated BareMetalHost is not in the same
namespace as the HostClaim. The exact definition of the BareMetalHost remains
hidden from the cluster user, but parts of its status and metadata are copied
back to the Host namespace. With this information,
the data template controller has enough details to compute the different
secrets (userData, metaData and networkData) associated to the Metal3Machine.
Those secrets are linked to the HostClaim and, ultimately, to the
BareMetalHost.

When the cluster is modified, new Machine and Metal3Machine resources replace
the previous ones. The HostClaims follow the life-cycle of the Metal3Machines
and are destroyed unless they are tagged for node reuse. The BareMetalHosts are
recycled and are bound to new HostClaims, potentially belonging to other
clusters.

#### Hybrid Clusters

As a cluster administrator I would like to build a cluster with different kind of nodes.

This scenario assumes that:

* the cloud technologies CT_i use qcow images and cloud-init to
  define workloads.
* Clouds C_i implementing CT_i are accessible through
  credentials and endpoints described in a resource Cr_i.
* a HostClaim controller exists for each CT_i. Compute resource can
  be parameterize through arguments in the HostClaim arg_ij.

One can build a cluster where each machine deployment MD_i
contains Metal3Machine templates referring to kind CT_i.
The arguments identify the credentials to use (CR_i)

```yaml
apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
kind: Metal3MachineTemplate
metadata:
  name: md-i
spec:
  dataTemplate:
    name: dt-i
  kind: CT_i
  args:
    arg_i1: v1
    ...
    arg_ik: vk
  hostSelector:
    matchLabels:
      ...
    image:
      checksum: https://image_url.qcow2.md5
      format: qcow2
      url: https://image_url.qcow2.md5
```

The Metal3Machine controllers will create HostClaims with different kinds
handled by different controllers creating the different compute resources.
Connectivity must be established between the different subnets where each
controller creates its compute resources.

The argument extension is not strictly necessary but is cleaner than using
matchLabels in specific domains to convey information to controllers.
Controllers for disposable resources such as virtual machine typically do not
use hostSelectors. Controllers for a "bare-metal as a service" service
may use selectors.

#### Manager Cluster Bootstrap

As a cluster administrator I would like to install a new baremetal cluster from a transient cluster.

The bootstrap process can be performed as usual from an ephemeral cluster
(e.g., a KinD cluster). The constraint that all resources must be in the same
namespace (Cluster and BareMetalHost resources) must be respected. The
BareMetalHost should be marked as movable.

The only difference with the behavior without Host is the presence of an
intermediate Host resource but the chain of resources is kept during the
transfer and the pause annotation is used to stop Ironic.

Because this operation is only performed by the administrator of a cluster
manager, the fact that the cluster definition and the BareMetalHosts are in
the same namespace should not be an issue.

The tenant clusters cannot be pivoted which can be expected from a security
point of vue as it would give the bare-metal servers credentials to the
tenants. Partial pivot can be achieved with the help of HostClaim replicating
HostClaims on other clusters but the specification of the associated
controller is beyond the scope of this specification.

## Design Details

### Implementation Details/Notes/Constraints

### Risks and Mitigations

#### Security Impact of Making BareMetalHost Selection Cluster-wide

The main difference between Metal3 machines and HostClaims it the
selection process where a HostClaim can be bound to a BareMetalHost
in another namespace. We must make sure that this behavior is expected
from the owner of BareMetalHost resources, especially when we upgrade the
metal3 cluster api provider to a version supporting HostClaim.

The solution is to enforce that BareMetalHost that can be bound to a
HostClaim have a label (proposed name: ``hosts.metal3.io/namespaces``)
restricting authorized HostClaims to specific namespaces. The value could be
either ``*`` for no constraint, or a comma separated list of namespace names.

#### Tenants Trying to Bypass the Selection Mechanism

The fact that a HostClaim is bound to a specific BareMetalHost will appear
as a label in the HostClaim and the HostClaim controller will use it to find
the associated BareMetalHost. This label could be modified by a malicious
tenant.

But the BareMetalHost has also a consumer reference. The label is only an
indication of the binding. If the consumer reference is invalid (different
from the HostClaim label), the label MUST be erased and the HostClaim
controller MUST NOT accept the binding.

#### Performance Impact

The proposal introduces a new resource with an associated controller between
the Metal3Machine and the BareMetalHost. There will be some duplication
of information between the BareMetalHost and the HostClaim status. The impact
for each node should still be limited especially when compared to the cost of
each Ironic action.

Because we plan to have several controllers for different kind of compute
resource, one can expect a few controllers working on the same custom resource.
This may create additional pressure on Kubernetes api server. It is possible
to limit the amount of exchanged information on a specific controller using
server based filters on the watch/list. To use this feature on current
Kubernetes versions, the HostClaim kind field must be copied in a label.

#### Impact on Other Cluster Api Components

There should be none: other components should mostly rely on Machine and Cluster
objects. Some tools may look at Metal3Machine conditions where some condition
names may be modified but the semantic of Ready condition will be preserved.

### Work Items

There are roughly three parts:

* Creation of the HostClaim resource and controller for BareMetal. This is
  a separate project or included as part of the baremetal-operator project.
* Modification of cluster-api-provider-metal3: introduction of a new variant
  of Metal3MachineManager.
* Modification of the companion controllers to bring back feature parity
  with direct binding to BareMetalHosts.

### Dependencies

### Test Plan

### Upgrade / Downgrade Strategy

### Version Skew Strategy

## Drawbacks

## Alternatives

### Multi-Tenancy Without HostClaim

We assume that we have a Kubernetes cluster managing a set of clusters for
cluster administrators (called tenants in the following). Multi-tenancy
is a way to ensure that tenants have only control over their clusters.

There are at least two other ways to implement multi-tenancy without HostClaim.
They roughly correspond to proxy the full definition of the cluster or the
BareMetalHost itself.

#### Isolation Through Overlays

A solution to multi-tenancy is to hide all the cluster resources to the end
user. In that approach clusters and BareMetalHosts are defined in a single
namespace but the process in charge of cluster creation ensures that resources
from different clusters do not overlap.

This approach was explored in the first versions of the Kanod project. Clusters
must be described by the tenant in a git repository and the descriptions are
imported by a GitOps framework (argocd). The definitions are treated by an
argocd plugin that translates the yaml expressing the user intent in Kubernetes
resources and the naming of resources created by this plugin will ensure
isolation.

Rather than a translation plugin, a set of custom resources would be a better
approach. We must ensure that they are defined in separate namespaces.

The approach has many drawbacks:

* The plugin or the controllers for the abstract clusters are fairly complex
  applications if we want to support many options
  and they are part of the trusted computing base of the cluster manager.
* We introduce a new level of access control distinct from the Kubernetes
  model. If we want tooling/observability around the created resources, we
  need custom tools that respect this new policy or we need to reflect
  everything we want to observe in the new custom resources.
* The approach does not solve the hybrid cluster problem.

#### Ephemeral BareMetalHost

Another solution is to have separate namespaces for each cluster but to
import BareMetalHost in those namespaces on demand when new compute resources
are needed.

The cluster needs a resource acting as a source of BareMetalServers
parameterized by requirements on servers and the number of replicas. The
notion of
[BareMetalPool](https://gitlab.com/Orange-OpenSource/kanod/baremetalpool)
in Kanod is the equivalent of replica sets for pods. This notion also appears in
[this proposal](https://github.com/metal3-io/metal3-docs/pull/268) for a
metal3host operator. The number of replicas may be updated by a
[controller](https://gitlab.com/Orange-OpenSource/kanod/kanod-poolscaler)
checking the requirements of machine deployments and control-planes.

The main security risk is that when a cluster releases a
BareMetalHost, it may keep the credentials giving full control over the server.
This can be solved if those credentials are temporary. In Kanod BareMetalPool
obtain new servers from a rest api implemented by a
[BareMetalHost broker](https://gitlab.com/Orange-OpenSource/kanod/brokerdef).
The broker implementation uses either the fact that Redfish is in fact a Rest
API to implement a proxy or the capability of Redfish to create new users
with an Operator role to implemented BareMetalHost resources with a limited
lifetime.

The pros of the approach are:

* Support for pivot operation even for tenant clusters because it exposes a
  complete bare-metal as a service solution.
* Cluster administrators have full access to the BMC and can configure servers
  to their needs with custom procedures (not exposed by standard Metal3
  controllers).
* Network isolation can be done before the BareMetalHost is created in the scope
  of the cluster. There is no transfer of server from one network configuration
  to another that may invalidate parts of introspection.

The cons of the BareMetalPool approach are:

* The implementation of the broker with its dedicated server is fairly complex.
* To have full dynamicity over the pool of servers, a new kind of autoscaler is
  needed.
* Unnecessary inspection of the servers when they are transferred from a
  cluster (tenant) to another one.
* The current implementation of the proxy is limited to the Redfish protocol
  and would require significant work for IPMI.

### Hybrid Clusters Without HostClaim

#### Control-Planes as a Service
The Kubernetes control-plane is an application with a single endpoint. Some
Cluster Api control-plane providers implement directly a factory for new
control-planes without relying on the infrastructure provider. Usually this
control-plane is hosted in the management cluster as a regular Kubernetes
application. [Kamaji]() or []() implement this approach.

The cluster is hybrid because the control-plane pods are not hosted on standard
nodes but workers are usually all implemented by a single infrastructure
provider and are homogeneous.

The approach solves the problem of sharing resources for control-planes but
does not address the creation of clusters with distinct needs for workers.

#### Cluster API Hack

#### Bring Your Own Hosts
Bring Your Own Host (BYOH) is a cluster api provider that uses existing compute 
resources that run a specific agent used for registering the resource and
deploying Kubernetes on the server.

BYOH does not impose many constraints on the compute resource. 
### Alternative implementations of HostClaims
#### HostClaims in the BareMetalHost Namespace

## References
