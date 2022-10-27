# Observability

This lab explores one of the main strengths of Istio: observability.

The services in our mesh are automatically observable, without adding any burden on developers.

## Deploy the Addons

The Istio distribution provides addons for a number of systems that together provide observability for the service mesh:

- [Zipkin](https://zipkin.io/){target=_blank} or [Jaeger](https://www.jaegertracing.io/){target=_blank} for distributed tracing
- [Prometheus](https://prometheus.io/){target=_blank} for metrics collection
- [Grafana](https://grafana.com/){target=_blank} provides dashboards for monitoring, using Prometheus as the data source
- [Kiali](https://kiali.io/){target=_blank} allows us to visualize the mesh

These addons are located in the `samples/addons/` folder of the distribution.

1. Navigate to the addons directory

    ```{.shell .language-shell}
    cd ~/istio-{{istio.version}}/samples/addons
    ```

1. Deploy each addon:

    ```{.shell .language-shell}
    kubectl apply -f extras/zipkin.yaml
    ```

    ```{.shell .language-shell}
    kubectl apply -f prometheus.yaml
    ```

    ```{.shell .language-shell}
    kubectl apply -f grafana.yaml
    ```

    ```{.shell .language-shell}
    kubectl apply -f kiali.yaml
    ```

1. Verify that the `istio-system` namespace is now running additional workloads for each of the addons.

    ```{.shell .language-shell}
    kubectl get pod -n istio-system
    ```

The `istioctl` CLI provides convenience commands for accessing the web UIs for each dashboard.

Take a moment to review the help information for the `istioctl dashboard` command:

```{.shell .language-shell}
istioctl dashboard --help
```

## Generate a load

In order to have something to observe, we need to generate a load on our system.

### Install a load generator

Install a simple load generating tool named [`siege`](https://github.com/JoeDog/siege){target=_blank}.

We normally install `siege` with the `apt-get` package manager.
However, given the cloud shell's ephemeral nature, anything installed outside our home directory will vanish after a session timeout.

Alternatives:

1. Install from source. It's a little more work, but does not exhibit the above-mentioned problem.
1. Run the load generator from your laptop.  On a mac, using homebrew the command is `brew install siege`.
1. Use a simple bash `curl` command wrapped in a `while` loop:
    ```shell
    while true; do curl $GATEWAY_IP | head; sleep 1s; done
    ```

Here are the steps to install from source:

1. Fetch the package

    ```{.shell .language-shell}
    wget http://download.joedog.org/siege/siege-latest.tar.gz
    ```

1. Unpack it

    ```{.shell .language-shell}
    tar -xzf siege-latest.tar.gz
    ```

1. Navigate into the siege subdirectory with `cd siege`++tab++

1. Run the `configure` script, and request that siege get installed inside your home directory

    ```{.shell .language-shell}
    ./configure --prefix=$HOME
    ```

1. Build the code

    ```{.shell .language-shell}
    make
    ```

1. Finally, install (copies the binary to `~/bin`)

    ```{.shell .language-shell}
    make install
    ```

Feel free to delete (or preserve) the downloaded tar file and source code.

### Generate a load

With `siege` now installed, familiarize yourself with the command and its options.

```{.shell .language-shell}
siege --help
```

Run the following command to generate a mild load against the application.

```{.shell .language-shell}
siege --delay=3 --concurrent=3 --time=20M http://$GATEWAY_IP/
```

!!! note

    The `siege` command stays in the foreground while it runs.
    It may be simplest to leave it running, and open a separate terminal in your cloud shell environment.

## Kiali

Launch the Kiali dashboard:

```{.shell .language-shell}
istioctl dashboard kiali
```

!!! warning

    If the dashboard page fails to open, just click on the hyperlink in the console output.

!!! note

    The `istioctl dashboard` command also blocks.
    Leave it running until you're finished using the dashboard, at which time 
    pressing ++ctrl+c++ can interrupt the process and put you back at the terminal prompt.

The Kiali dashboard displays.

Customize the view as follows:

1. Select the _Graph_ section from the sidebar.
1. Under _Select Namespaces_ (at the top of the page), select the `default` namespace, the location where the application's pods are running.
1. From the third "pulldown" menu, select _App graph_.
1. From the _Display_ "pulldown", toggle on _Traffic Animation_ and _Security_.
1. From the footer, toggle the legend so that it is visible.  Take a moment to familiarize yourself with the legend.

Observe the visualization and note the following:

- We can see traffic coming in through the ingress gateway to the `web-frontend`, and the subsequent calls from the `web-frontend` to the `customers` service
- The lines connecting the services are green, indicating healthy requests
- The small lock icon on each edge in the graph indicates that the traffic is secured with mutual TLS

Such visualizations are helpful with understanding the flow of requests in the mesh, and with diagnosis.

Feel free to spend more time exploring Kiali.

We will revisit Kiali in a later lab to visualize traffic shifting such as when performing a blue-green or canary deployment.

### Kiali Cleanup

Close the Kiali dashboard.  Interrupt the `istioctl dashboard kiali` command by pressing ++ctrl+c++.


## Zipkin

Launch the Zipkin dashboard:

```{.shell .language-shell}
istioctl dashboard zipkin
```

The Zipkin dashboard displays.

- Click on the red '+' button and select _serviceName_.
- Select the service named `web-frontend.default` and click on the _Run Query_ button (lightblue) to the right.

A number of query results will display.  Each row is expandable and will display more detail in terms of the services participating in that particular trace.

- Click the _Show_ button to the right of one of the traces having four (4) spans.

The resulting view shows spans that are part of the trace, and more importantly how much time was spent within each span.  Such information can help diagnose slow requests and pin-point where the latency lies.

Distributed tracing also helps us make sense of the flow of requests in a microservice architecture.

### Zipkin Cleanup

Close the Zipkin dashboard.  Interrupt the `istioctl dashboard zipkin` command with ++ctrl+c++.


## Prometheus

Prometheus works by periodically calling a metrics endpoint against each running service (this endpoint is termed the "scrape" endpoint).  Developers normally have to instrument their applications to expose such an endpoint and return metrics information in the format the Prometheus expects.

With Istio, this is done automatically by the Envoy sidecar.

### Observe how Envoy exposes a Prometheus scrape endpoint

1. Capture the customers pod name to a variable.

    ```{.shell .language-shell}
    CUSTOMERS_POD=$(kubectl get pod -l app=customers -ojsonpath='{.items[0].metadata.name}')
    ```

1. Run the following command:

    ```{.shell .language-shell}
    kubectl exec $CUSTOMERS_POD -- curl -s localhost:15020/stats/prometheus  | grep istio_requests
    ```

    !!! info "Why port 15020?"

        See [Ports used by Istio](https://istio.io/latest/docs/ops/deployment/requirements/#ports-used-by-istio){target=_blank} sidecar proxy.


    The list of metrics returned by the endpoint is rather lengthy, so we just peek at "istio_requests" metric.  The full response contains many more metrics.

### Access the dashboard

1. Start the prometheus dashboard

    ```{.shell .language-shell}
    istioctl dashboard prometheus
    ```

1. In the search field enter the metric named `istio_requests_total`, and click the _Execute_ button (on the right).

1. Select the tab named _Graph_ to obtain a graphical representation of this metric over time.

    Note that you are looking at requests across the entire mesh, i.e. this includes both requests to `web-frontend` and to `customers`.

1. As an example of Prometheus' dimensional metrics capability, we can ask for total requests having a response code of 200:

    ```text
    istio_requests_total{response_code="200"}
    ```

1. With respect to requests, it's more interesting to look at the rate of incoming requests over a time window.  Try:

    ```text
    rate(istio_requests_total[5m])
    ```

There's much more to the Prometheus query language ([this](https://prometheus.io/docs/prometheus/latest/querying/basics/){target=_blank} may be a good place to start).

Grafana consumes these metrics to produce graphs on our behalf.

- Close the Prometheus dashboard and terminate the corresponding `istioctl dashboard` command.

## Grafana

1. Launch the Grafana dashboard

    ```{.shell .language-shell}
    istioctl dashboard grafana
    ```

1. From the sidebar, select _Dashboards_ → _Browse_
1. Click on the folder named _Istio_ to reveal pre-designed Istio-specific Grafana dashboards
1. Explore the Istio Mesh Dashboard.  Note the Global Request Volume and Global Success Rate.
1. Explore the Istio Service Dashboard.  First select the service `web-frontend` and inspect its metrics, then switch to the `customers` service and review its dashboard.
1. Explore the Istio Workload Dashboard.  Select the `web-frontend` workload.  Look at Outbound Services and note the outbound requests to the customers service.  Select the `customers` workload and note that it makes no Oubtound Services calls.

Feel free to further explore these dashboards.

## Cleanup

1. Terminate the `istioctl dashboard` command (++ctrl+c++)
1. Likewise, terminate the `siege` command

## Next

We turn our attention next to traffic shifting features of a service mesh.
