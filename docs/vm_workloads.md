# VM workloads

In this lab, we will learn how to connect a workload running on a virtual machine to the Istio service mesh running on a Kubernetes cluster. Both Kubernetes cluster and the virtual machine will be running on Google Cloud Platform (GCP).

## Istio mesh installation

After we've created a Kubernetes cluster, we can download, install Istio, and configure Istio. We are using a pre-alpha feature that automatically creates WorkloadEntries for our VMs. To support this, we have to set the `PILOT_ENABLE_WORKLOAD_ENTRY_AUTOREGISTRATION` variable to true when installing Istio on the Kubernetes cluster.

Since we already installed Istio, we'll just update the existing installation using the IstioOperator. 

One of the differences between regular Istio installation and one that supports VM workloads is in setting cluster name and network. In this lab we'll set the network name to an empty string because we'll only use a single network - i.e. both the VM and the Kubernetes will be part of the same network. 

If had a scenario where the VM was running in a different network as the Kubernetes cluster, we'd set the network name to differentiate the networks.

??? note "Click for istio-vm-install.yaml"

    ```yaml linenums="1" title="istio-vm-install.yaml"
    --8<-- "vm-workloads/istio-vm-install.yaml"
    ```

Save the above YAML to `istio-vm-install.yaml`.

We'll use `istioctl` to install Istio on the cluster:

```shell
istioctl install -f istio-vm-install.yaml
```

Once the installation completes, we can deploy a separate ingress gateway that will be used to expose the Istio's control plane to the virtual machine.
## Installing and configuring the east-west gateway

The Istio package we downloaded contains a script we can use to generate the YAML that will deploy an Istio operator that creates the new gateway called `istio-eastwestgateway`.

Go to the folder where you downloaded Istio (e.g. `istio-{{istio.version}}`) and run this script:

```shell
samples/multicluster/gen-eastwest-gateway.sh --single-cluster | istioctl install -y -f -
```

You can list the Pods in the `istio-system` namespaces to check the gateway that was installed:

```shell
kubectl get po -n istio-system
```

```console
NAME                                     READY   STATUS    RESTARTS   AGE
istio-eastwestgateway-6d47fc4ddf-m89wn   1/1     Running   0          17s
istio-egressgateway-7767c67b7c-wmpcw     1/1     Running   0          11m
istio-ingressgateway-f5b854689-j9tcr     1/1     Running   0          11m
istiod-65cc74b9c7-j5n4d                  1/1     Running   0          4m59s
```

For virtual machines to access the Istio's control plane, we need to create a Gateway resource to configure the `istio-eastwestgateway`, and a VirtualService that has the Gateway attached.

We can use another script from the Istio package to create these resources and expose the control plane:

```shell
kubectl apply -n istio-system -f samples/multicluster/expose-istiod.yaml
```

```console
gateway.networking.istio.io/istiod-gateway created
virtualservice.networking.istio.io/istiod-vs created
```

The above resources (Gateway and VirtualService) configure the eastwest-gateway to listen on ports 15012 (used for control plane - istiod) and 15017 (used for pilot discovery validation webhook). The VirtualService matches the requests on the two ports and then routes them to istiod to appropriate ports.

The purpose of the eastwest-gateway in this scenario will be to allow the Istio sidecar proxy on the virtual machine to call the control plane running inside the cluster. The second role of the eastwest-gateway is in the scenarios where the VM and Kubernetes cluster are running in different networks. In that scenario, the workloads communicate to each through the eastwest-gateway.

## Preparing virtual machine namespace and files

For the virtual machine workloads, we have to create a separate namespace to store the WorkloadEntry resource and any other VM workload related resources. Additionally, we will have to export the cluster environment file, token, certificate, and other files we will have to transfer to the virtual machine.

Let's create a separate folder called `vm-files` to store these files. We can also save the full path to the folder in the `WORK_DIR` environment variable:

```shell
mkdir -p vm-files
export WORK_DIR="$PWD/vm-files"
```

We also set a couple of environment variables before continuing, so we don't have to re-type the values each time:

```shell
export VM_APP="hello-vm"
export VM_NAMESPACE="vm-namespace"
export SERVICE_ACCOUNT="vm-sa"
```

Let's create the VM namespace and the service account we will use for VM workloads in the same namespace:

```shell
kubectl create ns "${VM_NAMESPACE}"
```

```console
namespace/vm-namespace created
```

```shell
kubectl create serviceaccount "${SERVICE_ACCOUNT}" -n "${VM_NAMESPACE}"
```

```console
serviceaccount/vm-sa created
```

Next we'll create the WorkloadGroup resource using Istio CLI. Run the command to save the WorkloadGroup YAML to `workloadgroup.yaml`

