## Deployment

The base folder contains common resources used in all namespaces. 

The overlays folder contains resources specific to an environment/namespace, routes for example.

The folders and included resources are expected to be deployed/managed, in the first instance, by Kustomize.


### Brief description of Kustomize 

Kustomize is a command line tool supporting template-free, structured customization of declarative configuration targeted to k8s-style objects.

Targeted to k8s means that kustomize has some understanding of API resources, k8s concepts like names, labels, namespaces, etc. and the semantics of resource patching.

https://kustomize.io/
