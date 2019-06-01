---
title: A/B testing with Istio
overview: Deploy two different versions of a microservice and split the traffic between them.

weight: 101

---

In the previous module you deployed a new version of the `reviews` microservice, _v3_. This version returns ratings as
red stars, as opposed to the black stars of _reviews v2_.

While you performed a successful rollout of the new version, how can you be sure that the new version is actually
better than the previous one?
You may want to perform _A/B_ testing: to run two versions of the `reviews` microservice, _v2_ and _v3_, splitting the
requests 50:50. Then you will measure by various metrics which version is accepted better by the users.
(Measuring business metrics is out of scope of Istio). In this module you redeploy _reviews v2_ and configure Istio
to split the traffic destined to `reviews` equally between the _v2_ and _v3_ versions.

1.  Deploy _reviews v2_:

    {{< text bash >}}
    $ kubectl apply -l app=reviews,version=v2 -f {{< github_file >}}/samples/bookinfo/platform/kube/bookinfo.yaml
    deployment "reviews-v2" created
    {{< /text >}}

1.  Configure a virtual service to distribute the traffic 50:50 between _reviews v2_ and _reviews v3_.

    {{< text bash >}}
    $ kubectl apply -f - <<EOF
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: reviews
    spec:
      hosts:
        - reviews
      http:
      - match:
        - headers:
            user-agent:
              regex: ".*Firefox.*"
        route:
        - destination:
            host: reviews
            subset: v2
      - route:
        - destination:
            host: reviews
            subset: v3
    EOF
    {{< /text >}}

1.  Access application's webpage in Firefox and in some other browser, and verify that now the red stars are displayed
    in the other browser and black stars in Firefox. Alternatively, check the access by `curl`:

    {{< text bash >}}
    $ curl -s $MY_INGRESS_GATEWAY_HOST:$INGRESS_PORT/productpage  -H"User-Agent: Safari" | grep color | uniq
     <font color="red">
    {{< /text >}}

    {{< text bash >}}
    $ curl -s $MY_INGRESS_GATEWAY_HOST:$INGRESS_PORT/productpage  -H"User-Agent: Firefox" | grep color | uniq
     <font color="black">
    {{< /text >}}

1.  Check your Kiali console,
    [http://my-kiali.io/kiali/console](http://my-kiali.io/kiali/console), the graph of your namespace.