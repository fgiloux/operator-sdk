---
title: Quickstart for Go-based Operators
linkTitle: Quickstart
weight: 20
description: A simple set of instructions to set up and run a Go-based operator.
---

This guide walks through an example of building a simple memcached-operator using tools and libraries provided by the Operator SDK.

## Prerequisites

- Go through the [installation guide][install-guide].
- User authorized with `cluster-admin` permissions.
- An accessible image registry for various operator images (ex. [hub.docker.com](https://hub.docker.com/signup),
[quay.io](https://quay.io/)) and be logged in in your command line environment.
  - `example.com` is used as the registry Docker Hub namespace in these examples.
  Replace it with another value if using a different registry or namespace.
  - The registry/namespace must be public, or the cluster must be provisioned with an
  [image pull secret][k8s-image-pull-sec] if the image namespace is private.


## Steps

1. Create a project directory for your project and initialize the project:

  ```sh
  mkdir memcached-operator
  cd memcached-operator
  operator-sdk init --domain example.com --repo github.com/example/memcached-operator
  ```

1. Create a simple Memcached API:

  ```sh
  operator-sdk create api --group cache --version v1alpha1 --kind Memcached --resource --controller
  ```

1. Build and push your operator's image:

  ```sh
  make docker-build docker-push IMG="example.com/memcached-operator:v0.0.1"
  ```

### OLM deployment

1. Install [OLM][doc-olm]:

  ```sh
  operator-sdk olm install
  ```

1. Bundle your operator, then build and push the bundle image (defaults to `example.com/memcached-operator-bundle:v0.0.1`):

  ```sh
  make bundle IMG="example.com/memcached-operator:v0.0.1"
  make bundle-build bundle-push
  ```

1. Run your bundle. If your bundle image is hosted in a private registry,
add the image pull secret for that registry host to the service account in use
and set `--secret-name` to the secret name:
<!-- TODO(estroz): remove the service account requirement once OLM releases a patch or new
minor release containing https://github.com/operator-framework/operator-lifecycle-manager/pull/1941 -->

  ```sh
  kubectl patch serviceaccount default -p '{"imagePullSecrets":[{"name":"<reg secret name>"}]}'
  operator-sdk run bundle example.com/memcached-operator-bundle:v0.0.1
  ```

1. Create a sample Memcached custom resource:

  ```console
  $ kubectl apply -f config/samples/cache_v1alpha1_memcached.yaml
  memcached.cache.example.com/memcached-sample created
  ```

1. Uninstall the operator:

  ```sh
  operator-sdk cleanup memcached-operator
  ```


### Direct deployment

1. Deploy your operator:

  ```sh
  make deploy IMG="example.com/memcached-operator:v0.0.1"
  ```

1. Create a sample Memcached custom resource:

  ```console
  $ kubectl apply -f config/samples/cache_v1alpha1_memcached.yaml
  memcached.cache.example.com/memcached-sample created
  ```

1. Uninstall the operator:

  ```sh
  make undeploy
  ```


## Next Steps

Read the [full tutorial][tutorial] for an in-depth walkthough of building a Go operator.


[install-guide]:/docs/building-operators/golang/installation
[doc-olm]:/docs/olm-integration/quickstart-bundle/#enabling-olm
[tutorial]:/docs/building-operators/golang/tutorial/
[k8s-image-pull-sec]:https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/