```shell
istioctl x workload group create --name "${VM_APP}" --namespace "${VM_NAMESPACE}" --labels app="${VM_APP}" --serviceAccount "${SERVICE_ACCOUNT}" > workloadgroup.yaml
```

Here's how the contents of the `workloadgroup.yaml` should look like:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: WorkloadGroup
metadata:
  name: hello-vm
  namespace: vm-namespace
spec:
  metadata:
    labels:
      app: hello-vm
  template:
    serviceAccount: vm-sa
```

Save the above YAML to `workloadgroup.yaml` and create the resource with `kubectl apply -f workloadgroup.yaml`.

Virtual machine needs information about the cluster and Istio's control plane to connect to it. To generate the required files, we can run `istioctl x workload entry` command. We save all generated files to the `WORK_DIR`:

```shell
istioctl x workload entry configure \
  -f workloadgroup.yaml \
  -o "${WORK_DIR}" \
  --autoregister \
  --clusterID "Kubernetes"
```

```console
warning: a security token for namespace vm-namespace and service account vm-sa has been generated and stored at /vm-files/istio-token
configuration generation into directory /vm-files was successful
```

??? Warning "VM onboarding is not a one-time action"
      Whenever we update the WorkloadGroup resource, we also have to re-create the configuration (run the `istioctl x workload entry configure` command) and copy the generated files to the virtual machine.

The above command generates the following files:

- `cluster.env`: Contains metadata that identifies what namespace, service account, network CIDR and (optionally) what inbound ports to capture.
- `istio-token`: A Kubernetes token used to get certs from the CA. This is the the token from the `vm-sa` service account that's created using the `istio-ca` audience.
- `mesh.yaml`: Provides `ProxyConfig` to configure `discoveryAddress`, health-checking probes, and some authentication options.
- `root-cert.pem`: The root certificate used to authenticate. Comes from the `istio-ca-root-cert` ConfigMap in the `vm-namespace` namespace.
- `hosts`: An addendum to /etc/hosts that the proxy will use to reach istiod for xDS.

We'll copy the these files to the virtual machine, to locations known by the Envoy proxy. Note that the location where the files are copied can be customized, but it requires updates to the initial Envoy proxy configuration.

The files provide the Envoy proxy bootstrap configuration - they tell the proxy the information about the workload (e.g. the workload and service name and the namespace, the WorkloadGroup name, cluster ID and mesh ID and others).

To onboard the VM workload the sidecar also nees the Istio mTLS certificate. When the sidcear starts it uses the `istio-token` file to prove its identity to Istio control plane and to obtain the Istio mTLS certificate. The mTLS certificate is stored on disk for cases when the VM restarts.

The expiration time on the initial `istio-token` is 1 hour. If it takes you longer than 1 hour to onboard the VM, you' ll need to re-run the `istioctl x workload entry configure` command to generate a new token.

Once the VM obtains the mTLS certificate, it can use it to prove its identity. The certificate is valid for 24 hours and the proxy will automatically refresh it before it expires.


## Create the Virtual Machine

We'll be running the virtual machine in GCP, just like the Kubernetes cluster.

1. Create the VM

    ```shell
    gcloud compute instances create my-mesh-vm \
      --tags=mesh-vm \
      --machine-type=n1-standard-2
    ```

1. Obtain the cluster's Pod IP address range.

    ```shell
    export CLUSTER_POD_CIDR=$(gcloud container clusters list --format=json | jq -r '.[0].clusterIpv4Cidr')
    ```

1. Create a firewall rule to allow ingress on port 80 from the cluster pods to the VM.

    ```shell
    gcloud compute firewall-rules create "cluster-pods-to-vm" \
      --source-ranges=$CLUSTER_POD_CIDR \
      --target-tags=mesh-vm \
      --action=allow \
      --rules=tcp:80
    ```

## Applying configuration to the Virtual Machine

Now it's time to configure the Virtual machine. In this example, we run a simple Python HTTP server on port 80. You could configure any other service on a different port. Just make sure you configure the security and firewall rules accordingly.

1. Copy the files from `vm-files` folder to the home folder on the instance. Replace `INSTANCE_ZONE` accordingly.

    ```shell
    gcloud compute scp vm-files/* my-mesh-vm:~ --zone=[INSTANCE_ZONE]
    ```

    ```console
    cluster.env              100%  627   626.9KB/s   00:00
    hosts                    100%   36    50.7KB/s   00:00
    istio-token              100%  905     1.4MB/s   00:00
    mesh.yaml                100%  668   918.7KB/s   00:00
    root-cert.pem            100% 1094     1.7MB/s   00:00
    ```

1. SSH into the instance and copy the root certificate to `/etc/certs` (you can use the command from the instance details page in the SSH dropdown):

    ```shell
    gcloud beta compute ssh --zone=[INSTANCE_ZONE] my-mesh-vm
    ```

    ```shell
    sudo mkdir -p /etc/certs
    sudo cp root-cert.pem /etc/certs/root-cert.pem
    ```

1. Copy the `istio-token` file to `/var/run/secrets/tokens` folder:

    ```shell
    sudo mkdir -p /var/run/secrets/tokens
    sudo cp istio-token /var/run/secrets/tokens/istio-token
    ```

1. Download and install the Istio sidecar package:

    ```shell
    curl -LO https://storage.googleapis.com/istio-release/releases/{{istio.version}}/deb/istio-sidecar.deb
    sudo dpkg -i istio-sidecar.deb
    ```

1. Copy `cluster.env` to `/var/lib/istio/envoy/`:

    ```shell
    sudo cp cluster.env /var/lib/istio/envoy/cluster.env
    ```

1. Copy Mesh config (`mesh.yaml`) to `/etc/istio/config/mesh`:

    ```shell
    sudo cp mesh.yaml /etc/istio/config/mesh
    ```

1. Add the istiod host to the `/etc/hosts` file:

    ```shell
    sudo sh -c 'cat $(eval echo ~$SUDO_USER)/hosts >> /etc/hosts'
    ```

1. Change the ownership of files in `/etc/certs` and `/var/lib/istio/envoy` to the Istio proxy:

    ```shell
    sudo mkdir -p /etc/istio/proxy
    sudo chown -R istio-proxy /var/lib/istio /etc/certs /etc/istio/proxy /etc/istio/config /var/run/secrets /etc/certs/root-cert.pem
    ```

With all files in place, we are ready to start Istio on the virtual machine.

Because we used the VM auto-registration, Istio automatically creates the WorkloadEntry resource for us shortly after we start the Istio service on the VM.

You can see the WorkloadEntry creation _in action_ as follows:

1. Watch for Workloadentry resources using the `--watch` flag

      ```shell
      kubectl get workloadentry -n vm-namespace --watch
      ```

1. On the VM, start the Istio service:

      ```shell
      sudo systemctl start istio
      ```

1. You should see the WorkloadEntry appear

      ```console
      NAME                  AGE   ADDRESS
      hello-vm-10.128.0.7   12m   10.128.0.7
      ```

1. Press Ctrl-C to stop watching for changes.

On the VM, you can check that the `istio` service is running with `systemctl status istio`. Alternatively, we can look at the contents of the `/var/log/istio/istio.log` to see that the proxy was successfully started.

At this point, the virtual machine is configured to talk with the Istio's control plane in the Kubernetes cluster.

## Access services from the virtual machine

From a different terminal window, we can now deploy a Hello world application to the Kubernetes cluster

??? note "Click for hello-world.yaml"

    ```yaml linenums="1" title="hello-world.yaml"
    --8<-- "vm-workloads/hello-world.yaml"
    ```

Save the above YAML to `hello-world.yaml` and deploy it using `kubectl apply -f hello-world.yaml`.

Wait for the Pods to become ready and then go back to the virtual machine and try to access the Kubernetes service:

```shell
curl http://hello-world.default
```

```console
Hello World
```

You can access any service running within your Kubernetes cluster from the virtual machine.

## Run services on the virtual machine

We can also run a workload on the virtual machine. Switch to the virtual machine instance and run a simple Python HTTP server:

```shell
sudo python3 -m http.server 80
```

```console
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

What we want to do next is add the workload (Python HTTP service) to the mesh. For that reason, we created the VM namespace earlier. So let's create a Kubernetes service that represents the VM workload. Note that the name and the label values equal to the value of the `VM_APP` environment variable we set earlier. Don't forget to deploy the service to the `VM_NAMESPACE`.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-vm
  namespace: vm-namespace
  labels:
    app: hello-vm
spec:
  ports:
  - port: 80
    name: http-vm
    targetPort: 80
  selector:
    app: hello-vm
```

Save the above YAML to `hello-vm-service.yaml` and deploy it to the VM namespace using with `kubectl apply -f hello-vm-service.yaml`.

We can now use the Kubernetes service name `hello-vm.vm-namespace` to access the workload running on the virtual machine (the Python server). Let's run a Pod inside the cluster and try to access the service from there:

```shell
kubectl run curl --image=radial/busyboxplus:curl -i --tty --rm
```

```console
If you don't see a command prompt, try pressing enter.
[ root@curl:/ ]$
```

After you get the command prompt in the Pod, you can run curl and access the workload. You should see a directory listing response. Similarly, you will notice a log entry on the instance where the HTTP server is running:

```shell
[ root@curl:/ ]$ curl hello-vm.vm-namespace
```

```console
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN" "http://www.w3.org/TR/html4/strict.dt
d">
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=utf-8">
<title>Directory listing for /</title>
</head>
<body>
<h1>Directory listing for /</h1>
<hr>
...
```