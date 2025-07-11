# Quick start

## Contents

- [Quick start](#quick-start)
  - [Test your installation](#test-your-installation)
  - [Example VPA configuration](#example-vpa-configuration)
  - [Troubleshooting](#troubleshooting)

After [installation](./installation.md) the system is ready to recommend and set
resource requests for your pods.
In order to use it, you need to insert a *Vertical Pod Autoscaler* resource for
each controller that you want to have automatically computed resource requirements.
This will be most commonly a **Deployment**.
There are five modes in which *VPAs* operate:

- `"Auto"`: VPA assigns resource requests on pod creation as well as updates
  them on existing pods using the preferred update mechanism. Currently, this is
  equivalent to `"Recreate"` (see below).
- `"Recreate"`: VPA assigns resource requests on pod creation as well as updates
  them on existing pods by evicting them when the requested resources differ significantly
  from the new recommendation (respecting the Pod Disruption Budget, if defined).
  This mode should be used rarely, only if you need to ensure that the pods are restarted
  whenever the resource request changes.
- `"InPlaceOrRecreate"`[__alpha feature__]: VPA assigns resource requests on pod creation as well as updates
  them on existing pods by leveraging [Kubernetes `in-place` update](https://kubernetes.io/blog/2025/05/16/kubernetes-v1-33-in-place-pod-resize-beta/) capability.
  If `in-place` update fails, it falls back to evicting the pods, performing a _recreation_.
  For more details, see the [In-Place Updates documentation](https://github.com/kubernetes/autoscaler/blob/master/vertical-pod-autoscaler/docs/features.md#in-place-updates-inplaceorrecreate).
- `"Initial"`: VPA only assigns resource requests on pod creation and never changes them
  later.
- `"Off"`: VPA does not automatically change the resource requirements of the pods.
  The recommendations are calculated and can be inspected in the VPA object.

## Test your installation

A simple way to check if Vertical Pod Autoscaler is fully operational in your
cluster is to create a sample deployment and a corresponding VPA config:

```console
sudo k0s kubectl create -f examples/hamster.yaml
```

The above command creates a deployment with two pods, each running a single container
that requests 100 millicores and tries to utilize slightly above 500 millicores.
The command also creates a VPA config pointing at the deployment.
VPA will observe the behaviour of the pods, and after about 5 minutes, they should get
updated with a higher CPU request
(note that VPA does not modify the template in the deployment, but the actual requests
of the pods are updated). To see VPA config and current recommended resource requests run:

```console
sudo k0s kubectl describe vpa
```

*Note: if your cluster has little free capacity these pods may be unable to schedule.
You may need to add more nodes or adjust examples/hamster.yaml to use less CPU.*

## Example VPA configuration

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind:       Deployment
    name:       my-app
  updatePolicy:
    updateMode: "Auto"
```

## Troubleshooting

To diagnose problems with a VPA installation, perform the following steps:

- Check if all system components are running:

```console
sudo k0s kubectl --namespace=kube-system get pods|grep vpa
```

The above command should list 3 pods (recommender, updater and admission-controller)
all in state Running.

- Check if the system components log any errors.
  For each of the pods returned by the previous command do:

```console
sudo k0s kubectl --namespace=kube-system logs [pod name] | grep -e '^E[0-9]\{4\}'
```

- Check that the VPA Custom Resource Definition was created:

```console
sudo k0s kubectl get customresourcedefinition | grep verticalpodautoscalers
```
