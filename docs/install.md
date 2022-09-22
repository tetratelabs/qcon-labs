# Install Istio

In this lab you will install Istio.


## Download Istio

1. Run the following command from your home directory.

    ```{.shell .language-shell}
    curl -L https://istio.io/downloadIstio | ISTIO_VERSION={{istio.version}} sh -
    ```

1. Navigate into the directory created by the above command.

    ```{.shell .language-shell}
    cd istio-{{istio.version}}
    ```


## Add `istioctl` to your PATH

The `istioctl` CLI is located in the `bin/` subdirectory.

!!! note

    Cloud Shell only preserves files located inside your home directory across sessions.

    This means that if you install a binary to a `PATH` such as `/usr/local/bin`, after your session times out that file will no longer be there!

    As a workaround, you will add `${HOME}/bin` to your `PATH` and place the binary there.


1. Create a `bin` subdirectory in your home directory:

    ```{.shell .language-shell}
    mkdir ~/bin
    ```

1. Copy the CLI to that subdirectory:

    ```{.shell .language-shell}
    cp ./bin/istioctl ~/bin
    ```

1. Add your home `bin` subdirectory to your `PATH`

    ```shell
    cat << EOF >> ~/.bashrc

    export PATH="~/bin:\$PATH"

    EOF
    ```

    And then:

    ```shell
    source ~/.bashrc
    ```

Verify that `istioctl` is installed with:

```{.shell .language-shell}
istioctl version
```

The output should indicate that the version is {{istio.version}}.

With the CLI installed, proceed to install Istio to Kubernetes.

## Pre-check

The `istioctl` CLI provides a convenient `precheck` command that can be used to "_inspect a Kubernetes cluster for Istio install and upgrade requirements._"

To verify whether it is safe to install Istio on your Kubernetes cluster, run:

```shell
istioctl x precheck
```

Make sure that the output of the above command returns a green "checkmark" stating that no issues were found when checking the cluster.

## Install Istio

Take a moment to learn about [Istio installation profiles](https://istio.io/latest/docs/setup/additional-setup/config-profiles/){target=_blank}.

Choose your journey:

=== "Simple Recipe"

    This installation recipe uses the `default` profile, which installs both `istiod` and `istio-ingressgateway` components in a single step.

    1. Istio can be installed directly with the CLI:

        ```{.shell .language-shell}
        istioctl install
        ```

    1. When prompted, enter `y` to proceed to install Istio.

=== "Separate Gateway Recipe"

    In production settings, it is preferrable to install the Istio gateway component separately, providing the flexibility of upgrading each component independently.

    This full procedure can be found [in the Istio documentation](https://istio.io/latest/docs/setup/additional-setup/gateway/){target=_blank}.

    ### Install Istio by itself

    ```shell
    istioctl install --set profile=minimal
    ```

    ### Install the ingress gateway separately

    1. Review the `ingress.yaml` installation configuration (included in `artifacts/`)

        ???+ tldr "install/ingress.yaml"
            ```yaml linenums="1" hl_lines="7 11 16"
            --8<-- "install/ingress.yaml"
            ```

        Note:

        1. The use of the _empty_ profile, with the ingress gateway component added "a la carte."
        1. For simplicity, the gateway is installed to the `istio-system` namespace.
        1. The assignment of a label to the gateway deployment (so that later, we can define `Gateway` custom resources that target this component).

    1. Install the Gateway component

        ```shell
        istioctl install -f ingress.yaml
        ```

## Verify that Istio is installed

Post-installation, Istio provides the command `verify-install`: it runs a series of checks to ensure that the installation was successful and complete.

Go ahead and run it:

```shell
istioctl verify-install
```

Inspect the output and confirm that the it states that "_✔ Istio is installed and verified successfully._"

Keep probing:

1. List Kubernetes namespaces and note the new namespace `istio-system`

    ```{.shell .language-shell}
    kubectl get ns
    ```

1. Verify that the `istiod` controller pod is running in that namespace

    ```{.shell .language-shell}
    kubectl get pod -n istio-system
    ```

1. Re-run `istioctl version`.  The output should include a _control plane_ version, indicating that Istio is indeed present in the cluster.

## Next

With Istio installed, we are ready to deploy an application to the mesh.
