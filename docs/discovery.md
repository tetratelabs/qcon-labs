# Service discovery and load balancing

This lab explores service discovery and load balancing in Istio.

You will begin by spying on Istio and the Envoy configuration of the sidecars to confirm that workloads are indeed configured by Istio to know about other services and their endpoints.
The `istioctl` CLI's two diagnostic commands `proxy-status` and `proxy-config` will be of assistance for this purpose.

Next, you will scale the `customers` service to two replicas in order to witness and confirm that clients indeed load-balance requests across a target service's endpoints.

Finally, you will apply a destination rule to alter the load balancing algorithm used to call the `customers` service.

## Clusters and endpoints

Confirm that `istiod` knows about the workloads running on Kubernetes:

```shell
istioctl proxy-status
```

### The service registry

Istio maintains an internal service registry which can be observed through a debug endpoint, as follows:

1. Capture the name of the istiod pod to an environment variable

    ```shell
    export ISTIO_POD=$(kubectl get pod -n istio-system -l app=istiod -ojsonpath='{.items[0].metadata.name}')
    ```

1. `curl` the registry endpoint:

    ```shell
    kubectl exec -n istio-system $ISTIO_POD -- curl localhost:15014/debug/registryz
    ```

    The output can be prettified with a tool such as [`jq`](https://stedolan.github.io/jq/){target=_blank}.

### The sidecar configuration

Review the deployments in the `default` namespace:

```shell
kubectl get deploy
```

Envoy's term for a service is "cluster".

Confirm that `web-frontend` knows about other services, including the `customers` service:

```shell
istioctl proxy-config clusters deploy/web-frontend
```

List the endpoints backing each "cluster":

```shell
istioctl proxy-config endpoints deploy/web-frontend
```

## Load balancing

Zero in on the endpoints for the `customers` service, and note the `customers` endpoint:

```shell
istioctl proxy-config endpoints deploy/web-frontend --cluster "outbound|80||customers.default.svc.cluster.local"
```

Scale `customers` to two replicas:

```shell
kubectl scale deploy customers-v1 --replicas 2
```

Re-run the above `proxy-config endpoints` command; you should now see two endpoints:

```shell
istioctl proxy-config endpoints deploy/web-frontend --cluster "outbound|80||customers.default.svc.cluster.local"
```

### Check the logs

1. Capture the name of each `customers` workload in environment variables:

    ```shell
    CUSTOMERS_POD_1=$(kubectl get pod -l app=customers -ojsonpath='{.items[0].metadata.name}')
    ```

    And:

    ```shell
    CUSTOMERS_POD_2=$(kubectl get pod -l app=customers -ojsonpath='{.items[1].metadata.name}')
    ```

1. Tail the logs of each customers pods in two separate terminals:

    ```shell
    kubectl logs --follow $CUSTOMERS_POD_1
    ```

    And:

    ```shell
    kubectl logs --follow $CUSTOMERS_POD_2
    ```

The `sleep` pod's container image has `curl` pre-installed.  `sleep`, like `web-frontend`, also knows about the `customers` service endpoints.

1. Capture the name of the sleep pod in an environment variable:

    ```shell
    SLEEP_POD=$(kubectl get pod -l app=sleep -ojsonpath='{.items[0].metadata.name}')
    ```

1.  Make repeated calls to the `customers` service from the `sleep` pod:

    ```shell
    kubectl exec $SLEEP_POD -- curl -s customers
    ```

Watch the log output from both `customers` pods and confirm that calls are indeed load-balanced.

## Destination Rules

Explore applying a [DestinationRule](https://istio.io/latest/docs/reference/config/networking/destination-rule/){target=_blank}
CRD (Custom Resource Definition) to to alter the load balancer configuration.

Apply the following resource, which sets the load balancing algorithm to the `customers` service to [`LEAST_REQUEST`](https://istio.io/latest/docs/reference/config/networking/destination-rule/#LoadBalancerSettings-SimpleLB){target=_blank}:

!!! tldr "destination-rule.yaml"
    ```yaml linenums="1"
    --8<-- "discovery/destination-rule.yaml"
    ```

In Envoy, the load balancer policy is associated to a given upstream service, in Envoy's terms, it's in the "cluster" config.

Look for `lbPolicy` field in cluster configuration YAML output:

```shell
istioctl proxy-config cluster $SLEEP_POD \
  --fqdn customers.default.svc.cluster.local -o yaml \
  | grep lbPolicy -B 5 -A 3
```

In the output, confirm that the value of `lbPolicy` is `LEAST_REQUEST`.
