---
title: Spire SDS Integration
linktitle: How integrate with Spire CA.
description: Describes how to configure Istio to integrate with Spire CA via Secret Discovery Service.
weight: 40
keywords: [kubernetes,spiffe,spire]
aliases:
owner: istio/wg-networking-maintainers
test: no
---

[Spire](/docs/ops/integrations/spire/)

SPIRE is a production-ready implementation of the SPIFFE APIs that performs node
and workload attestation in order to securely issue SVIDs to workloads, and verify
the SVIDs of other workloads, based on a predefined set of conditions. For more information,
access [SPIRE](https://spiffe.io/docs/latest/spire-about/spire-concepts/).

## Install Spire

### Option 1: Quick start

Istio provides a basic sample installation to quickly get Spire up and running:

{{< text bash >}}
$ kubectl apply -f /home/alexandre/Goland/fork/ale/istio/samples/security/envoy-sds/spire/spire-quickstart.yaml
{{< /text >}}

This will deploy Spire into your cluster, along with [SPIFFE CSI Driver](https://github.com/spiffe/spiffe-csi) used to share the Spire Agent UDS socket with other pods throughout
the node, as well as the [SPIRE Kubernetes Workload Registrar](https://github.com/spiffe/spire/tree/main/support/k8s/k8s-workload-registrar) that facilitates automatic
workload registration within Kubernetes. See [Install Istio](#install-istio) to configure Istio to integrate with the SPIFFE CSI Driver.

### Option 2: Customizable Spire install

Consult the [Spire Installation guide](https://spiffe.io/docs/latest/try/getting-started-k8s/) to get started deploying Spire into
your Kubernetes environment. See [Integration Prerequisites](#integration-prerequisites) for more information on configuring Spire to integrate with Istio deployments.

## Integration Prerequisites

There are a couple of necessary configuration requirements to successfully integrate Istio with Spire CA:

1. Access [Spire Agent reference](https://spiffe.io/docs/latest/deploying/spire_agent/#agent-configuration-file) and
   configure the Spire Agent socket path to match Envoy SDS path.

   {{< text >}}
   socket_path = "/run/secrets/workload-spiffe-uds/socket"
   {{< /text >}}

2. Share the Spire Agent socket with the pods within the node by leveraging the
   [SPIFFE CSI Driver](https://github.com/spiffe/spiffe-csi).

3. See [Install Istio](#install-istio) to configure Istio to integrate with the SPIFFE CSI Driver.

## Install Istio

[Download the latest Istio release](https://istio.io/latest/docs/setup/getting-started/#download).
After deploying successfully Spire into your environment, install Istio with custom patches for istio-ingressgateways/egressgateways as well as the istio-proxy
to access the Spire Agent socket shared by the [SPIFFE CSI Driver](https://github.com/spiffe/spiffe-csi). Use
[Automatic sidecar injection](https://istio.io/latest/docs/setup/additional-setup/sidecar-injection/#automatic-sidecar-injection)
to inject automatically the spire template below into new workloads pods.

{{< text bash >}}
$ istioctl install --skip-confirmation -f - <<EOF
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
namespace: istio-system
spec:
profile: default
meshConfig:
trustDomain: example.org
values:
global:
# This is used to customize the sidecar template
sidecarInjectorWebhook:
templates:
spire: |
spec:
containers:
- name: istio-proxy
volumeMounts:
- name: workload-socket
mountPath: /run/secrets/workload-spiffe-uds
readOnly: true
volumes:
- name: workload-socket
csi:
driver: "csi.spiffe.io"
components:
ingressGateways:
- name: istio-ingressgateway
enabled: true
k8s:
overlays:
- apiVersion: apps/v1
kind: Deployment
name: istio-ingressgateway
patches:
- path: spec.template.spec.volumes.[name:workload-socket]
value:
name: workload-socket
csi:
driver: "csi.spiffe.io"
- path: spec.template.spec.containers.[name:istio-proxy].volumeMounts.[name:workload-socket]
value:
name: workload-socket
mountPath: /run/secrets/workload-spiffe-uds
readOnly: true
EOF
{{< /text >}}

//TODO Explain in more details about what this configuration deploys (patch ingress/egress and create the spire template for the sidecar injection)
This will deploy Istio control plane along with an Ingressgateway

## Registering Workloads

Registering workloads with SPIFFE IDs in the SPIRE Server.

### Option 1: Automatically

By deploying [SPIRE Kubernetes Workload Registrar](https://github.com/spiffe/spire/tree/main/support/k8s/k8s-workload-registrar)
along with Spire Server to register new entries automatically for each new pod that is created.

### Option 2: Manually

Spire Server is able to verify a group of values to improve workload attestation security
robustness.

1. To generate an entry for an ingress-gateway with a set of selectors for example, get the
   pod name and pod-uid:

{{< text bash >}}
$ INGRESS_POD=$(kubectl get pod -l istio=ingressgateway -n istio-system -o jsonpath="{.items[0].metadata.name}")
$ INGRESS_POD_UID=$(kubectl get pods -n istio-system $INGRESS_POD -o jsonpath='{.metadata.uid}')
{{< /text >}}

2. Get the spire-server pod:

{{< text bash >}}
$ SPIRE_SERVER_POD=$(kubectl get pod -l app=spire-server -n spire -o jsonpath="{.items[0].metadata.name}")
{{< /text >}}

3. Attest the Spire Agent running on the node
   //TODO add command output
   {{< text bash >}}
   $ kubectl exec -n spire $SPIRE_SERVER_POD -- \
   /opt/spire/bin/spire-server entry create \
   -spiffeID spiffe://example.org/ns/spire/sa/spire-agent \
   -selector k8s_psat:cluster:demo-cluster \
   -selector k8s_psat:agent_ns:spire \
   -selector k8s_psat:agent_sa:spire-agent \
   -node -socketPath /run/spire/sockets/server.sock
   {{< /text >}}

4. and then register an entry for the pod:
   //TODO add command output
   //TODO Explain the necessity for creating the entries with the Istio ns and sa pattern
   {{< text bash >}}
   $ kubectl exec -n spire $SPIRE_SERVER_POD -- \
   /opt/spire/bin/spire-server entry create \
   -spiffeID spiffe://example.org/ns/istio-system/sa/istio-ingressgateway-service-account \
   -parentID spiffe://example.org/ns/spire/sa/spire-agent \
   -selector k8s:sa:istio-ingressgateway-service-account \
   -selector k8s:ns:istio-system \
   -selector k8s:pod-uid:$INGRESS_POD_UID \
   -dns $INGRESS_POD \
   -dns istio-ingressgateway.istio-system.svc \
   -socketPath /run/spire/sockets/server.sock
   {{< /text >}}


Consult [Registering workloads](https://spiffe.io/docs/latest/deploying/registering/) on how
to create new entries for workloads and attest a set of multiples selectors for securing services.

## Verifying Entries

Confirm that identities were created for the workloads.
//TODO add command output
{{< text bash >}}
$ kubectl exec -i -t $SPIRE_SERVER_POD -n spire -c spire-server -- /bin/sh -c "bin/spire-server entry show -socketPath /run/spire/sockets/server.sock"
{{< /text >}}

## Spire Federation

//TODO
## Cleanup Spire

* Remove created Kubernetes resources:

{{< text bash >}}
$ kubectl delete CustomResourceDefinition spiffeids.spiffeid.spiffe.io
$ kubectl delete -n spire serviceaccount spire-agent
$ kubectl delete -n spire configmap spire-agent
$ kubectl delete -n spire deployment spire-agent
$ kubectl delete csidriver csi.spiffe.io
$ kubectl delete -n spire configmap spire-server
$ kubectl delete -n spire service spire-server
$ kubectl delete -n spire serviceaccount spire-server
$ kubectl delete -n spire statefulset spire-server
$ kubectl delete clusterrole spire-server-trust-role spire-agent-cluster-role
$ kubectl delete clusterrolebinding spire-server-trust-role-binding spire-agent-cluster-role-binding
$ kubectl delete namespace spire
{{< /text >}}
