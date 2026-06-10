# Update the control plane with Ceph backend configuration

## Assumptions

- The [pre-ceph data plane](dataplane-pre-ceph.md) has been deployed
- Ceph has been installed on the compute nodes
- The `ceph-conf-files` secret will be created during this step

## Initialize

Switch to the "openstack" namespace

```shell
oc project openstack
```

Change to the control-plane-post-ceph directory

```shell
cd architecture/examples/dt/nova/nova05epsilon/control-plane-post-ceph
```

## Prepare values files

The kustomization requires three values files:

**network-values.yaml** — Provides storageClass, bridgeName, endpoint
annotations, and DNS options that the lib components need. This file
must match the `network-values` used in the initial control-plane stage
to preserve all endpoint IPs, service types, and DNS configuration.
In CI, `ci_gen_kustomize_values` generates it from the automation vars
(`src_file: network-values.yaml`). For manual deployment, copy the
environment-customized file from the control-plane networking stage:

```shell
cp ../control-plane/networking/nncp/values.yaml network-values.yaml
```

Edit `network-values.yaml` and replace any remaining `CHANGEME`
placeholders to match your environment (same values used in stage 2).

**values.yaml** — Ceph configuration for the `ceph-conf-files` secret.
Replace the `CHANGEME` placeholders with base64-encoded Ceph keyring
and config from your Ceph deployment:

```shell
vi values.yaml
# Set data.ceph_conf."ceph.client.openstack.keyring" to:
#   base64 -w0 /etc/ceph/ceph.client.openstack.keyring
# Set data.ceph_conf."ceph.conf" to:
#   base64 -w0 /etc/ceph/ceph.conf
```

**service-values.yaml** — Edit if you need to adjust the Glance RBD
backend or Ceph extraMounts configuration:

```shell
vi service-values.yaml
```

## Update the control plane

Generate the control-plane-post-ceph CRs:

```shell
kustomize build > control-plane-post-ceph.yaml
```

Apply the CRs:

```shell
oc apply -f control-plane-post-ceph.yaml
```

Wait for the control plane to be ready:

```shell
oc wait osctlplane controlplane --for condition=Ready --timeout=1200s
```
