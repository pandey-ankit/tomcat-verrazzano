# Deploy Tomcat Application on Verrazzano

## Introduction

This lab walks you through the process of deploying the tomcat sample application docker image.

Estimated Time: 10 minutes

### Verrazzano and Application Deployment

Verrazzano supports application definition using [Open Application Model (OAM)](https://oam.dev/). Verrazzano applications are composed of components and application configurations.

When you deploy applications with Verrazzano, the platform sets up connections, and network policies, ingresses in the service mesh, and wires up a monitoring stack to capture the metrics, logs, and traces. Verrazzano employs OAM components to define the functional units of a system that are then assembled and configured by defining associated application configurations.

### Verrazzano components

A Verrazzano OAM component is a [Kubernetes Custom Resource](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) describing an application’s general composition and environment requirements.

The following code shows a simple Helidon application component for Helidon *quickstart-mp* application used in this lab. This resource describes a component which is implemented by a single Docker image containing a Helidon application exposing a single endpoint.

```yaml
apiVersion: core.oam.dev/v1alpha2
kind: Component
metadata:
  name: tomcat-component
spec:
  workload:
    apiVersion: core.oam.dev/v1alpha2
    kind: ContainerizedWorkload
    metadata:
      name: tomcat-workload
      labels:
        app: tomcat
        version: v2
    spec:
      containers:
      - name: tomcat-container
        image: "ENDPOINT_OF_REGION/TENANCY_NAMESPACE/tomcat-example-your_firstname:v1"
        ports:
          - containerPort: 8080
            name: ecnj
```

A brief description of each field of the component:

* **apiVersion** - Version of the component custom resource definition
* **kind** - Standard name of the component custom resource definition
* **metadata.name** - The name used to create the component’s custom resource
* **spec.workload.kind** - VerrazzanoHelidonWorkload defines a stateless workload of Kubernetes
* **spec.workload.spec.deploymentTemplate.podSpec.containers.ports** - Ports exposed by the container

### Verrazzano Application Configurations

A Verrazzano application configuration is a Kubernetes Custom Resource which provides environment-specific customizations. The following code shows the application configuration for the Helidon *quickstart-mp* example used in this lab. This resource specifies the deployment of the application to the hello-helidon namespace.

Additional runtime features are specified using traits, or runtime overlays that augment the workload. For example, the ingress trait specifies the ingress host and path, while the metrics trait provides the Prometheus scraper used to obtain the application-related metrics.

```yaml
apiVersion: core.oam.dev/v1alpha2
kind: ApplicationConfiguration
metadata:
  name: tomcat-appconf
  annotations:
    version: v1.0.0
    description: "Verrazzano demo application"
spec:
  components:
    - componentName: tomcat-component
      traits:
        - trait:
            apiVersion: oam.verrazzano.io/v1alpha1
            kind: MetricsTrait
            spec:
              scraper: verrazzano-system/vmi-system-prometheus-0
        - trait:
            apiVersion: oam.verrazzano.io/v1alpha1
            kind: IngressTrait
            metadata:
              name: tomcat-ingress
            spec:
              rules:
                - paths:
                    - path: "/"
                      pathType: Prefix
        - trait:
            apiVersion: core.oam.dev/v1alpha2
            kind: ManualScalerTrait
            spec:
              replicaCount: 2
```

A brief description of each field in the application configuration:

* **apiVersion** - Version of the ApplicationConfiguration custom resource definition
* **kind** - Standard name of the application configuration custom resource definition
* **metadata.name** - The name used to create this application configuration resource
* **spec.components** - Reference to the application’s components leveraged to specify runtime configuration
* **spec.components[].traits** - The traits specified for the application’s components

To explore traits, we can examine the fields of an ingress trait:

* **apiVersion** - Version of the OAM trait custom resource definition
* **kind** - IngressTrait is the name of the OAM application ingress trait custom resource definition
* **spec.rules.paths** - The context paths for accessing the application



### Objectives

In this lab, you will:

* Deploy the tomcat sample application.
* Verify the deployment of the tomcat sample application.

### Prerequisites

To run this lab, you must have:

* Kubernetes (OKE) cluster running on the Oracle Cloud Infrastructure.
* Verrazzano installation started on a Kubernetes (OKE) cluster.
* Container packaged *tomcat-example-your_firstname* application available in a container registry.



## Task 1: Deploy the Tomcat sample application 


1. Create a `tomcat-ns` namespace for the tomcat sample application. We will keep all Kubernetes artefacts in a separate namespace.

    ```bash
    <copy>kubectl create namespace tomcat-ns</copy>
    ```

    >Namespaces are a way to organize clusters into virtual sub-clusters. We can have any number of namespaces within a cluster, each logically separated from others but with the ability to communicate with each other.

2. We need to make Verrazzano aware that we store in that namespace Verrazzano artifacts. So we need to add a label identifying the `tomcat-ns` namespace as managed by Verrazzano. Labels are intended to be used to specify identifying attributes of objects that are meaningful and relevant to users.

    Here, for the `tomcat-ns` namespace, we are attaching a label to it, which marks this namespace as managed by Verrazzano. The *istio-injection=enabled*, enables an Istio "sidecar", and as such, helps establish an Istio proxy. With an Istio proxy, we can access other Istio services like an Istio gateway. To add the label to the `tomcat-ns` namespace with the previously mentioned attributes, copy the following command and run it in the Cloud Shell:

    ```bash
    <copy>kubectl label namespace tomcat-ns verrazzano-managed=true istio-injection=enabled</copy>
    ```


3. In this location, we have the configuration file for the bobs-books application. As part of Lab 5, we modified bobbys-helidon-stock-application and built a new Docker image for that component. In Lab 6, we pushed that Docker image to the Oracle Cloud Container Registry repository. Now, in this lab, we will modify the *bobs-books-comp.yaml* file so that it takes the new updated Docker image from the Oracle Cloud Container Registry repository. To modify the *bobs-books-comp.yaml* file, copy the following command and paste it in the *Cloud Shell*.

    ```bash
    <copy>vi bobs-books-comp.yaml</copy>
    ```

    ![Open file](images/openfile.png " ")

4. As part of Lab 5, you saved your Docker image full name. You need to copy the following line and paste it in your text editor. Then, you need to replace `docker image full name` with your Docker image name. Then copy the modified line and press *i* to insert the text in the `*bobs-books-comp.yaml*` file. Paste the output at line number 148 (make sure you keep the indentation) and comment out the exiting line number 147 with *#* as shown in the following image, then press *Esc* and then type *:wq* to save the file.

    ```bash
    <copy>image:  `docker image full name`</copy>
    ```

    ![Insert line](images/insert-line.png " ") 

5. Now, we want to deploy tomcat containerized application on *cluster1*. For this, we need a Kubernetes deployment configuration. This deployment instructs the Kubernetes to create and update instances for the tomcat application. Here, we have the `tomcat-comp.yaml` file, which instructs Kubernetes.

    To deploy the tomcat sample application, copy and paste the following two commands as shown. The `tomcat-comp.yaml` file contains definitions of various OAM components, where, an OAM component is a Kubernetes Custom Resource describing an application’s general composition and environment requirements.
    ```bash
    <copy>kubectl apply -f ~/example/tomcat-comp.yaml -n tomcat-ns</copy>
    ```
    The `tomcat-app.yaml` file is a Verrazzano application configuration file, which provides environment-specific customizations.
    ```bash
    <copy>kubectl apply -f ~/example/tomcat-app.yaml -n tomcat-ns</copy>
    ```

6. Wait for the pods to be in *Running* status. Use this *kubectl* command to wait for all the pods to be in the *Running* state within the hello-helidon namespace. It takes around 1-2 minutes.

    ```bash
    <copy>kubectl wait --for=condition=Ready pods --all -n tomcat-ns --timeout=600s</copy>
    ```

    When the pods are ready you can see a similar response:

    ```bash
    $ kubectl wait --for=condition=Ready pods --all -n tomcat-ns --timeout=600s
    pod/hello-helidon-deployment-58fdd5cd4-94wjf condition met
    ```
    You can also list the pods directly to check their status:

    ```bash
    $ kubectl  get po -n tomcat-ns
    NAME                                       READY   STATUS    RESTARTS   AGE
    hello-helidon-deployment-58fdd5cd4-94wjf   2/2     Running   0          34m
    ```



## Task 2: Verify the Successful Deployment of the Hello Helidon Application

1. Verify the `/greet` endpoint. To determine the URL that was constructed from the external/load balancer IP and application configuration, execute the following command:

    ```bash
    <copy>echo https://$(kubectl get gateways.networking.istio.io hello-helidon-hello-helidon-gw -n hello-helidon -o jsonpath={.spec.servers[0].hosts[0]})/greet</copy>
    ```

    This will print the proper URL to your REST endpoint, for example:

    ```bash
    https://hello-helidon.hello-helidon.xx.xx.xx.xx.nip.io/greet
    ```

2. Use this link to test from your browser. Due to self-signed certificates, however, you need to accept risk and allow the browser to continue the request processing.

    You may find it easier to use `curl` because the response is only a string:

    ```bash
    <copy>curl -k https://$(kubectl get gateways.networking.istio.io hello-helidon-hello-helidon-gw -n hello-helidon -o jsonpath={.spec.servers[0].hosts[0]})/greet; echo</copy>
    ```

    You should see the same result you received during the development:


    ```yaml
    {"message":"Hello World!"}
    ```

You may now **proceed to the next lab**.



## Acknowledgements

* **Author** -  Ankit Pandey
* **Contributors** - Maciej Gruszka, Sid Joshi
* **Last Updated By/Date** - Ankit Pandey, March 2023