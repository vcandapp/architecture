# Nova GPU Passthrough (VFIO) on DCN with CephHCI

This is a collection of CR templates that represent a Red Hat OpenStack Services on OpenShift deployment that has the following characteristics:

- Single Node OpenShift (SNO) cluster with external EDPM compute nodes
- 1-replica Galera database
- RabbitMQ
- OVN networking with spine-and-leaf topology (multi-subnet ready for DCN)
- Baremetal compute nodes with GPU passthrough via VFIO
- CephHCI installed on compute nodes and used by various OSP services
  - Glance using RBD for backend
  - Nova using RBD for ephemeral storage
- Nova configured for PCI device passthrough (NVIDIA GPUs)
  - Host kernel configured with IOMMU and vfio-pci driver
  - GPU PCI alias registered in Placement for scheduling
  - CPU pinning and tuned profile for performance isolation
- Metal3 bare metal provisioning with virtual media

## Considerations

1. These CRs are validated for the overall functionality of the OSP cloud deployed, but they nonetheless require customization for the particular environment in which they are utilized. In this sense they are _templates_ meant to be consumed and tweaked to fit the specific constraints of the hardware available.

2. The CRs are applied against an OpenShift cluster in _stages_. That is, there is an ordering in which each grouping of CRs is fed to the cluster. It is _not_ a case of simply taking all CRs from all stages and applying them all at once.

