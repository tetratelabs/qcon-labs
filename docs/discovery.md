# Service discovery and load balancing

This lab explores service discovery and load balancing in Istio.

## Clusters and endpoints

The `istioctl` CLI's diagnostic command `proxy-status` provides a simple way to list all proxies that Istio knows about.

Confirm that `istiod` knows about the workloads running on Kubernetes:

```shell
istioctl proxy-status
```

## Deploy the `helloworld` sample

The Istio distribution comes with a sample application "helloworld".

```shell
cd ~/istio-{{istio.version}}
```

Deploy `helloworld` to the default namespace.

```shell
kubectl apply -f samples/helloworld/helloworld.yaml
```

Check the output of `proxy-status` again, and confirm that `helloworld` is listed:

```shell
istioctl proxy-status
```

### The service registry

Istio maintains an internal service registry which can be observed through a debug endpoint `/debug/registryz` exposed by `istiod`:

1. Capture the name of the `istiod` pod to an environment variable

    ```shell
    export ISTIO_POD=$(kubectl get pod -n istio-system -l app=istiod -ojsonpath='{.items[0].metadata.name}')
    ```

1. `curl` the registry endpoint:

    ```shell
    kubectl exec -n istio-system $ISTIO_POD -- curl localhost:15014/debug/registryz
    ```

    The output can be prettified with a tool such as [`jq`](https://stedolan.github.io/jq/){target=_blank}.

    ```shell
    kubectl exec -n istio-system $ISTIO_POD -- curl localhost:15014/debug/registryz | jq .[].hostname
    ```

### The sidecar configuration

Review the deployments in the `default` namespace:

```shell
kubectl get deploy
```

The `istioctl` CLI's diagnostic command `proxy-config` will help us inspect the configuration of proxies.

Envoy's term for a service is "cluster".

Confirm that `sleep` knows about other services (`helloworld`, `customers`, etc..):

```shell
istioctl proxy-config clusters deploy/sleep
```

List the endpoints backing each "cluster":

```shell
istioctl proxy-config endpoints deploy/sleep
```

Zero in on the endpoints for the `helloworld` service:

```shell
istioctl proxy-config endpoints deploy/sleep \
  --cluster "outbound|5000||helloworld.default.svc.cluster.local"
```

## Load balancing

The `sleep` pod's container image has `curl` pre-installed.

1. Capture the name of the sleep pod in an environment variable:

    ```shell
    SLEEP_POD=$(kubectl get pod -l app=sleep -ojsonpath='{.items[0].metadata.name}')
    ```

1.  Make repeated calls to the `helloworld` service from the `sleep` pod:

    ```shell
    kubectl exec $SLEEP_POD -- curl -s helloworld:5000/hello
    ```

Some responses will be from `helloworld-v1` while others from `helloworld-v2`, an indication that Envoy is load-balancing requests between these two endpoints.
