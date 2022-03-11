---
title: Istio CA Integration with SPIRE 
linktitle: How to integrate Istio with SPIRE through Envoy SDS API.
description: Describes how to configure Istio to integrate with SPIRE to get cryptographic identities through Envoy's SDS API.
weight: 40
keywords: [kubernetes,spiffe,spire]
aliases:
owner: istio/wg-networking-maintainers
test: no
---

[SPIRE](https://spiffe.io/docs/latest/spire-about/spire-concepts/) is a production-ready implementation of the SPIFFE specification that performs node and workload attestation in order to securely issue cryptographic identities to workloads running in heterogeneous environments.  
In an Istio deployment, SPIRE can be configured as a source of cryptographic identities for Istio workloads through the integration with [Envoy's SDS API](https://www.envoyproxy.io/docs/envoy/latest/configuration/security/secret).
Istio can detect the existence of a UNIX domain socket that implements the Envoy SDS API on a defined socket path, allowing Envoy to communicate and 
fetch identities directly from it.

## Install SPIRE

### Option 1: Quick start

Istio provides a basic sample installation to quickly get SPIRE up and running:

{{< text bash >}}
$ kubectl apply -f @samples/security/envoy-sds/spire/spire-quickstart.yaml
{{< /text >}}

This will deploy SPIRE into your cluster, along with two additional components: the [SPIFFE CSI Driver](https://github.com/spiffe/spiffe-csi) — used to share the SPIRE Agent's UNIX domain socket with the other
pods throughout the node — and the [SPIRE Kubernetes Workload Registrar](https://github.com/spiffe/spire/tree/main/support/k8s/k8s-workload-registrar), a facilitator that performs automatic workload registration
within Kubernetes. See [Install Istio](#install-istio) to configure Istio and integrate with the SPIFFE CSI Driver.

{{< warning >}}
Note that some configuration used here may not be fully appropriate for a production environment. 

Please see [Scaling SPIRE](https://spiffe.io/docs/latest/planning/scaling_spire/) for more information on configuring SPIRE for a production environment.
{{< /warning >}}

### Option 2: Configure custom SPIRE installations

See the [SPIRE's Quickstart for Kubernetes guide](https://spiffe.io/docs/latest/try/getting-started-k8s/) 
to get started deploying SPIRE into your Kubernetes environment. See [Integration Prerequisites](#integration-prerequisites)
for more information on configuring SPIRE to integrate with Istio deployments.

#### CA Integration Prerequisites

These configuration requirements are necessary to successfully integrate Istio with SPIRE:

1. Access [SPIRE Agent reference](https://spiffe.io/docs/latest/deploying/spire_agent/#agent-configuration-file) and
   configure the SPIRE Agent socket path to match Envoy SDS defined socket path.

   {{< text >}}
   socket_path = "/run/secrets/workload-spiffe-uds/socket"
   {{< /text >}}

2. Share the SPIRE Agent socket with the pods within the node by leveraging the
   [SPIFFE CSI Driver](https://github.com/spiffe/spiffe-csi).

3. See [Install Istio](#install-istio) to configure Istio to integrate with the SPIFFE CSI Driver.

## Install Istio

1. [Download the latest Istio release](https://istio.io/latest/docs/setup/getting-started/#download).
to access the SPIRE Agent socket shared by the [SPIFFE CSI Driver](https://github.com/spiffe/spiffe-csi). 

2. After successfully [deploying SPIRE](#install-spire) into your environment, install Istio with custom patches for istio-ingressgateways/egressgateways as well as the istio-proxy.

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
           label:
             istio: ingressgateway
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
                       mountPath: "/run/secrets/workload-spiffe-uds"
                       readOnly: true
   EOF
   {{< /text >}}

3. Use [Automatic sidecar injection](https://istio.io/latest/docs/setup/additional-setup/sidecar-injection/#automatic-sidecar-injection)
to automatically inject the spire template in the configuration above, into new workloads pods.

This will share the spiffe-csi-driver with the Ingress Gateway and the sidecars that are going
to be automatically injected on workload pods, granting them access to the SPIRE Agent's UNIX Domain Socket.

* Check istio-ingressgateway pod state:

{{< text bash >}}
$ kubectl get pods -n istio-system
NAME                                    READY   STATUS    RESTARTS   AGE
istio-ingressgateway-5b45864fd4-lgrxs   0/1     Running   0          17s
istiod-989f54d9c-sg7sn                  1/1     Running   0          23s
{{< /text >}}

Data plane containers will only reach the `Ready` if a corresponding registration entry is created for them on the SPIRE Server. Then, Envoy will be able
to fetch cryptographic identities from SPIRE.
See [Registering Workloads](#registering-workloads) to register entries for services in your mesh.

## Registering Workloads

This section describes the options available for registering workloads in SPIRE Server.

### Option 1: Automatic registration using the SPIRE workload registrar

By deploying [SPIRE Kubernetes Workload Registrar](https://github.com/spiffe/spire/tree/main/support/k8s/k8s-workload-registrar)
along with SPIRE Server so new entries are automatically registered for each new pod that is created.

See [Verifying that identities were created for workloads](#verifying-that-identities-were-created-for-workloads) 
to check issued identities.

### Option 2: Manual Registration

SPIRE Server is able to verify a group of values to improve workload attestation security
robustness.

1. To generate an entry for an ingress-gateway with a set of selectors such as the
   pod name and pod UID:

{{< text bash >}}
$ INGRESS_POD=$(kubectl get pod -l istio=ingressgateway -n istio-system -o jsonpath="{.items[0].metadata.name}")
$ INGRESS_POD_UID=$(kubectl get pods -n istio-system $INGRESS_POD -o jsonpath='{.metadata.uid}')
{{< /text >}}

2. Get the spire-server pod:

{{< text bash >}}
$ SPIRE_SERVER_POD=$(kubectl get pod -l app=spire-server -n spire -o jsonpath="{.items[0].metadata.name}")
{{< /text >}}

3. Attest the SPIRE Agent running on the node
   {{< text bash >}}
   $ kubectl exec -n spire $SPIRE_SERVER_POD -- \
   /opt/spire/bin/spire-server entry create \
   -spiffeID spiffe://example.org/ns/spire/sa/spire-agent \
   -selector k8s_psat:cluster:demo-cluster \
   -selector k8s_psat:agent_ns:spire \
   -selector k8s_psat:agent_sa:spire-agent \
   -node -socketPath /run/spire/sockets/server.sock
   
   Entry ID         : d38c88d0-7d7a-4957-933c-361a0a3b039c
   SPIFFE ID        : spiffe://example.org/ns/spire/sa/spire-agent
   Parent ID        : spiffe://example.org/spire/server
   Revision         : 0
   TTL              : default
   Selector         : k8s_psat:agent_ns:spire
   Selector         : k8s_psat:agent_sa:spire-agent
   Selector         : k8s_psat:cluster:demo-cluster
   {{</ text >}}

4. Register an entry for the istio-ingressgateway pod.

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

   Entry ID         : 6f2fe370-5261-4361-ac36-10aae8d91ff7
   SPIFFE ID        : spiffe://example.org/ns/istio-system/sa/istio-ingressgateway-service-account
   Parent ID        : spiffe://example.org/ns/spire/sa/spire-agent
   Revision         : 0
   TTL              : default
   Selector         : k8s:ns:istio-system
   Selector         : k8s:pod-uid:63c2bbf5-a8b1-4b1f-ad64-f62ad2a69807
   Selector         : k8s:sa:istio-ingressgateway-service-account
   DNS name         : istio-ingressgateway.istio-system.svc
   DNS name         : istio-ingressgateway-c48554dd6-cff5z
   {{</ text >}}

5. Deploy an example workload:

{{< text bash >}}
$ istioctl kube-inject --filename @samples/httpbin/httpbin.yaml | kubectl apply -f -
{{< /text >}}

Note that the workload will need the SPIFFE CSI Driver volume to access the SPIRE Agent socket, you can leverage the spire template pod annotation
from [Install Istio](#install-istio) section or add the csi volume to the deployment spec of your workload.

6. Get pod information:

{{< text bash >}}
$ HTTPBIN_POD=$(kubectl get pod -l app=httpbin -o jsonpath="{.items[0].metadata.name}")
$ HTTPBIN_POD_UID=$(kubectl get pods $HTTPBIN_POD -o jsonpath='{.metadata.uid}')
{{< /text >}}

7. Register workload.

   {{< text bash >}}
   $ kubectl exec -n spire spire-server-0 -- \
   /opt/spire/bin/spire-server entry create \
   -spiffeID spiffe://example.org/ns/default/sa/httpbin \
   -parentID spiffe://example.org/ns/spire/sa/spire-agent \
   -selector k8s:ns:default \
   -selector k8s:pod-uid:$HTTPBIN_POD_UID \
   -dns $HTTPBIN_POD \
   -socketPath /run/spire/sockets/server.sock
   {{< /text >}}

{{< warning >}}
SPIFFE IDs for workloads must follow the Istio SPIFFE ID pattern: `spiffe://<trust.domain>/ns/<namespace>/sa/<service-account>`

{{< /warning >}}

See the [SPIRE help on Registering workloads](https://spiffe.io/docs/latest/deploying/registering/) to learn how to create new entries for workloads and get them attested using multiple selectors to strengthen attestation criteria.

## Verifying that identities were created for workloads

Confirm that identities were created for the workloads.
   {{< text bash >}}
   $ kubectl exec -i -t $SPIRE_SERVER_POD -n spire -c spire-server -- /bin/sh -c "bin/spire-server entry show -socketPath /run/spire/sockets/server.sock"
   Found 3 entries
   Entry ID         : c8dfccdc-9762-4762-80d3-5434e5388ae7
   SPIFFE ID        : spiffe://example.org/ns/istio-system/sa/istio-ingressgateway-service-account
   Parent ID        : spiffe://example.org/ns/spire/sa/spire-agent
   Revision         : 0
   TTL              : default
   Selector         : k8s:ns:istio-system
   Selector         : k8s:pod-uid:88b71387-4641-4d9c-9a89-989c88f7509d
   Selector         : k8s:sa:istio-ingressgateway-service-account
   DNS name         : istio-ingressgateway-c48554dd6-cff5z

   Entry ID         : af7b53dc-4cc9-40d3-aaeb-08abbddd8e54
   SPIFFE ID        : spiffe://example.org/ns/default/sa/httpbin
   Parent ID        : spiffe://example.org/ns/spire/sa/spire-agent
   Revision         : 0
   TTL              : default
   Selector         : k8s:ns:default
   Selector         : k8s:pod-uid:ee490447-e502-46bd-8532-5a746b0871d6
   DNS name         : httpbin-5f4d47c948-njvpk

   Entry ID         : f0544fd7-1945-4bd1-88dc-0a5513fdae1c
   SPIFFE ID        : spiffe://example.org/ns/spire/sa/spire-agent
   Parent ID        : spiffe://example.org/spire/server
   Revision         : 0
   TTL              : default
   Selector         : k8s_psat:agent_ns:spire
   Selector         : k8s_psat:agent_sa:spire-agent
   Selector         : k8s_psat:cluster:demo-cluster
   {{< /text >}}

* Check istio-ingressgateway pod state:

{{< text bash >}}
$ kubectl get pods -n istio-system
NAME                                    READY   STATUS    RESTARTS   AGE
istio-ingressgateway-5b45864fd4-lgrxs   1/1     Running   0          60s
istiod-989f54d9c-sg7sn                  1/1     Running   0          45s
{{< /text >}}

After registering an entry for istio-ingressgateway pod, Envoy receives the identity issued by SPIRE and uses it for all TLS and mTLS 
communications.

### Check Certificate

* Get httpbin service address:

{{< text bash >}}
$ kubectl get service httpbin
NAME      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
httpbin   ClusterIP   10.96.55.188   <none>        8000/TCP   40s
{{< /text >}}

* Verify that the httpbin certificate was issued by SPIRE.

{{< text bash >}}
$ kubectl exec $INGRESS_POD -n istio-system -- bash -c "openssl s_client -showcerts -connect 10.96.55.188:8000"
{{< /text >}}

   {{< text >}}
   ...
    ---
    Certificate chain
    0 s:C = US, O = SPIRE, CN = httpbin-5f4d47c948-zm6st
    i:C = US, O = SPIFFE
    -----BEGIN CERTIFICATE-----
    MIIC8DCCAdigAwIBAgIRANo+nhRtC6kTnbLUz1MelwcwDQYJKoZIhvcNAQELBQAw
    HjELMAkGA1UEBhMCVVMxDzANBgNVBAoTBlNQSUZGRTAeFw0yMjAzMTAwMjMyMTVa
    Fw0yMjAzMTAwMzMyMjVaMEAxCzAJBgNVBAYTAlVTMQ4wDAYDVQQKEwVTUElSRTEh
    MB8GA1UEAxMYaHR0cGJpbi01ZjRkNDdjOTQ4LXptNnN0MFkwEwYHKoZIzj0CAQYI
    KoZIzj0DAQcDQgAEMveAwlgnO7YjRZ4vKsAkCYZu3W0REs1nKu/XPEWEBzfht3hl
    grziL3mLnECWeDnTdakjTnowESCsg3cidOpWEaOB0TCBzjAOBgNVHQ8BAf8EBAMC
    A6gwHQYDVR0lBBYwFAYIKwYBBQUHAwEGCCsGAQUFBwMCMAwGA1UdEwEB/wQCMAAw
    HQYDVR0OBBYEFHnxGd7U2ADDw4GouBb0wPLorrHeMB8GA1UdIwQYMBaAFLnNFhHO
    0Xsdte9uf3VLILKA7TGtME8GA1UdEQRIMEaCGGh0dHBiaW4tNWY0ZDQ3Yzk0OC16
    bTZzdIYqc3BpZmZlOi8vZXhhbXBsZS5vcmcvbnMvZGVmYXVsdC9zYS9odHRwYmlu
    MA0GCSqGSIb3DQEBCwUAA4IBAQA6UZagZrw/htbpD7HbcmD17sQ2xv1+nMrRFZGZ
    M9bO924uyz9MiEf/iR3jKD+wXrJdmyjJ+5Mas4QGnggaGUfDmE2wrVlFKIqY8R1E
    hZonQm8YeWN4CtDijyAxHihgKTV2V2inrHaaqp1r2/Upt3/4XrZf1f1eqpl03hT9
    +OPu0SV8MUPTPAXqaIPfjmV9fTTMYQg4g4smsijVKAl6lVs+PsZiqbcgEeUnT2JL
    lel5nNXdXqKhygfqKesI30BERGPvZ5vdMtUrGxe0L5CPJrkcrtn/vRTuMsXWSr4m
    VAw3TEEXVdnd3VuAIsKO+6J7biMZH6jhm/sPUqTrdjH/L7S7
    -----END CERTIFICATE-----
    ---
    Server certificate
    subject=C = US, O = SPIRE, CN = httpbin-5f4d47c948-zm6st
    
    issuer=C = US, O = SPIFFE
   ...
   {{< /text >}}

## SPIFFE Federation

SPIRE Servers are able to authenticate SPIFFE identities originated from different trust domains, this is known as SPIFFE federation.
SPIRE Agent can be configured to push federated bundles to Envoy through the Envoy SDS API, allowing Envoy to use [validation context](https://spiffe.io/docs/latest/microservices/envoy/#validation-context)
to verify peer certificates and trust a workload from another trust domain.   
To enable Istio to federate SPIFFE identities through SPIRE integration:

* Consult [SPIRE Agent SDS configuration](https://github.com/spiffe/spire/blob/main/doc/spire_agent.md#sds-configuration) and set the following 
SDS configuration values for your SPIRE Agent configuration file.

| Configuration              | Description                                                                                      | Resource Name |
|----------------------------|--------------------------------------------------------------------------------------------------|---------------|
| `default_svid_name`        | The TLS Certificate resource name to use for the default X509-SVID with Envoy SDS                | default       |
| `default_bundle_name`      | The Validation Context resource name to use for the default X.509 bundle with Envoy SDS          | null          |
| `default_all_bundles_name` | The Validation Context resource name to use for all bundles (including federated) with Envoy SDS | ROOTCA        |

This will allow Envoy to get federated bundles directly from SPIRE.

### Create Federated Registration Entries

* If using the SPIRE Kubernetes Workload Registrar, create federated entries for workloads by  
adding the pod annotation `spiffe.io/federatesWith` to the service deployment spec, specifying the trust domain you want the pod to federate with:

   {{< text  >}}
   podAnnotations:
     spiffe.io/federatesWith: "<trust.domain>"
   {{< /text >}}

* For manual registration see [Create Registration Entries for Federation](https://spiffe.io/docs/latest/architecture/federation/readme/#create-registration-entries-for-federation).

## Cleanup SPIRE

* If you installed SPIRE using the quick start SPIRE deployment provided by Istio, use
the following commands to remove those Kubernetes resources:

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
