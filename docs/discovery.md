# Service discovery and load balancing

In this lab you will spy on the Envoy configuration of the sidecars to confirm that workloads have been configured by Istio to know about other services and their endpoints.  Envoy's term for a service is a "cluster".  The `istioctl` CLI's two diagnostic commands `proxy-status` and `proxy-config` will be of assistance for this purpose.

Second, you will scale the customers service to two replicas in order to witness and confirm that clients indeed load-balance requests across a target service's endpoints.

## Clusters and endpoints

Review the deployments in the `default` namespace:

```shell
kubectl get deploy
```

`istiod` knows about the workloads running on Kubernetes:

```shell
istioctl proxy-status
```

`web-frontend` knows about other services (aka "clusters" in Envoy parlance), including the `customers` service:

```shell
istioctl proxy-config clusters deploy/web-frontend
```

We can also list the specific endpoints backing each "cluster":

```shell
istioctl proxy-config endpoints deploy/web-frontend
```

## Load balancing

Zero in on the endpoints for the customers service, and note the one `customers` endpoint:

```shell
istioctl proxy-config endpoints deploy/web-frontend --cluster "outbound|80||customers.default.svc.cluster.local"
```

Scale customers to two replicas:

```shell
kubectl scale deploy customers-v1 --replicas 2
```

Re-run the `proxy-config endpoints` command; should now see two endpoints:

```shell
istioctl proxy-config endpoints deploy/web-frontend --cluster "outbound|80||customers.default.svc.cluster.local"
```

If possible, tail the logs of both customers pods in two separate terminals, with the help of a command similar to this:

```shell
kubectl logs --follow customers-v1-<tab>
```

The `sleep` pod's container image has `curl` pre-installed.  `sleep`, like web-frontend, also knows about the `customers` service workloads.

Make repeated calls to the `customers` service from the `sleep` pod:

```shell
kubectl exec sleep-<tab> -- curl -s customers
```

Watch the log output from both `customers` pods and confirm that calls are indeed load-balanced.
