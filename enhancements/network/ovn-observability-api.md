---
title: ovn-observability-api
authors:
  - npinaeva
reviewers: # Include a comment about what domain expertise a reviewer is expected to bring and what area of the enhancement you expect them to focus on. For example: - "@networkguru, for networking aspects, please look at IP bootstrapping aspect"
  - dceara
  - msherif1234
  - jotak
approvers: # A single approver is preferred, the role of the approver is to raise important questions, help ensure the enhancement receives reviews from all applicable areas/SMEs, and determine when consensus is achieved such that the EP can move forward to implementation.  Having multiple approvers makes it difficult to determine who is responsible for the actual approval.
  - trozet
api-approvers: # In case of new or modified APIs or API extensions (CRDs, aggregated apiservers, webhooks, finalizers). If there is no API change, use "None"
  - "None"
creation-date: 2024-10-02
last-updated: 2024-10-02
tracking-link: # link to the tracking ticket (for example: Jira Feature or Epic ticket) that corresponds to this enhancement
  - https://issues.redhat.com/browse/SDN-5226
---

# OVN Observability API

## Summary

OVN Observability is a feature that provides visibility for ovn-kubernetes-managed network by using sampling mechanism.
That means, that network packets generate samples with user-friendly description to give more information about what and why is happening in the network.
In openshift, this feature is integrated with the Network Observability Operator.

## Motivation

The purpose of this enhancement is to define an API to configure OVN observability in an openshift cluster.

### User Stories

- Observability: As a cluster admin, I want to enable OVN Observability integration with netobserv so that I can use network policy correlation feature.
- Debuggability: As a cluster admin, I want to debug a networking issue using OVN Observability feature.

### Goals

- Design a CRD to configure OVN Observability for given user stories

### Non-Goals

- Discuss OVN Observability feature in general

## Proposal

There is a number of k8s features that can be used to configure OVN Observability right now, and some more that we expect
to be added in the future. 
Currently supported features are:
- NetworkPolicy
- (Baseline)AdminNetworkPolicy
- EgressFirewall
- UDN Isolation
- Multicast ACLs

In the future more observability features, like egressIP and services, but also networking-specific features 
like logical port ingress/egress will be added.

Every feature may set a sampling probability, that defines how often a packet will be sampled, in percent. 100% means every packet
will be sampled, 0% means no packets will be sampled.

Currently only on/off mode is supported, where on mode turns all available features with 100% probability.

### Observability use case

For observability use case, a cluster admin can set a sampling probability for each feature across the whole cluster.
As sampling configuration may affect cluster performance, not every admin may want all features enabled with 100% probability.
We expect this type of configuration to change rarely.

This configuration should be handled by ovn-kubernetes to configure sampling, and by netobserv ebpf agent to "subscribe" to
the samples enabled by this configuration.

### Debuggability use case

Debuggability is expected to be turned on during debugging sessions, and turned off after the session is finished.
It will most likely have 100% probability for required features, and also may enable more networking-related features, like
logical port ingress/egress for cluster admins. Debugging configuration is expected to have more granular filters to
debug specific problems with minimal overhead. For example, per-node or per-namespace filtering may be required.

Debuggability sampling may be used with or without Network Observability Operator being installed:
- without netobserv, samples may be printed or forwarded to a file by ovnkube-observ binary, which is a part of ovnkube-node pod on every node.
- with neobserv, samples may be displayed in a similar to observability use case way, but may need another interface to ensure
cluster users won't be distracted by samples generated for debugging.

This configuration should be handled by ovn-kubernetes to configure sampling, and by netobserv ebpf agent OR ovnkube-observ script
to "subscribe" to the samples enabled by this configuration.

### Workflow Description

