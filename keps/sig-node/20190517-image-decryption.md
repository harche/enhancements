---
title: Adding the Support for Encrypted Images
authors:
  - "@harche"
owning-sig: sig-node
participating-sigs:
  - sig-architecture
reviewers:
  - smarterclayton
  - tallclair
  - yujuhong
approvers:
  - smarterclayton
  - tallclair
creation-date: 2019-05-16
status: provisional
---

# Adding the Support for Encrypted Images

## Table of Contents

* [Summary](#summary)
* [Motivation](#motivation)
  * [Goals](#goals)
  * [Non\-Goals](#non-goals)
  * [User Stories](#user-stories)
* [Proposal](#proposal)
  * [API](#api)
    * [Image Handler](#Image-handler)
  * [Relationship with imagePullPolicy](#Relationship-with-imagePullPolicy)
  * [Consumption of the ImageDecryptSecrets](#Consumption-of-the-ImageDecryptSecrets)
* [Alternatives Considered](#alternatives-considered)
* [Graduation Criteria](#graduation-criteria)
* [Implementation History](#implementation-history)

## Summary

The underlying specification for the containers, the OCI spec, is soon going to support encrypted images. Kubernetes should be able to support decryption of these encrypted images with the addition of a new type of `Secret`, which we would like to call `ImageDecryptSecret`. 

Along with OCI spec, there is an ongoing effort to enable the support for encrypted images in containerd. 

OCI Spec Issue - https://github.com/opencontainers/image-spec/issues/747

OCI Spec PR - https://github.com/opencontainers/image-spec/pull/775

POC - https://github.com/harche/kubernetes/commit/8a0ef9898a55110d7e41f8e1a846c66cfc2fa691

POC Demo - https://youtu.be/S3FK4y5McOk


## Motivation

The kubernetes worker nodes are where container images are pulled by a runtime such as `containerd`. If the images pulled are encrypted then containerd will have to decrypt them before running the pod. In order to be able to decrypt the images, the worker node needs to have access to corresponding the private keys.

Kubernetes `Secrets` are used to securely deliver sensitive data to pods in corresponding worker nodes. We need to have a secret that can be utilized *before* the pod is provisioned. Regular Kubernetes secret gets attached to the pod as tmpfs mount after the pod is started. However, there exists another type of kubernetes secret called `ImagePullSecrets`. ImagePullSecrets are used to pull the images from the private container image registry, hence they contain login credentials. `ImagePullSecrets` get utilized *before* the pod is started, this is exactly the same kind of requirement for being able to decrypt the encrypted container image. We need to have a `secret` that can be used *before* the pod is provisioned to decrypt the image. Hence, we are submitting this KEP to kubernetes community to propose a new type of kubernetes secret that is modeled after `ImagePullSecret`, called `ImageDecryptSecret`. While the `ImagePullSecret` holds the login credentials for the private registry, the `ImageDecryptSecret` will hold the private keys to decrypt encrypted images.


### Goals

- Introduce a new type of secret, `ImageDecryptSecret` to represent the necessary key(s) required to decrypt the contents of the image. 
- Define how `ImageDecryptSecret` can be used by configuration yaml(s) of the Pod (or Deployments)
- Define how `ImageDecryptSecret` can be integrated into the service accounts
- Define the Image Authorization process to prevent unauthorized access to the cached encrypted images. 


### Non-Goals

- Kubernetes should be able to decrypt the images. However, in a typical workflow kubernetes has no role to play to encrypt the images. This is similar to how kubernetes plays a role in downloading and using the images instead of building and uploading container images to the registry.  

### User Stories

- As a cluster user, I want to create a secret that carries the private keys required for my encrypted images
- As a cluster user, I want to run the encrypted container images using the private keys carried by ImageDecryptSecret
- As a cluster operator, I want to add the ImageDecryptSecret to the service account
- As a cluster user, I want to run the encrypted images using the private keys from the service account
- As an application developer, I want to encrypt my container images and be able to run them securely using kubernetes
- As an application developer, I want to protect the content of my container images such that only me (as an application developer) and the execution runtime can read them. Any other third party, such as container registry, should **NOT** be able to read the content of my container image.

## Proposal

The initial design includes:

- `ImageDecryptSecret` API resource definition
- `ImageDecryptSecret` pod field for specifying the ImageDecryptSecret the pod should be run with
- Kubelet implementation for fetching & interpreting the ImageDecryptSecret
- CRI API & implementation for passing along the ImageDecryptSecret

### API

`ImageDecryptSecret` is a new cluster-scoped Secret


The private key is selected by the pod by specifying the ImageDecryptSecret in the PodSpec. Once the pod is
scheduled, the ImageDecryptSecret cannot be changed.

_(This is a simplified declaration, syntactic details will be covered in the API PR review)_

```go
// Container represents a single container that is expected to be run on the host.
type Container struct {
  <snip>
  // ImageDecryptSecrets is an optional list of references to secrets in the same namespace to use for decrypting any of the images used by this PodSpec.
  // If specified, these secrets will be passed to individual puller implementations for them to use.
  // +optional
  ImageDecryptSecrets []LocalObjectReference
  </snip>
}
```
[Reference Implementation](https://github.com/harche/kubernetes/blob/8a0ef9898a55110d7e41f8e1a846c66cfc2fa691/pkg/apis/core/types.go#L1915) of the `Container` struct

```go
const (
  <snip>
  // SecretTypeDecryptKeys defines the type for the decrypt secrets
  SecretTypeDecryptKey SecretType = "kubernetes.io/decryptionkey"

  // ImageDecryptionKeys represent the key required to access secret data
  ImageDecryptionKey = ".imagedecryptionkey"
  </snip>
)
```
[Reference Implementation](https://github.com/harche/kubernetes/blob/8a0ef9898a55110d7e41f8e1a846c66cfc2fa691/staging/src/k8s.io/api/core/v1/types.go#L4986) for the `constant` that hold the actual private key required for decrypting the image


In order to support ImageDecryptSecret using service account, we need to define a struct that holds the ImageDecryptSecret and the corresponding image(s) that can be decrypted.


```go
// ImageDecryptServiceSecret holds the name of the decrypt secret and the corresponding list of images that can be decrypted using that secret.
type ImageDecryptServiceSecret struct {
	// Required.
	Name string `json:"name,omitempty" protobuf:"bytes,1,opt,name=name"`
	// Required.
	Images []string `json:"images,omitempty" protobuf:"bytes,8,rep,name=images"`
}
```
See the [reference implementation](https://github.com/harche/kubernetes/blob/8a0ef9898a55110d7e41f8e1a846c66cfc2fa691/staging/src/k8s.io/api/core/v1/types.go#L3627) for `ImageDecryptServiceSecret`

```go
type ServiceAccount struct {
  <snip>
  // ImageDecryptSecrets is a list of references to secrets in the same namespace to use for decrypting any encrypted images
  // in pods that reference this ServiceAccount. ImageDecryptSecrets are distinct from Secrets because Secrets
  // can be mounted in the pod, but ImageDecryptSecrets are only accessed by the kubelet.
  // +optional
  ImageDecryptSecrets []ImageDecryptServiceSecret `json:"imageDecryptSecrets,omitempty" protobuf:"bytes,5,rep,name=imageDecryptSecrets"`
  </snip>
```
See the [reference implementation](https://github.com/harche/kubernetes/blob/8a0ef9898a55110d7e41f8e1a846c66cfc2fa691/staging/src/k8s.io/api/core/v1/types.go#L3668) for modification to the `ServiceAccount` struct


An unspecified `nil` or empty `""` ImageDecryptSecret is equivalent to the backwards-compatible
default behavior as if the ImageDecryptSecret feature is disabled.

#### Examples

Suppose we operate a cluster that lets users create a secret of type `ImageDecryptSecret`. 

For the kubectl command line, we propose that the user create a secret of type `image-decrypt` and give it a name. One or multiple private keys may then be stored under this secret.

```bash
kubectl create secret image-decrypt <secret name> --decrypt-secret=<private key>[:<password>] [--decrypt-secret=<private key>[:<password>]]
```
`private key` - base64 representation of key

`password` (optional) - password in *cleartext* if the given private key is protected with the password

For example,
```bash
kubectl create secret image-decrypt keysecret --decrypt-secret="base64OfPrivateKey":"password"
```
See it in action in the [demo](https://youtu.be/S3FK4y5McOk?t=19)

A variant for passing the ImageEncrypSecret is the following:

```bash
kubectl create secret image-decrypt <secret name> --from-file=<path to file>:[:<password>] 
```

`path to file` - The path of the private key in binary format

`password` (optional) - password in *cleartext* if the given private key is protected with the password
For example,
```bash
kubectl create secret image-decrypt keysecret --from-file="/path/to/private_key":"password"
```


Then when a user creates a workload, they can choose the desired ImageDecryptSecret to use 

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    name: nginx
spec:
  containers:
  - name: nginx
    image: localhost:5000/nginx:enc
    imageDecryptSecrets:
    - name: <secret name>
    imagePullPolicy: Always
    ports:
    - containerPort: 80

```

ImageDecryptSecrets can be added to the `service account` by,

```bash
kubectl patch serviceaccount <account name> -p '{"imageDecryptSecrets":[{"name":<secret name>, "images":[<image name>}]}'
```
or while creating a secret account,
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: 2015-08-07T22:02:39Z
  name: default
  namespace: default
  selfLink: /api/v1/namespaces/default/serviceaccounts/default
  uid: 052fb0f4-3d50-11e5-b066-42010af0d7b6
secrets:
- name: default-token-uudge
imagePullSecrets:
- name: myregistrykey
imageDecryptSecrets:
- name: <secret name>
- image: [<image name>]
```

For example,
```bash
kubectl patch serviceaccount default -p '{"imageDecryptSecrets":[{"name":"keysecret", "images":["localhost:5000/nginx:enc"]}]}'
```

#### Image Handler

The privake keys are extracted from `ImageDecryptSecret` and passed to the CRI as by bundling them in `DecryptParams` as a part of `PullImageRequest`:


```protobuf
// DecryptParams is used to send decryption parameters to the image service of the CRI
message DecryptParams {
    repeated string privateKeyPasswds = 1;
}
```
```protobuf
message PullImageRequest {
    // Spec of the image.
    ImageSpec image = 1;
    // Authentication configuration for pulling the image.
    AuthConfig auth = 2;
    // Config of the PodSandbox, which is used to pull image in PodSandbox context.
    PodSandboxConfig sandbox_config = 3;
    // DecryptParams for the images service of the CRI
    DecryptParams dcparams = 4;
}
```

See the [reference implementation](https://github.com/harche/kubernetes/blob/8a0ef9898a55110d7e41f8e1a846c66cfc2fa691/pkg/kubelet/apis/cri/runtime/v1alpha2/api.proto#L1063)

Just like in the case of `ImagePullSecrets`, CRI [forwards](https://github.com/stefanberger/containerd/blob/image-encryption.v4/vendor/github.com/containerd/cri/pkg/server/image_pull.go#L103) the private keys to containerd (or CRI-O) where the actual decryption of the image will take place. 

### Relationship with imagePullPolicy

`ImageDecryptSecrets` are designed by taking the inspiration from `ImagePullSecrets` due similarities in how they are consumed. Both the secrets need to provided to the runtime _before_ the pod is provisioned.

Because of this `ImageDecryptSecrets` inherit the existing contraints of `ImagePullSecrets`, especially around `imagePullPolicy`

| imagePullPolicy        | New Image           | Cached Image  |
| ------------- |:-------------:| -----:|
| Always | Keys Required | Keys Required |
| IfNotPresent      | Keys Required   |  Keys Not Required |
| Never | N/A  | Keys Not Required |

So if the image already exists on the target system, only the `Always` type of `imagePullPolicy` will guard the user against the unauthorized access to the image. However, this can be easily mitigated by sending down the required private keys during container creation as well, specifically by extending `createContainer` [here](https://github.com/harche/kubernetes/blob/pr_branch/pkg/kubelet/kuberuntime/kuberuntime_container.go#L124)

But this solution will be specific to the `ImageDecryptSecrets` and `ImagePullSecrets` will continue to suffer from the same issue. So looking at that, we believe that we need to have a generic enough solution in `createContainer` mentioned above that can enforce authorizattion for both `ImageDecryptSecrets` as well as `ImagePullSecrets` for the cached images. Hence, we are considering that work to be out of scope from this KEP's point of view.

### Consumption of the ImageDecryptSecrets

In the diagram https://imgur.com/a/eVkUF9w, when the kubelet wants to create a new pod which has encrypted images it has to first retrieve the decryption keys referenced decryption keys (They can be referenced directly in the pod or deployment yaml or in the pod’s service account). 

Kubelet sends the request to pull the image along with the decryption keys to CRI which then forwards it to containerd. Containerd looks up for the image in the snapshotter used by CRI, if the image doesn’t exist then it downloads it and decrypts it using the decryption keys that were passed by CRI. 

In case containerd finds the image in the snapshotter cache which was already pulled and decrypted by earlier request, it performs a check we call ‘Image Authorization’. This check only attempts to unwrap the keys in the image manifest of the image. In simple terms it means everytime you want to use an encrypted image, you will have to prove that you have the necessary keys to decrypt it even if the actual image is present in the decrypted form in the snapshotter. This prevents an attack where a user, without having decryption keys, might get access to encrypted image content if their pod gets scheduled on a worker node where that particular encrypted image was already pulled and decrypted by earlier request. 

There is no change required in CRI RuntimeService or Kubernetes

## Alternatives Considered

In order to decrypt an image containerd needs to have access to the corresponding private key. By the nature of it, a private key is a very sensitive piece of data. If it's lost, image confidentiality is compromised. It's a kind of a `secret` that's securing the data in an encrypted image. Kubernetes already has an infrastructure to handle secrets. This was the motivation to extend existing secrets to handle private keys required to decrypt an image. 

Containerd (or CRI-O) is the component that actually pulls the image, and hence does the decryption, on the worker node. So alternatively, if containerd manages to fetch keys on its own then we do not need Kubernetes to provision them. We did a POC around this idea where containerd talks to a Key Management Service (KMS) provider to fetch appropriate keys before decrypting the image. While this solution works as intended, the user needs to set up and maintain KMS. 

Using existing secret management in Kubernetes to provide decryption keys simplifies user flow, although using containerd to fetch keys also has it's use cases (mainly where the k8s master is not well trusted). 

## Graduation Criteria

Alpha:

- [] Everything described in the current proposal:
  - [] Introduce the ImageDecryptSecret API resource
  - [] Add a ImageDecryptSecret field to the PodSpec
  - [] Add a ImageDecryptSecret field to the CRI `PullImageRequest`
  - [] Plumb through the ImageHandler in the Kubelet
  - [] `kubectl` command to create a secret by using the base64 of the private key
- [] ImageDecryptSecret support in CRI-Containerd. 
  - [] An error is reported when the private key is invalid or is unknown or unsupported
- [] Testing
  - [] Kubernetes Unit Test cases

Beta:

- [] ImageDecryptSecret support in CRI-O
- [] `kubectl` command to create a secret by using the actual of the private key binary
- [] Comprehensive test coverage
  - [] [CRI validation tests][cri-validation]
- [] Testing
  - [] Kubernetes E2E tests (only validating single image handler cases)

 [cri-validation]: https://github.com/kubernetes-sigs/cri-tools/blob/master/docs/validation.md

## Implementation History
- 2018-11-26: Initial KEP [published](https://github.com/kubernetes/community/issues/2970)

