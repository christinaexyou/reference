# Configuring the OpenTelemetry Collector

## Pre-requisites

* Install the Red Hat OpenShift distributed tracing platform (Jaegar)
* Install the Red Hat build of OpenTelemetry

1. Create a new project, for example, `my-otel-demo`
    ```
    oc create project my-otel-demo
    ```

2. Create a Jaegar instance

    ```
    $ cat <<EOF |oc apply -f -
    apiVersion: jaegertracing.io/v1
    kind: Jaeger
    metadata:
    name: my-jaeger
    spec: {}
    EOF
    jaeger.jaegertracing.io/my-jaeger created
    ```

3. Sanity check the Jaegar instance by checking the route:
    ```
    oc get routes my-jaeger
    ```
    Open a new browser window and go to the route URL and login with your OpenShift credentials


   Sanity check the list of Jaegar services:
   ```
   oc get svc | grep jaegar
   ```

   Expected output:
   ```
   my-jaegar-agent                            ClusterIP      None             <none>                                                5775/UDP,5778/TCP,6831/UDP,6832/UDP,14271/TCP                        144m
    my-jaegar-collector                        ClusterIP      172.30.243.189   <none>                                                9411/TCP,14250/TCP,14267/TCP,14268/TCP,14269/TCP,4317/TCP,4318/TCP   144m
    my-jaegar-collector-headless               ClusterIP      None             <none>                                                9411/TCP,14250/TCP,14267/TCP,14268/TCP,14269/TCP,4317/TCP,4318/TCP   144m
    my-jaegar-query                            ClusterIP      172.30.237.229   <none>                                                443/TCP,16685/TCP,16687/TCP                                          144m
   ```

4. Create the OpenTelemetry Collector:

   (a) Create a configmap that generates TLS web server certificates

   (b) Create a OpenTelemetry Collector instance

   ```
   apiVersion: v1
    kind: ConfigMap
    metadata:
    annotations:
        service.beta.openshift.io/inject-cabundle: "true"
    name: my-otelcol-cabundle
    ---
    kind: OpenTelemetryCollector
    apiVersion: opentelemetry.io/v1beta1
    metadata:
    name: my-otelcol
    namespace: nemo-test
    spec:
    config:
        exporters:
        debug: {}
        otlp:
            endpoint: "my-jaegar-collector-headless.nemo-test.svc.cluster.local:14250"
            tls:
            ca_file: /etc/pki/ca-trust/source/service-ca/service-ca.crt
        receivers:
        otlp:
            protocols:
            grpc: {}
            http: {}
        service:
        pipelines:
            traces:
            exporters:
                - debug
                - otlp
            receivers:
                - otlp
    mode: deployment
    resources: {}
    targetAllocator: {}
    volumeMounts:
    - mountPath: /etc/pki/ca-trust/source/service-ca
        name: cabundle-volume
    volumes:
    - configMap:
        name: my-otelcol-cabundle
        name: cabundle-volume
   ```

5. Sanity check the OpenTelemetry Collector instance by checking the pods' logs

    (a) Retreive the OpenTelemtry pods:

        oc get pods | grep my-otelcol

     Expected output:

        my-otelcol-collector-77d97c47f6-gnqvm               1/1     Running            0               4h32m
        my-otelcol-tls-test-collector-54b4cd6c7d-d965l      1/1     Running            0               4h9m


    (b) Retreive the logs for `my-otel-collector-xxx`:

        oc logs my-otelcol-collector-77d97c47f6-gnqvm -c otc-container


    Expected output:
    ```
    ...
    2025-07-31T13:17:05.411Z        info    service@v0.127.0/service.go:289 Everything is ready. Begin running and processing data. {"resource": {}}
    ```


