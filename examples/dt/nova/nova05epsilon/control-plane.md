# Configuring networking and deploy the OpenStack control plane

## Assumptions

- A storage class called `lvms-local-storage` should already exist.

## Initialize

Switch to the "openstack" namespace

```shell
oc project openstack
```

Change to the nova05epsilon directory

```shell
cd architecture/examples/dt/nova/nova05epsilon/control-plane
```

Edit the [networking/nncp/values.yaml](control-plane/networking/nncp/values.yaml),
[networking/dns/values.yaml](control-plane/networking/dns/values.yaml) and
[service-values.yaml](control-plane/service-values.yaml) files to suit
your environment.

```shell
vi networking/nncp/values.yaml
vi networking/dns/values.yaml
vi service-values.yaml
```

## Apply node network configuration

Generate the node network configuration

```shell
kustomize build networking/nncp > nncp.yaml
```

Apply the NNCP CRs

```shell
oc apply -f nncp.yaml
```

Wait for NNCPs to be available

```shell
oc wait nncp -l osp/nncm-config-type=standard --for jsonpath='{.status.conditions[0].reason}'=SuccessfullyConfigured --timeout=300s
```

## Apply networking configuration

Generate the networking CRs (MetalLB, NetConfig, NetworkAttachmentDefinitions).

```shell
kustomize build networking > networking.yaml
```

Apply the CRs

```shell
oc apply -f networking.yaml
```

Wait for MetalLB to be ready

```shell
oc -n metallb-system wait pod -l app=metallb -l component=speaker --for condition=Ready --timeout=300s
```

## Apply control-plane configuration

Generate the control-plane CRs.

```shell
kustomize build > control-plane.yaml
```

Apply the CRs

```shell
oc apply -f control-plane.yaml
```

Wait for the control plane DNS to be ready (the full `condition=Ready`
is deferred until after the DNS operator configuration is applied, so that
the two stages can proceed without waiting for all OpenStack services):

```shell
oc wait osctlplane controlplane --for condition=OpenStackControlPlaneDNSReadyCondition --timeout=1200s
```

## Apply Openshift DNS configuration for ctlplane DNS zone

Generate the `dns.operator/default` CR to update the ctlplane resolver
for the DNSMasq instance created during control-plane configuration.

```shell
kustomize build networking/dns > dns.yaml
```

Apply the CRs

```shell
oc apply -f dns.yaml
```

Wait for DNS to be available

```shell
oc -n openshift-dns wait dns.operator/default --for condition=Available --timeout=300s
```

Now wait for the full control plane readiness before proceeding to
the data plane stages:

```shell
oc wait osctlplane controlplane --for condition=Ready --timeout=600s
```
