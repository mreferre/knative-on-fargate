
This folder contains the assets that have been pre-created for you to deploy Knative on AWS Fargate. The istructions below detail how these files have been created. These instructions are only useful if you need to re-create them (or want to understand all the details re how they have been created). Otherwise you can just follow the [main instructions](../README.md).

To begin with, you have to generate the YAML file with a dry-run of the `glooctl` command and manually adjust a few things. You can generate the file launching this command: 
```
glooctl install knative --dry-run > knative-gloo-fargate.yaml
```

Open the `knative-gloo-fargate.yaml` file in an editor. Locate the `knative-external-proxy` Kubernetes Service and change its type from `LoadBalancer` to `NodePort`:
```
apiVersion: v1
kind: Service
metadata:
  labels:
    app: gloo
    gloo: knative-external-proxy
  name: knative-external-proxy
  namespace: gloo-system
spec:
  ports:
  - port: 80
    protocol: TCP
    name: http
  - port: 443
    protocol: TCP
    name: https
  selector:
    gloo: knative-external-proxy
  type: NodePort
```

This will make sure that a CLB doesn't get created. However, we now need a way to setup the ingress that leverages ALB. To do so, right after the Service above, copy and paste this entire new section:
```
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: knative-external-proxy
  namespace: gloo-system
  annotations:
    kubernetes.io/ingress.class: alb # check this, your ingress.class may be different 
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip 
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}]'
    alb.ingress.kubernetes.io/load-balancer-attributes: idle_timeout.timeout_seconds=600
  labels:
    app: gloo
    gloo: knative-external-proxy
spec:
  rules:
    - http:
        paths:
          - path: /*
            - backend:
                serviceName: knative-external-proxy
                servicePort: 80
            - backend:
                serviceName: knative-external-proxy
                servicePort: 443
---
```
The above brand new session will expose the `knative-external-proxy` Service through the ALB ingress. For the purpose of this PoC I have only configured the http (port 80) listener.

Note: with the `idle_timeout.timeout_seconds=600` custom attributes we change the timeout for the ALB from 60 seconds (default) to 600 seconds. This is important particularly for the "scale-to-zero" scenarios where Fargate may take more than 60 seconds to run the first pod.

For reasons discussed in [this issue](https://github.com/knative/docs/issues/2255) the file generated by the dry-run doesn't create the `gloo-system` Kubernetes namespace that the remaining of the file assumes exist. Because of this, you need to manually add the following section to the `knative-gloo-fargate.yaml` file: 
```
--- 
apiVersion: v1
kind: Namespace
metadata:
  name: gloo-system
---
```

We are almost there. There is one last tweak we need to do. The `glooctl install knative` command handle race conditions and setup things in the proper order. When you export the YAML file with the --dry-run option and attempt to run it in its entirety in a single apply, some of the objects will spit an error at creation because (presumably) other objects they are dependent on are not yet ready. Because of this we need to split the `knative-gloo-fargate.yaml` file into two distinct files that we will apply at different times: 
- `knative-gloo-fargate-first-batch.yaml` contains everything but objects of kind `Image` , `Gateway` and `Settings` 
- `knative-gloo-fargate-second-batch.yaml` contains only objects of kind `Image` , `Gateway` and `Settings` 

These are the two files that are shipped with the repo. You don't need to re-create them but now you know how they have been built. 

Go back to the [main istructions](../README.md).