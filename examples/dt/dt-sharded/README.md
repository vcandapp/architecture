# DT-Sharded: Per-Service Galera, RabbitMQ, and Memcached Isolation

This is a collection of CR templates that represent a validated Red Hat OpenStack Services on OpenShift deployment that has the following characteristics:

- 3 master+worker combo OpenShift nodes
- Per-service dedicated Galera databases (replicas: 1)
- Per-service dedicated RabbitMQ clusters (replicas: 1)
- Per-service dedicated Memcached instances (replicas: 1)
- OVN networking
- Network isolation over a single NIC
- 2 compute nodes
- Glance with file backend (no Ceph)

## Sharding Architecture

This DT tests full infrastructure sharding where each OpenStack service gets its own dedicated database, message bus, and cache instance.

### Galera Databases

Each service gets a dedicated Galera cluster:

openstack-barbican, openstack-cinder, openstack-designate, openstack-glance, openstack-keystone, openstack-neutron, openstack-octavia, openstack-nova, openstack-nova-cell1, openstack-placement

### RabbitMQ Clusters

Per-service dedicated RabbitMQ clusters:

rabbitmq-barbican, rabbitmq-cinder, rabbitmq-designate, rabbitmq-glance, rabbitmq-keystone, rabbitmq-neutron, rabbitmq-octavia

Nova cells use the standard `rabbitmq` (cell0/notifications) and `rabbitmq-cell1` (cell1).

### Memcached Instances

Per-service dedicated Memcached instances for services that support `memcachedInstance`:

memcached-keystone, memcached-glance, memcached-cinder, memcached-neutron, memcached-horizon, memcached-nova

Services without `memcachedInstance` support (barbican, designate, octavia, placement) or disabled services (manila) are not included.

### Services Enabled

barbican, cinder, designate, glance, horizon, keystone, neutron, nova, octavia

## Assumptions

- A storage class called `local-storage` should already exist.

## Prerequisites

[Install the OpenStack K8S operators and their dependencies](https://github.com/openstack-k8s-operators/architecture/blob/main/examples/common)

## Initialize

Switch to the "openstack" namespace
```
oc project openstack
```
Change to the dt-sharded directory
```
cd architecture/examples/dt/dt-sharded
```
Edit the [control-plane/networking/nncp/values.yaml](control-plane/networking/nncp/values.yaml) and
[control-plane/service-values.yaml](control-plane/service-values.yaml) files to suit
your environment.
```
vi control-plane/networking/nncp/values.yaml
vi control-plane/service-values.yaml
```

## Apply node network configuration

Generate the node network configuration
```
kustomize build control-plane/networking/nncp > nncp.yaml
```
Apply the NNCP CRs
```
oc apply -f nncp.yaml
```
Wait for NNCPs to be available
```
oc wait nncp -l osp/nncm-config-type=standard --for jsonpath='{.status.conditions[0].reason}'=SuccessfullyConfigured --timeout=300s
```

## Apply networking configuration

Generate the networking CRs.
```
kustomize build control-plane/networking > network.yaml
```
Apply the CRs
```
oc apply -f network.yaml
```

Wait for MetalLB speaker pods to be ready
```
oc -n metallb-system wait pod -l app=metallb -l component=speaker --for condition=Ready --timeout=300s
```

## Apply control-plane configuration

Generate the control-plane CRs.
```
kustomize build control-plane > control-plane.yaml
```
Apply the CRs
```
oc apply -f control-plane.yaml
```

Wait for control plane to be available
```
oc wait osctlplane controlplane --for condition=Ready --timeout=60m
```

## Apply dataplane configuration

Generate the dataplane CRs.
```
kustomize build > edpm.yaml
```
Apply the CRs
```
oc apply -f edpm.yaml
```

Wait for the dataplanedeployment to reach the "Ready" condition
```
oc -n openstack wait openstackdataplanedeployment edpm-deployment --for condition=Ready --timeout=40m
```