1. Cluster admin wants to enable OVN Observability for admin-managed features.
2. Cluster admin created a CR with 100% probabilities for (Baseline)AdminNetworkPolicy and EgressFirewall.
3. Messages like "Allowed by admin network policy test, direction Egress" appear on netobserv UI.
4. Cluster admin sees CPU usage by ovnkube pods grows up 10%, and reduces the sampling probability to 50%.
5. Still most of the flows have correct enrichment with lower overhead.
6. Admin is happy. Until...
7. A user from project "bigbank" reports that they can't access the internet from pod A.
8. Cluster admin enables debugging for project "bigbank" with 100% probability for NetworkPolicies.
9. netobserv UI or ovnkube-oberv script shows a sample with <`source pod A`, `dst_IP`> saying "Denied by network policy in namespace bigbank, direction Egress"
10. Admin now has to convince the user that they are missing an allow egress rule to `dst_IP` for pod A.
11. Network policy is fixed, debugging can be turned off.


### API Extensions

#### Observability filtering for debugging

Mostly for debugging (but also can be used for observability if needed), we may need more granular filtering, 
like per-node or per-namespace filtering.

- Per-node filtering may be implemented by adding a node selector to `ObservabilityConfig`, and is easy to implement in ovn-k
  as every node has its own ovn-k instance with OVN-IC.
- Per-namespace filtering can only be applied to the namespaced Observability Features, like NetworkPolicy and EgressFirewall.
  Cluster-scoped features, like (Baseline)AdminNetworkPolicy, can only be additionally filtered based on name. Label selectors
  can't be used, as observability module doesn't watch any objects.

#### ObservabilityConfig CRD

The CRD for this feature will be a part of ovn-kubernetes to allow using it upstream.

```go

// ObservabilityConfig described OVN Observability configuration.
//
// +genclient
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
// +kubebuilder:resource:path=observabilityconfig,scope=cluster
// +kubebuilder:singular=observabilityconfig
// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
type ObservabilityConfig struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`
	// +kubebuilder:validation:Required
	// +required
	Spec ObservabilitySpec `json:"spec"`
	// +optional
	Status ObservabilityStatus `json:"status,omitempty"`
}

type ObservabilitySpec struct {
	// CollectorID is the ID of the collector that should be unique across the cluster, and will be used to receive configured samples.
	// +kubebuilder:validation:Required
	// +kubebuilder:validation:Minimum=1
	// +required
	CollectorID int32 `json:"collectorID"`
	// Features is a list of Observability features that can generate samples and their probabilities for a given collector.
	// +kubebuilder:validation:Required
	// +kubebuilder:validation:MinItems=1
	// +required
	Features []FeatureConfig `json:"features"`

	// Filter allows to apply ObservabilityConfig in a granular manner.
	// +optional
	Filter *Filter `json:"filter,omitempty"`
}

type FeatureConfig struct {
	// Probability is the probability of the feature being sampled in percent.
	// +kubebuilder:validation:Required
	// +kubebuilder:validation:Minimum=0
	// +kubebuilder:validation:Maximum=100
	// +required
	Probability int32
	// Feature is the Observability feature that should be sampled.
	// +kubebuilder:validation:Required
	// +required
	Feature ObservabilityFeature
}

// ObservabilityStatus contains the observed status of the ObservabilityConfig.
type ObservabilityStatus struct {
	// +listType=map
	// +listMapKey=type
	Conditions []metav1.Condition `json:"conditions,omitempty"`
}

// Filter allows to apply ObservabilityConfig in a granular manner.
// Currently, it supports node and namespace based filtering.
// If both node and namespace filters are specifies, they are logically ANDed.
// +kubebuilder:validation:MinProperties=1
type Filter struct {
	// nodeSelector applies ObservabilityConfig only to nodes that match the selector.
	// +optional
	NodeSelector *metav1.LabelSelector `json:"nodeSelector,omitempty"`
	// namespaces is a list of namespaces to which the ObservabilityConfig should be applied.
	// It only applies to the namespaced features, currently that includes NetworkPolicy and EgressFirewall.
	// +kubebuilder:MinItems=1
	// +optional
	Namespaces *[]string `json:"namespaces,omitempty"`
}

