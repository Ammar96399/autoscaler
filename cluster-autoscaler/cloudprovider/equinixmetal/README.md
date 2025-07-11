<!--TODO: Remove "previously referred to as master" references from this doc once this terminology is fully removed from k8s-->
# Cluster Autoscaler for Equinix Metal

The cluster autoscaler for [Equinix Metal](https://metal.equinix.com) worker nodes performs
autoscaling within any specified nodepools. It will run as a `Deployment` in
your cluster. The nodepools are specified using tags on Equinix Metal.

Note: Packet was acquired by Equinix in 2020 and renamed to Equinix Metal.

This README will go over some of the necessary steps required to get
the cluster autoscaler up and running.

## Permissions and credentials

The autoscaler needs a `ServiceAccount` with permissions for Kubernetes and
requires credentials for interacting with Equinix Metal.

An example `ServiceAccount` is given in [examples/cluster-autoscaler-svcaccount.yaml](examples/cluster-autoscaler-svcaccount.yaml).

The credentials for authenticating with Equinix Metal are stored in a secret and
provided as an env var to the container. [examples/cluster-autoscaler-secret](examples/cluster-autoscaler-secret.yaml)
In the above file you can modify the following fields:

| Secret                              | Key                     | Value                        |
|-------------------------------------|-------------------------|------------------------------------------------------------------------------------------------------------------------------------|
| cluster-autoscaler-equinixmetal | authtoken               | Your Equinix Metal API token. It must be base64 encoded.                                                                                 |
| cluster-autoscaler-cloud-config     | Global/project-id       | Your Equinix Metal project id                                                                                                             |
| cluster-autoscaler-cloud-config     | Global/api-server       | The ip:port for you cluster's k8s api (e.g. K8S_MASTER_PUBLIC_IP:6443)                                                             |
| cluster-autoscaler-cloud-config     | Global/facility         | The Equinix Metal facility for the devices in your nodepool (eg: sv15)                                                                    |
| cluster-autoscaler-cloud-config     | Global/plan             | The Equinix Metal plan (aka size/flavor) for new nodes in the nodepool (eg: c3.small.x86)                                                 |
| cluster-autoscaler-cloud-config     | Global/billing          | The billing interval for new nodes (default: hourly)                                                                               |
| cluster-autoscaler-cloud-config     | Global/os               | The OS image to use for new nodes (default: ubuntu_18_04). If you change this also update cloudinit.                               |
| cluster-autoscaler-cloud-config     | Global/cloudinit        | The base64 encoded [user data](https://metal.equinix.com/developers/docs/servers/user-data/) submitted when provisioning devices. In the example file, the default value has been tested with Ubuntu 18.04 to install Docker & kubelet and then to bootstrap the node into the cluster using kubeadm. The kubeadm, kubelet, sudo k0s kubectl are pinned to version 1.17.4. For a different base OS or bootstrap method, this needs to be customized accordingly|
| cluster-autoscaler-cloud-config     | Global/reservation      | The values "require" or "prefer" will request the next available hardware reservation for new devices in selected facility & plan. If no hardware reservations match, "require" will trigger a failure, while "prefer" will launch on-demand devices instead (default: none)  |
| cluster-autoscaler-cloud-config     | Global/hostname-pattern | The pattern for the names of new Equinix Metal devices (default: "k8s-{{.ClusterName}}-{{.NodeGroup}}-{{.RandString8}}" )                  |

You can always update the secret with more nodepool definitions (with different plans etc.) as shown in the example, but you should always provide a default nodepool configuration.

## Configure nodepool and cluster names using Equinix Metal tags

The Equinix Metal API does not yet have native support for groups or pools of devices. So we use tags to specify them. Each Equinix Metal device that's a member of the "cluster1" cluster should have the tag k8s-cluster-cluster1. The devices that are members of the "pool1" nodepool should also have the tag k8s-nodepool-pool1. Once you have a Kubernetes cluster running on Equinix Metal, use the Equinix Metal Portal or API to tag the nodes accordingly.

## Autoscaler deployment

The deployment in `examples/cluster-autoscaler-deployment.yaml` can be used,
but the arguments passed to the autoscaler will need to be changed
to match your cluster.

| Argument              | Usage                                                                                                      |
|-----------------------|------------------------------------------------------------------------------------------------------------|
| --cluster-name        | The name of your Kubernetes cluster. It should correspond to the tags that have been applied to the nodes. |
| --nodes               | Of the form `min:max:NodepoolName`. For multiple nodepools you can add the same argument multiple times. E.g. for pool1, pool2 you would add `--nodes=0:10:pool1` and `--nodes=0:10:pool2`. In addition, each node provisioned by the autoscaler will have a label with key: `pool` and with value: `NodepoolName`. These labels can be useful when there is a need to target specific nodepools. |
| --expander=price      |  This is an optional argument which allows the cluster-autoscaler to take into account the pricing of the Equinix Metal nodes when scaling with multiple nodepools. |

## Target Specific Nodepools (New!)

In case you want to target a specific nodepool(s) for e.g. a deployment, you can add a `nodeAffinity` with the key `pool` and with value the nodepool name that you want to target. This functionality is not backwards compatible, which means that nodes provisioned with older cluster-autoscaler images won't have the key `pool`. But you can overcome this limitation by manually adding the correct labels. Here are some examples:

Target a nodepool with a specific name:
```
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: pool
          operator: In
          values:
          - pool3
```
Target a nodepool with a specific Equinix Metal instance:
```
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: beta.kubernetes.io/instance-type
          operator: In
          values:
          - c3.small.x86
```

## CCM and Controller node labels

### CCM
By default, autoscaler assumes that you have an older deprecated version of `packet-ccm` installed in your
cluster.  If however, that is not the case and you've migrated to the new `cloud-provider-equinix-metal` CCM,
then this must be told to autoscaler. This can be done via setting an environment variable in the deployment:
```
env:
  - name: INSTALLED_CCM
    value: cloud-provider-equinix-metal
```
**NOTE**: As a prerequisite, ensure that all worker nodes in your cluster have the prefix `equinixmetal://` in
the Node spec `.spec.providerID`. If there are any existing worker nodes with prefix `packet://`, then drain
the node, remove the node and restart the kubelet on that worker node to re-register the node in the cluster,
this would ensure that `cloud-provider-equinix-metal` CCM sets the uuid with prefix `equinixmetal://` to the
field `.spec.ProviderID`.

### Controller node labels

Autoscaler assumes that control plane nodes in your cluster are identified by the label
`node-role.kubernetes.io/master`. If for some reason, this assumption is not true in your case, then set the
environment variable in the deployment:

```
env:
  - name: METAL_CONTROLLER_NODE_IDENTIFIER_LABEL
    value: <label>
```

## Notes

The autoscaler will not remove nodes which have non-default kube-system pods.
This prevents the node that the autoscaler is running on from being scaled down.
If you are deploying the autoscaler into a cluster which already has more than one node,
it is best to deploy it onto any node which already has non-default kube-system pods,
to minimise the number of nodes which cannot be removed when scaling. For this reason in
the provided example the autoscaler pod has a nodeaffinity which forces it to deploy on
the control plane (previously referred to as master) node.

### Changes

1. It is now possible to use multiple nodepools, scale nodepools to 0 nodes and prioritize scaling of specific nodepools by taking into account the pricing of the Equinix Metal instances.

2. In order to take advantage of the new features mentioned above, you might need to update the cloud-config and the autoscaler deployment as shown in the examples. For example, the default/global cloud-config is applied to all the nodepools and if you want to override it for a specific nodepool you have to modify the cloud-config according to the examples.

3. You can target specific nodepools, as described above.

4. Cloud inits in the examples have pinned versions for Kubernetes in order to minimize potential incompatibilities as a result of nodes provisioned with different Kubernetes versions.

5. In the provided cluster-autoscaler deployment example, the autoscaler pod has a nodeaffinity which forces it to deploy on the control plane (previously referred to as master) node, so that the cluster-autoscaler can scale down all of the worker nodes. Without this change there was a possibility for the cluster-autoscaler to be deployed on a worker node that could not be downscaled.
