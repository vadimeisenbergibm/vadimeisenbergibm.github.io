---
title: Setup Kubernetes cluster
overview: Set up your Kubernetes cluster for the tutorial.
weight: 2
---

{{< boilerplate work-in-progress >}}

In this module you set up a Kubernetes cluster with Istio installed and with a dedicated namespace for the tutorial.

{{< warning >}}
In case you participate in a workshop and the instructors provide a cluster for you,
proceed to [the next module](/docs/examples/tutorial/setup-local-computer).
{{</ warning >}}

1.  Ensure you have access to a [Kubernetes cluster](https://kubernetes.io/docs/tutorials/kubernetes-basics/).
    You can try using the [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine/docs/quickstart) or the
    [IBM Cloud Kubernetes Service](https://cloud.ibm.com/docs/containers?topic=containers-getting-started).

1.  Create an environment variable to store the name of a namespace to perform the commands of the tutorial on.
    You can use any name, for example `tutorial`. `coolstuff` will do as well.

    {{< text bash >}}
    $ export NAMESPACE=tutorial
    {{< /text >}}

1.  Create the namespace:

    {{< text bash >}}
    $ kubectl create namespace $NAMESPACE
    {{< /text >}}

    {{< tip >}}
    If you run the tutorial for multiple participants (you are an instructor in a class or you want to complete this
    tutorial with your friends using your cluster), you may want to allocate a separate namespace per each
    participant. The tutorial supports work in multiple namespaces simultaneously by multiple participants.
    {{< /tip >}}

1.  Install Istio with strict mutual TLS enabled. The simplest way is to follow
    [these instructions](/docs/setup/kubernetes/install/kubernetes/#installation-steps),
    select the `strict mutual TLS` tab.

1.  [Enable Envoy's access logging](/docs/tasks/telemetry/logs/access-log/#enable-envoy-s-access-logging).

1.  Create a Kubernetes Ingress resource for common Istio services:

    - [Grafana](https://grafana.com/docs/guides/getting_started/)
    - [Jaeger](https://www.jaegertracing.io/docs/1.13/getting-started/)
    - [Prometheus](https://prometheus.io/docs/prometheus/latest/getting_started/)
    - [Kiali](https://www.kiali.io/documentation/getting-started/)

    If you do not know what are the services above - great, this tutorial will teach you about each of them.

    {{< text bash >}}
    $ kubectl apply -f - <<EOF
    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
      name: istio-system
      namespace: istio-system
    spec:
      rules:
      - host: my-istio-dashboard.io
        http:
          paths:
          - path: /
            backend:
              serviceName: grafana
              servicePort: 3000
      - host: my-istio-tracing.io
        http:
          paths:
          - path: /
            backend:
              serviceName: tracing
              servicePort: 80
      - host: my-istio-logs-database.io
        http:
          paths:
          - path: /
            backend:
              serviceName: prometheus
              servicePort: 9090
      - host: my-kiali.io
        http:
          paths:
          - path: /
            backend:
              serviceName: kiali
              servicePort: 20001
    EOF
    {{< /text >}}

1.  You may want to limit the permissions of each participant so they will be able to create resources only in their
    namespace. Even if you run the tutorial alone on your own cluster, it could be a good practice to limit yourself so
    your learning will not interfere with other namespaces in your cluster.

    Perform the following steps to achieve this:

    1.  Create a role to provide read access to `istio-system` namespace:

        {{< text bash >}}
        $ kubectl apply -f - <<EOF
        kind: Role
        apiVersion: rbac.authorization.k8s.io/v1beta1
        metadata:
          name: istio-system-access
          namespace: istio-system
        rules:
        - apiGroups: ["", "extensions", "apps"]
          resources: ["*"]
          verbs: ["get", "list"]
        EOF
        {{< /text >}}

    1.  Create a service account for the user of the namespace and provide access to the namespace's resources to that
    service account:

        {{< text bash >}}
        $ kubectl apply -f - <<EOF
        apiVersion: v1
        kind: ServiceAccount
        metadata:
          name: ${NAMESPACE}-user
          namespace: $NAMESPACE
        ---
        kind: Role
        apiVersion: rbac.authorization.k8s.io/v1beta1
        metadata:
          name: ${NAMESPACE}-access
          namespace: $NAMESPACE
        rules:
        - apiGroups: ["", "extensions", "apps", "networking.k8s.io", "networking.istio.io", "authentication.istio.io",
                      "rbac.istio.io", "config.istio.io"]
          resources: ["*"]
          verbs: ["*"]
        ---
        kind: RoleBinding
        apiVersion: rbac.authorization.k8s.io/v1beta1
        metadata:
          name: ${NAMESPACE}-access
          namespace: $NAMESPACE
        subjects:
        - kind: ServiceAccount
          name: ${NAMESPACE}-user
          namespace: $NAMESPACE
        roleRef:
          apiGroup: rbac.authorization.k8s.io
          kind: Role
          name: ${NAMESPACE}-access
        ---
        kind: RoleBinding
        apiVersion: rbac.authorization.k8s.io/v1beta1
        metadata:
          name: ${NAMESPACE}-istio-system-access
          namespace: istio-system
        subjects:
        - kind: ServiceAccount
          name: ${NAMESPACE}-user
          namespace: $NAMESPACE
        roleRef:
          apiGroup: rbac.authorization.k8s.io
          kind: Role
          name: istio-system-access
        EOF
        {{< /text >}}

    1.  Generate the `./${NAMESPACE}-user-config.yaml` Kubernetes configuration file needed for each user of each
        namespace:

        {{< text bash >}}
        $ cat <<EOF > ./${NAMESPACE}-user-config.yaml
        apiVersion: v1
        kind: Config
        preferences: {}

        clusters:
        - cluster:
            certificate-authority-data: $(kubectl get secret $(kubectl get sa ${NAMESPACE}-user -n $NAMESPACE -o jsonpath={.secrets..name}) -n $NAMESPACE -o jsonpath='{.data.ca\.crt}')
            server: $(kubectl config view -o jsonpath="{.clusters[?(.name==\"$(kubectl config view -o jsonpath="{.contexts[?(.name==\"$(kubectl config current-context)\")].context.cluster}")\")].cluster.server}")
          name: ${NAMESPACE}-cluster

        users:
        - name: ${NAMESPACE}-user
          user:
            as-user-extra: {}
            client-key-data: $(kubectl get secret $(kubectl get sa ${NAMESPACE}-user -n $NAMESPACE -o jsonpath={.secrets..name}) -n $NAMESPACE -o jsonpath='{.data.ca\.crt}')
            token: $(kubectl get secret $(kubectl get sa ${NAMESPACE}-user -n $NAMESPACE -o jsonpath={.secrets..name}) -n $NAMESPACE -o jsonpath={.data.token} | base64 --decode)

        contexts:
        - context:
            cluster: ${NAMESPACE}-cluster
            namespace: ${NAMESPACE}
            user: ${NAMESPACE}-user
          name: ${NAMESPACE}

        current-context: ${NAMESPACE}
        EOF
        {{< /text >}}

        Now you can send the generated config file to each tutorial's participant (or use it yourself, if you are the participant). The participant will define the `KUBECONFIG` environment variable to store the location of the
        file, and as a result will have access to the tutorial's namespace only.
        See [the next module](/docs/examples/tutorial/setup-local-computer) for the local computer setup.

You performed the setup of your cluster for the tutorial and can proceed to the setup of your local computer in
[the next module](/docs/examples/tutorial/setup-local-computer).
We understand that for some of you this setup process could be rather boring.
Bear with us, you are just a single module before running your first microservice!