// ObservabilityStatus contains the observed status of the ObservabilityConfig.
type ObservabilityStatus struct {
	// +listType=map
	// +listMapKey=type
	Conditions []metav1.Condition `json:"conditions,omitempty"`
}

// +kubebuilder:validation:Enum=NetworkPolicy;AdminNetworkPolicy;EgressFirewall;UDNIsolation;MulticastIsolation
type ObservabilityFeature string

const (
	NetworkPolicy      ObservabilityFeature = "NetworkPolicy"
	AdminNetworkPolicy ObservabilityFeature = "AdminNetworkPolicy"
	EgressFirewall     ObservabilityFeature = "EgressFirewall"
	UDNIsolation       ObservabilityFeature = "UDNIsolation"
	MulticastIsolation ObservabilityFeature = "MulticastIsolation"
)
```

For now, we only have use cases for 2 different `ObservabilityConfig`s in a cluster, and ovn-kubernetes will likely have a
limit for the number of CRs that will be handled.

For the integration with netobserv, we could use a label on `ObservabilityConfig` to signal that configured samples should be
reflected via netobserv. For example, `network.openshift.io/observability: "true"`.

### Topology Considerations

#### Hypershift / Hosted Control Planes

As long as netobserv can be installed on the management or hosted cluster, integration should work fine.
If we need to make it work for shared-cluster netobserv deployment, more work.testing will be needed.
Netobserv team confirmed that separate installations for management and hosted clusters are supported as a result of https://issues.redhat.com/browse/NETOBSERV-1196.

For debuggability mode without netobserv integration, everything should work separately on every cluster with its own ovn-kubernetes instance.

#### Standalone Clusters
#### Single-node Deployments or MicroShift

SNO should be supported with netobserv, Microshift is not.
Microshift most likely will not be supported even without netobserv, as observability requires new features to be enabled
and adds overhead to the cluster.

### Implementation Details/Notes/Constraints

Filtering for debugging use case may be tricky as explained in "Observability filtering for debugging" section.
But it is important to implement, as OVN implementation works in a way, where performance optimizations can only be
applied when there is only 1 `ObservabilityConfig` for a given feature. That means, adding a second `ObservabilityConfig`
for a feature will have a bigger performance impact than creating a new one.

### Risks and Mitigations

The biggest risk of enabling observability is performance overhead.
This will be mitigated by scale-testing and sampling percentage configuration.

### Drawbacks

- e2e testing upstream is not possible until github runners allow using kernel 6.11

## Test Plan

- e2e downstream
- perf/scale testing for dataplane and control plane as a GA plan
- netobserv testing for correct GUI representation

## Graduation Criteria

### Dev Preview -> Tech Preview

None

### Tech Preview -> GA

- More testing (upgrade, downgrade)
- Available by default, but no overhead until observability config CR is created
- Run perf/scale tests and document related metrics to track perf overhead, write down any limitations/recommendations
based on the workload and cluster size
- User facing documentation created in openshift-docs and netobserv docs
- Enable netobserv metrics based on samples

### Removing a deprecated feature

## Upgrade / Downgrade Strategy

No upgrade impact is expected, as this feature will only be enabled when a new CR is created.

## Version Skew Strategy

All components were designed to be backwards-compatible
- old kernel + new OVS: we control these versions in a cluster and make sure to bump OVS version on supported kernel
- old OVS + new OVN: OVN uses "sample" action that is available in older OVS versions
- old OVN + new ovn-k: rolled out at the same time, no version skew is expected
- old kernel/OVS/ovn-kubernetes + new netobserv: netobserv can detect if it's running on older ocp version and warn when the users try to enable the feature.

## Operational Aspects of API Extensions

Perf/scale considerations will be documented as a part of GA plan.

## Support Procedures

## Alternatives