3. In stage 2 [kustomize](https://kustomize.io/) is used to generate the networking and control plane CRs dynamically. The `control-plane/networking/nncp/values.yaml`, `control-plane/networking/dns/values.yaml` and `control-plane/service-values.yaml` files must be updated to fit your environment. kustomize version 5 or newer required.

4. In stages 3 and 4 [kustomize](https://kustomize.io/) is used to generate the dataplane CRs dynamically. The `edpm-pre-ceph/nodeset/values.yaml`, `values.yaml` and `service-values.yaml` files must be updated to fit your environment. kustomize version 5 or newer required.

5. Between stages 3 and 4, _it is assumed that the user installs Ceph on the compute nodes._ OpenStack K8S CRDs do not provide a way to install Ceph via any sort of combination of CRs.

6. In stage 5 the `control-plane-post-ceph` kustomization needs the same network values used in stage 2 to preserve endpoint IPs, service types, and DNS configuration. For manual deployment, copy your environment-customized `control-plane/networking/nncp/values.yaml` into `control-plane-post-ceph/network-values.yaml` and populate `control-plane-post-ceph/values.yaml` with base64-encoded Ceph keyring and config. In CI, `ci_gen_kustomize_values` generates `network-values.yaml` in-place using the common Jinja2 template with environment overlays. See [control-plane-post-ceph.md](control-plane-post-ceph.md) for details.

7. On SNO with a single EDPM compute (single-host CephHCI), the Ceph ingress service (haproxy/keepalived) is not deployed. The default Swift endpoint (`<vip>:8080`) is unreachable because no ingress fronts the RGW daemon. Instead, clients must reach RGW directly on the compute's storage IP at port 8082 (the `rgw_frontend_port` set in the Ceph RGW spec).
   For CI automation, set `cifmw_cephadm_rgw_port: 8082` and `cifmw_cephadm_rgw_vip: <compute_storage_ip>` in the scenario vars so that `cifmw_cephadm` creates the Keystone endpoint with the correct address.
   For manual deployment, after installing Ceph, update the Swift endpoints in Keystone to point at the RGW daemon directly:

   ```shell
   STORAGE_IP=<compute storage network IP>
   for ep_id in $(openstack endpoint list --service object-store -f value -c ID); do
     openstack endpoint set --url "http://${STORAGE_IP}:8082/swift/v1/AUTH_%(tenant_id)s" "$ep_id"
   done
   ```

8. For CI automation, this DT uses `automation/vars/nova05epsilon.yaml` which maps the manual stages above to 10 granular automation steps (NNCP, networking, control-plane, DNS, baremetalhosts, pre-ceph nodeset, pre-ceph deployment, control-plane-post-ceph, post-ceph nodeset, post-ceph deployment).

## Host Configuration

The following parameters are crucial for host-level GPU passthrough configuration in `edpm-pre-ceph/nodeset/values.yaml`:

- **`edpm_kernel_args`**: Enables IOMMU and binds NVIDIA GPUs to the `vfio-pci` driver at boot time.
  - `intel_iommu=on iommu=pt`: Enables the IOMMU for device passthrough.
  - `vfio-pci.ids=10de:20f1`: Claims GPU(s) by vendor and product IDs.
  - `rd.driver.pre=vfio-pci`: Loads vfio-pci early to avoid race conditions.

- **`edpm_tuned_profile`** and **`edpm_tuned_isolated_cores`**: Configure CPU isolation for performance.

- **`vfio-pci-bind` service**: Blacklists `nouveau` and `nvidia` kernel modules and regenerates initramfs.

## Nova Configuration

A count of `X` PCI devices may be requested through `"pci_passthrough:alias"="nvidia_a2:X"` flavor extra specs:
```
openstack --os-compute-api=2.86 flavor set --property "pci_passthrough:alias"="nvidia_a2:1" device_passthrough
```

## Stages

All stages must be executed in the order listed below. Everything is required unless otherwise indicated.

1. [Install the OpenStack K8S operators and their dependencies](../../../common/)
2. [Configuring networking and deploy the OpenStack control plane](control-plane.md)
3. [Configure and deploy the initial data plane to prepare for Ceph installation](dataplane-pre-ceph.md)
4. Install Ceph on the compute nodes (without changing OpenStack CP CR)
5. [Update the control plane with Ceph backend configuration](control-plane-post-ceph.md)
6. [Finish deploying the data plane after Ceph has been installed](dataplane-post-ceph.md)

## Extending to a Full DCN Deployment

This topology deploys a single availability zone with computes on a
remote L3 subnet. The multi-subnet networking (subnet1 for SNO,
subnet2 for EDPM computes) provides the routed-networking foundation
for DCN, but several additional changes are required to make this a
true multi-AZ distributed compute deployment. Use
[dt/dcn](../../dcn/README.md) as a reference.

### Glance and Local Ceph Storage

In this topology, it is implied that Ceph runs on remote EDPM compute
nodes (CephHCI), but Glance runs on the control plane at another site.
This means Glance's RBD backend is accessing a Ceph cluster over the WAN,
which adds latency to every image operation. In a DCN deployment with
storage, at least one Ceph cluster must be local to the control plane
so that the central Glance instance has low-latency access to its default
RBD store. The DCN DT achieves this by designating az0 as the central site
with its own local Ceph cluster and configuring Glance az0 as type `split`
with `default_backend = az0`. Edge sites use Glance type `edge` and
replicate images from the central store on demand.

### Per-AZ Ceph Configuration

In `values.yaml`, switch from plain Ceph filenames (`ceph.conf`,
`ceph.client.openstack.keyring`) to az-prefixed filenames
(`az0.conf`, `az0.client.openstack.keyring`, `az1.conf`, etc.) so
that each site's Ceph cluster is independently addressable. Update
the Nova Ceph config (`nova.ceph.conf`) to reference the correct
az-prefixed conf file for each site.

### Per-AZ Glance

Replace the single Glance backend with per-AZ `glanceAPIs` entries
in `service-values.yaml`. The central site should be type `split`
with backends for all AZs; edge sites should be type `edge` with
backends for their own AZ plus the central AZ (for copy-on-write
image import).

### OVN Availability Zones

Set `edpm_ovn_availability_zones` in the nodeset values for each
site instead of leaving it empty. This enables OVN chassis-level AZ
awareness for network scheduling.

### Additional Networking Subnets

Add a subnet per site for each network (ctlplane, internalapi,
storage, tenant) in the network values. The current topology
defines subnet1 and subnet2; a full DCN deployment with N sites
needs N+1 subnets (one for the control plane site, one per edge
site).

### Per-Site Dataplane Stages

The dataplane stages (pre-ceph nodeset, pre-ceph deployment, Ceph
install, post-ceph nodeset, post-ceph deployment) must be repeated
for each DCN site with site-specific values. Each site's computes
should reference the appropriate cell, subnet, and AZ.
