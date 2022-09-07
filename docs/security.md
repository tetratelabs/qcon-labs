# Security - Access control

In this lab, we will learn how to use an authorization policy to control access between workloads.

## Prerequisites and setup

Let's start by deploying the Gateway:

??? note "Click for gateway-all.yaml"

    ```yaml linenums="1" title="gateway-all.yaml"
    --8<-- "gateway-all.yaml"
    ```

Save the above YAML to `gateway-all.yaml` and deploy the Gateway using `kubectl apply -f gateway-all.yaml`.

Next, we will create the Web frontend deployment, service account, service, and a VirtualService.

??? note "Click for web-frontend.yaml"
    ```yaml linenums="1" title="web-frontend.yaml"
    --8<-- "web-frontend.yaml"
    ```

Save the above YAML to `web-frontend.yaml` and create the resource using `kubectl apply -f web-frontend.yaml`.

Finally, we will deploy the `customers` service.

??? note "Click for customers.yaml"
    ```yaml linenums="1" title="customers.yaml"
    --8<-- "customers.yaml"
    ```

Save the above YAML to `customers.yaml` and create the resources with `kubectl apply -f customers.yaml`.

Let's set the `GATEWAY_IP` variable:

```shell
export GATEWAY_IP=$(kubectl get svc istio-ingressgateway -n istio-system -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
```

If we open the `GATEWAY_IP` the web frontend page with the data from the `customers` service should be displayed.

## Denying all requests between services

Let's start by creating an authorization policy that denies all requests in the default namespace.

```yaml linenums="1" title="deny-all.yaml"
--8<-- "deny-all.yaml"
```

Save the above YAML to `deny-all.yaml` and create the policy using `kubectl apply -f deny-all.yaml`.

If we try to access `GATEWAY_IP` we will get back the following response:

```console
RBAC: access denied
```

Similarly, if we try to run a Pod inside the cluster and make a request from within the default namespace to either the web frontend or the customer service, we'll get the same error.

Let's try that:

```shell
kubectl run curl --image=radial/busyboxplus:curl -i --tty --rm
```

```shell
[ root@curl:/ ]$ curl customers
RBAC: access denied
[ root@curl:/ ]$ curl web-frontend
RBAC: access denied
[ root@curl:/ ]$
```

In both cases, we get back the access denied error. You can type `exit` to exit the curl pod.

## Allow requests from ingress to `web-frontend`

The first thing we will do is to allow requests being sent from the ingress gateway to the `web-frontend` application using an ALLOW action. In the rules, we are specifying the source namespace (`istio-system`) where the ingress gateway is running and the ingress gateway's service account name in the `principals` field.

```yaml linenums="1" title="allow-ingress-frontend.yaml"
--8<-- "allow-ingress-frontend.yaml"
```

Save the above YAML to `allow-ingress-frontend.yaml` and create the policy using `kubectl apply -f allow-ingress-frontend.yaml`.

If we try to make a request from our host to the `GATEWAY_IP`, we will get a different error this time:

```shell
curl http://$GATEWAY_IP
```

```console
"Request failed with status code 403"
```

This error is coming from the `customers` service - remember we allowed calls to the web frontend. However, `web-frontend` still can't make calls to the `customers` service.

If we go back to the `curl` Pod we are running inside the cluster and try to request `http://web-frontend` we will get an RBAC error.

The DENY policy is in effect, and we are only allowing calls to be made from the ingress gateway.

## Allow requests from `web-frontend` to `customers`

When we deployed the `web-frontend`, we also created a service account for the Pod to use. Otherwise, all Pods in the namespace are assigned the default service account. Using the service account we give your pods an identity.

We can now use that service account to specify where the customer service calls can come from.

```yaml linenums="1" title="allow-web-frontend-customers.yaml"
--8<-- "allow-web-frontend-customers.yaml"
```

Save the above YAML to `allow-web-frontend-customers.yaml` and create the policy using `kubectl apply -f allow-web-frontend-customers.yaml`.

As soon as the policy is created, we will see the web frontend working again - it will get the customer service responses. You can try that it works by opening the GATEWAY_IP in the browser or using cURL to send the request to the GATEWAY_IP.

```shell
export GATEWAY_IP=$(kubectl get svc istio-ingressgateway -n istio-system -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl http://$GATEWAY_IP
```

```console
<!DOCTYPE html>
<html>
  <head>
    <title>Web Frontend</title>
    <link rel="stylesheet" href="/css/style.css" />
  </head>
  <body class="bg-tetrate-black">
...
```

If we create a Pod inside the cluster again to try and directly access `web-frontend` and `customers` service, we'll notice that the calls will continue to fail, which is expected because we haven't explicitly allowed calls from the cURL Pod:

```shell
kubectl run curl --image=radial/busyboxplus:curl -i --tty --rm
```

```console
If you don't see a command prompt, try pressing enter.
[ root@curl:/ ]$ curl customers
RBAC: access denied
[ root@curl:/ ]$ curl web-frontend
RBAC: access denied
[ root@curl:/ ]$
```

## Conclusion

We have used multiple authorization policies to explicitly allow calls from the ingress to the front end and from the `web-frontend` to the `customers` service. Using a deny-all policy is a good way to start because we can control, manage, and then explicitly allow the communication we want to happen between services.

## Cleanup

Delete the Deployments, Services, VirtualServices, and the Gateway:

```shell
kubectl delete deploy web-frontend customers-v1
kubectl delete svc customers web-frontend
kubectl delete vs --all
kubectl delete gateway gateway
kubectl delete authorizationpolicy --all
kubectl delete pod curl
```
