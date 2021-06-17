---
title: "Control Plane"
date: 2020-09-13T17:33:04+01:00
weight: 10
---

Let's start our eploration with the first stage of any Kubernetes cluster's lifecycle -- bootstrapping. At this stage, a cluster admin is expected to provide a number of parameters one of which will be called `service-cidr` (or something similar depending on the orchestrator) which gets mapped to a `service-cluster-ip-range` argument of the `kube-apiserver`.

{{% notice note %}}
For the sake of simplicity we'll assume `kubeadm` is used to orchestrate a cluster.
{{% /notice %}}

An  Orchestrator will suggest a default value for this range (e.g. `10.96.0.0/12`) which most of the times is safe to use. As we'll see later, this range is completely "virtual", i.e. does not need to have any coordination with the underlying network and can be re-used between clusters (one notable exception being [this Calico feature](https://docs.projectcalico.org/networking/advertise-service-ips#advertise-service-cluster-ip-addresses)). The only constraints for this value are:

- It must not overlap with any of the Pod IP ranges or Node IPs of the same cluster.
- It must not be loopback (127.0.0.0/8 for IPv4, ::1/128 for IPv6) or link-local (169.254.0.0/16 and 224.0.0.0/24 for IPv4, fe80::/64 for IPv6).

Once a Kubernetes cluster has been bootstrapped, every new `ClusterIP` ServiceType will get a unique IP allocated from this range, for example:


```yaml
$ kubectl create svc clusterip test --tcp=80 && kubectl get svc test
service/test created
NAME   TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
test   ClusterIP   10.96.37.70   <none>        80/TCP    0s
```

{{% notice info %}}
The first IP from the Service CIDR range is reserved and always assigned to a special `kubernetes` service. See [this explanation](https://networkop.co.uk/post/2020-06-kubernetes-default/) for more details.
{{% /notice %}}


Inside the Endpoints controller's [reconciliation loop](https://github.com/kubernetes/kubernetes/blob/52eea971c57580c6b1b74f0a12bf9cc6083a4d6b/pkg/controller/endpoint/endpoints_controller.go#L378), it builds an internal state for each service which includes a list of all associated Endpoints. From then on, both `Service` and `Endpoints` resources co-exist, with the former being the use-facing, aggregated view of a load-balancer and the latter being the detailed, low-level set of individual endpoints that will be programmed in the dataplane.  There are two ways these Endpoints are determined, starting with the most common way:


- **Label selectors** allow Service controllers [identify](https://github.com/kubernetes/kubernetes/blob/52eea971c57580c6b1b74f0a12bf9cc6083a4d6b/pkg/controller/endpoint/endpoints_controller.go#L414) all matching Pods, and collect their [IP](https://github.com/kubernetes/kubernetes/blob/52eea971c57580c6b1b74f0a12bf9cc6083a4d6b/pkg/controller/endpoint/endpoints_controller.go#L259) and [port](https://github.com/kubernetes/kubernetes/blob/52eea971c57580c6b1b74f0a12bf9cc6083a4d6b/pkg/controller/endpoint/endpoints_controller.go#L479) information.
- **Manual configuration** relies on users to assemble their own set of Endpoints; this approach is very rarely used but can enable intra-cluster addresses for services located outside of the cluster.

All Endpoints are stored in an `Endpoints` resource that bears the same name as its parent Service. Below is an example of how it might look like for the `kubernetes` service:

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  labels:
    endpointslice.kubernetes.io/skip-mirror: "true"
  name: kubernetes
  namespace: default
subsets:
- addresses:
  - ip: 172.18.0.4
  ports:
  - name: https
    port: 6443
    protocol: TCP
```

{{% notice info %}}
Under the hood Endpoints are implemented as a set of slices; this will be covered in the Optimisations sections.
{{% /notice %}}

It is worth noting that the [DNS Spec](https://github.com/kubernetes/dns/blob/master/docs/specification.md#23---records-for-a-service-with-clusterip), mentioned briefly in the previous chapter, also defines the behaviour for `ClusterIP` type services. Specifically the following 3 query types must be supported:

* A/AAAA Records -- will return a single ClusterIP for any query matching the Service Name in the same namespace or `<service>.<ns>.svc.<zone>` in a different namespace.
* SRV Record -- will return an SRV record for each unique port + protocol combination.
* PTR Record -- can be used to lookup a service name based on provided `ClusterIP`.


---

The Kubernetes' `kube-controller-manager` is constantly collecting, processing and updating all Endpoints and Service resources, however nothing is being done with this information yet. The ultimate consumers of it are a set of node-local agents (controllers) that will use this information to program their local dataplanes. Most of these node-local agents are using 
[client-go]((https://github.com/kubernetes/sample-controller/blob/master/docs/controller-client-go.md)) to synchronize and process updates coming the the API server, which means they all will share the following optimisations:

* Each node-local agent maintins a local cache of all interesting objects, which gets sync'ed in the beginning (via `List` operation) and observed for the remainder of the agents' lifecycle (via `Watch` operation).
* The architecture with two queues and a local cache ensures that controllers can absorb multiple frequent changes of the same object thereby minimising the churn in the dataplane.
