# VM workloads

Istio supports running workloads outside of Kubernetes.
The canonical example is a legacy application that, for some reason, is not containerized and must run on a virtual machine (VM).

To explore [onboarding VM workloads with Istio](https://istio.io/latest/docs/ops/deployment/vm-architecture/){target=_blank}, let us pretend that our venerable `customers` service is such a legacy application.

In this lab, you will:

1. Enable [VM] workload entry auto-registration (a feature considered experimental at time of writing)
1. Provision a VM and deploy the `customers` app on it.
1. Enable data-plane communication between the VM workload and the mesh
1. Configure control-plane communication between `istiod` and the VM
1. Define a WorkloadGroup, a concept analogous to Kubernetes [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/){target=_blank} resource
1. Install and configure the Envoy sidecar on the VM

The fruit of this labor will be to witness the `web-frontend` service successfully make calls to the `customers` backend running on the VM.

The instructions in this lab are specific to Google Cloud Platform (GCP).

## Set default compute region and zone

Configuring the default region and zone helps avoid specifying them in subsequent commands:

```shell
gcloud config set compute/region us-west1
```

```shell
gcloud config set compute/zone us-west1-a
```

## Enable WorkloadEntry auto-registration

Update the mesh configuration to turn on this capability:

```shell
istioctl install \
  --set values.pilot.env.PILOT_ENABLE_WORKLOAD_ENTRY_AUTOREGISTRATION=true
```

## Provision the VM

```shell
gcloud compute instances create my-mesh-vm \
  --tags=mesh-vm \
  --machine-type=n1-standard-2
```

Above, we tag the VM to simplify configuring communication to the VM.

## Deploy `customers` to the VM

Ssh to the VM:

```shell
gcloud compute ssh ubuntu@my-mesh-vm
```

The most expedient way to deploy the `customers` app is to leverage its Docker image.  So..

1. Install docker:

    ```shell
    sudo apt-get update
    sudo apt-get -y install docker.io
    ```

1. Run the container, using version 2.0 to easily distinguish this workload from the one already running on Kubernetes:

    ```shell
    sudo docker run -d --name customers \
      --net host \
      gcr.io/tetratelabs/customers:2.0.0
    ```

Verify that the `customers` app is running:

```shell
curl localhost:3000
```

## Enable data-plane communication

In this scenario, both the VM and the cluster exist in the same network.

Use a firewall rule to enable communication from the Pod IP addresses to the VM in question.

1. Capture the Pod IP address range:

    ```shell
    export CLUSTER_POD_CIDR=$(gcloud container clusters list --format=json | jq -r '.[0].clusterIpv4Cidr')
    ```

1. Apply the firewall rule:

    ```shell
    gcloud compute firewall-rules create "cluster-pods-to-vm" \
      --source-ranges=$CLUSTER_POD_CIDR \
      --target-tags=mesh-vm \
      --action=allow \
      --rules=tcp:3000
    ```

!!! info

    The path of communication between the VM workload and the Kubernetes cluster depends on the network topology.

    In scenarios where the VM and the cluster are on separate networks, that communication path is altered to use an "east-west" ingress gateway.

## Configure control-plane communication

1. Deploy an additional Istio Ingress Gateway, that we label as "eastwest".  The Istio distribution provides a simple script for this purpose.

    First, navigate to your Istio distribution folder, then run:

    ```shell
    ./samples/multicluster/gen-eastwest-gateway.sh --single-cluster | istioctl install -y -f -
    ```

1. Confirm the presence of the new "eastwest" gateway deployment:

    ```shell
    kubectl get pod -n istio-system
    ```

1. Expose the services of `istiod` with a `Gateway` and `VirtualService` resource.  Here too, the Istio distribution provides the recipe:

    ```shell
    kubectl apply -n istio-system -f ./samples/multicluster/expose-istiod.yaml
    ```

!!! info

    The above resources (Gateway and VirtualService) configure the eastwest-gateway to listen on ports 15012 (used for control plane - istiod) and 15017 (used for pilot discovery validation webhook).

    The VirtualService matches the requests on the two ports and then routes them to istiod to appropriate ports.

## Define a WorkloadGroup

Istio provides the WorkloadGroup and WorkloadEntry resources; they are the VM analogs of the Kubernetes Deployment and Pod resources.

???+ tldr "customers-workloadgroup.yaml"
    ```yaml linenums="1"
    --8<-- "vm-workloads/customers-workloadgroup.yaml"
    ```

This resource provides a template for Workload Entries, and ensures that newly-created workload entries are associated with the existing `customers` service account, for workload identity purposes.

Apply the above to the cluster:

```shell
kubectl apply -f customers-workloadgroup.yaml
```

## Install and configure the Envoy sidecar on the VM

### Generate the configuration files

The `istioctl` CLI provides a command to automate the generation of all files needed to configure the Envoy sidecar on the VM:

1. Create a folder to contain these files:

    ```shell
    mkdir vm_files
    ```

1. Run the command to generate the VM files:

    ```shell
    istioctl x workload entry configure \
      --file customers-workloadgroup.yaml \
      --output vm_files \
      --autoregister
    ```

!!! Warning "VM onboarding is not a one-time action"

    Whenever we update the WorkloadGroup resource, we also have to re-create the configuration (run the `istioctl x workload entry configure` command) and copy the generated files to the VM.

Inspect the contents of the `vm_files` folder.

Here is how each file is described in the [Istio documentation](https://istio.io/latest/docs/setup/install/virtual-machine/#create-files-to-transfer-to-the-virtual-machine):

- `cluster.env`: Contains metadata that identifies what namespace, service account, network CIDR and (optionally) what inbound ports to capture.
- `istio-token`: A Kubernetes token used to get certs from the CA.
- `mesh.yaml`: Provides `ProxyConfig` to configure `discoveryAddress`, health-checking probes, and some authentication options.
- `root-cert.pem`: The root certificate used to authenticate.
- `hosts`: An addendum to `/etc/hosts` that the proxy will use to reach istiod for xDS.

!!! info

    To onboard the VM workload the sidecar also nees the Istio mTLS certificate. When the sidcear starts it uses the `istio-token` file to prove its identity to Istio control plane and to obtain the Istio mTLS certificate. The mTLS certificate is stored on disk for cases when the VM restarts.

    The expiration time on the initial `istio-token` is 1 hour. If it takes you longer than 1 hour to onboard the VM, you' ll need to re-run the `istioctl x workload entry configure` command to generate a new token.

    Once the VM obtains the mTLS certificate, it can use it to prove its identity. The certificate is valid for 24 hours and is automatically rotated before it expires.

### Transfer the files to the VM

Use the following command to copy the contents of the `vm_files` folder to the VM:

```shell
gcloud compute scp vm_files/* ubuntu@my-mesh-vm:
```

### Configure the VM

1. SSH onto the VM

    ```shell
    gcloud compute ssh my-mesh-vm
    ```

1. Install the root certificate to `/etc/certs`:

    ```shell
    sudo mkdir -p /etc/certs
    sudo cp ~/root-cert.pem /etc/certs/root-cert.pem
    ```

1. Install the token to `/var/run/secrets/tokens`:

    ```shell
    sudo mkdir -p /var/run/secrets/tokens
    sudo cp ~/istio-token /var/run/secrets/tokens/istio-token
    ```

1. Download and install the Istio sidecar package:

    ```shell
    curl -LO https://storage.googleapis.com/istio-release/releases/{{istio.version}}/deb/istio-sidecar.deb
    sudo dpkg -i istio-sidecar.deb
    ```

1. Install `cluster.env` to `/var/lib/istio/envoy/`:

    ```shell
    sudo cp ~/cluster.env /var/lib/istio/envoy/cluster.env
    ```

1. Install the Mesh Config to `/etc/istio/config/mesh`:

    ```shell
    sudo cp ~/mesh.yaml /etc/istio/config/mesh
    ```

1. Add the `istiod` host to `/etc/hosts`:

    ```shell
    sudo sh -c 'cat $(eval echo ~$SUDO_USER)/hosts >> /etc/hosts'
    ```

1. Change the ownership of files in `/etc/certs` and `/var/lib/istio/envoy` to the Istio proxy:

    ```shell
    sudo mkdir -p /etc/istio/proxy
    sudo chown -R istio-proxy /etc/certs /var/run/secrets /var/lib/istio /etc/istio/config /etc/istio/proxy
    ```

With all files in place, we are ready to start the sidecar on the VM.

## Start the sidecar

Because we used the VM auto-registration, Istio automatically creates the WorkloadEntry resource for us shortly after we start the Istio service on the VM.

1. Watch for Workloadentry resources using the `--watch` flag

    ```shell
    kubectl get workloadentry --watch
    ```

1. On the VM, start the Istio service:

    ```shell
    sudo systemctl start istio
    ```

1. See the WorkloadEntry appear:

    ```console
    NAME                      AGE   ADDRESS
    customers-10.128.0.7   12s   10.128.0.7
    ```

1. Press Ctrl-C to stop watching for changes.


On the VM, check that the `istio` service is running:

```shell
sudo systemctl status istio.service
```

The full logs can be accessed with `journalctl` (or by tailing `/var/log/istio/istio.log`):

```shell
sudo journalctl -u istio.service
```

## Test it!

Confirm that the `customer` workload is accessible from the Kubernetes cluster.

1. Confirm that `web-frontend` now has **two** endpoints for the customers service:

    ```shell
    istioctl proxy-config endpoints deploy/web-frontend.default | grep customers
    ```

1. Open a browser to your `$GATEWAY_IP` and see whether you can get the two-column version of the customers service to display (you may need to refresh the page once or twice)

1. Call `customers` directly from `sleep`:

    ```shell
    kubectl exec deploy/sleep -- curl -s customers
    ```

## Access services from the VM

Try to access the `helloworld` service running on Kubernetes _from the VM_:

```shell
curl http://helloworld.default:5000/hello
```
