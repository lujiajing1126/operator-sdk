---
title: Ansible Operator Tutorial
linkTitle: Tutorial
weight: 3
description: An in-depth walkthough that demonstrates how to build and run a Ansible-based operator.
---

This guide walks through an example of building a simple memcached-operator powered by Ansible using tools and libraries provided by the Operator SDK.

## Prerequisites

- [Install `operator-sdk`][operator_install] and the [Ansible prequisites][ansible-operator-install] 
- Access to a Kubernetes v1.16.0+ cluster.
- User authorized with `cluster-admin` permissions.

## Creating an Operator

In this section we will:
  - extend the Kubernetes API with a [Custom Resource Definition][custom-resources] that allows users to create `Memcached` resources.
  - create a manager that updates the state of the cluster to the desired state defined by `Memcached` resources.

#### Scaffold a New Project

Begin by generating a new project from a new directory.

```sh
$ mkdir memcached-operator 
$ cd memcached-operator
$ operator-sdk init --plugins=ansible --domain example.com
```

Among the files generated by this command is a Kubebuilder `PROJECT`
file. Subsequent `operator-sdk` commands (and help text) run from the
project root read this file and are aware that the project type is
Ansible.

```sh
# Since this is an Ansible-based project, this help text is Ansible specific.
$ operator-sdk create api -h
```

Next, we will create a `Memcached` API.

```sh
$ operator-sdk create api --group cache --version v1alpha1 --kind Memcached --generate-role
```

The scaffolded operator has the following structure:

 - `Memcached` Custom Resource Definition, and a sample `Memcached` resource.
 - A "Manager" that reconciles the state of the cluster to the desired state
    - A reconciler, which is an Ansible Role or Playbook.
    - A `watches.yaml` file, which connects the `Memcached` resource to the `memcached` Ansible Role.

See [scaffolded files reference][layout-doc] and [watches reference][ansible-watches] for more detailed information

#### Modify the Manager

Now we need to provide the reconcile logic, in the form of an Ansible
Role, which will run every time a `Memcached` resource is created,
updated, or delete.

Update `roles/memcached/tasks/main.yml`:


```yaml
---
- name: start memcached
  community.kubernetes.k8s:
    definition:
      kind: Deployment
      apiVersion: apps/v1
      metadata:
        name: '{{ ansible_operator_meta.name }}-memcached'
        namespace: '{{ ansible_operator_meta.namespace }}'
      spec:
        replicas: "{{size}}"
        selector:
          matchLabels:
            app: memcached
        template:
          metadata:
            labels:
              app: memcached
          spec:
            containers:
            - name: memcached
              command:
              - memcached
              - -m=64
              - -o
              - modern
              - -v
              image: "docker.io/memcached:1.4.36-alpine"
              ports:
                - containerPort: 11211
```

This memcached role will:
- Ensure a memcached Deployment exists
- Set the Deployment size

It is good practice to set default values for variables used in Ansible
Roles, so edit `roles/memcached/defaults/main.yml`:

```yaml
---
# defaults file for Memcached
size: 1
```

Finally, update the `Memcached` sample, `config/samples/cache_v1alpha1_memcached.yaml`:

```yaml
apiVersion: cache.example.com/v1alpha1
kind: Memcached
metadata:
  name: memcached-sample
spec:
  size: 3
```

The key-value pairs in the Custom Resource spec are passed
to Ansible as extra variables.

__Note:__ The names of all variables in the spec field are converted to
snake_case by the operator before running ansible. For example,
serviceAccount in the spec becomes service_account in ansible. You can
disable this case conversion by setting the `snakeCaseParameters` option
to `false` in your `watches.yaml`. It is recommended that you perform some
type validation in Ansible on the variables to ensure that your
application is receiving expected input.

#### Finishing up

All that remains is building and pushing the operator container to your favorite registry.

``` sh
$  make docker-build docker-push IMG=<some-registry>/<project-name>:tag
```

NOTE: To allow the cluster pull the image the repository needs to be set as public or you must configure an image pull secret


## Using the Operator

This section walks through the steps that operator users will perform
to deploy the operator and managed resources.

#### Install the CRD

To apply the `Memcached` Kind (CRD):

```sh
$ make install
```
#### Deploy the Operator:

```sh
# IMG environment variable must be set
$ export IMG=<yourimage>
$ make deploy
```

We are using the `memcached-operator-system` Namespace, so let's set
that context. 

```sh
$ kubectl config set-context --current --namespace=memcached-operator-system
```

Verify that the memcached-operator is up and running:

```sh
$ kubectl get deployment 

NAME                     DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
memcached-operator       1         1         1            1           1m
```

#### Create Memcached Resource

Create the resource, the operator will do the rest.

```sh
$ kubectl apply -f config/samples/cache_v1alpha1_memcached.yaml
```

Verify that Memcached pods are created

```sh
$ kubectl get pods

NAME                                                     READY   STATUS    RESTARTS   AGE
memcached-operator-controller-manager-7b667d9979-4jkfb   2/2     Running   0          14s
memcached-sample-memcached-6456bdd5fc-8zgjf              1/1     Running   0          5s
memcached-sample-memcached-6456bdd5fc-hjkrp              1/1     Running   0          5s
memcached-sample-memcached-6456bdd5fc-mcqc5              1/1     Running   0          5s
```

#### Cleanup

Clean up the resources:

```sh
$ make undeploy
```

## Next Steps

We recommend reading through the our [Ansible development
section][ansible-developer-tips] for tips and tricks, including how to
run the operator locally.

In this tutorial, the scaffolded `watches.yaml` could be used as-is, but
has additional optional features. See [watches reference][ansible-watches].

For brevity, some of the scaffolded files were left out of this guide.
See [Scaffolding Reference][layout-doc]

This example built a namespaced scope operator, but Ansible operators
can also be used with cluster-wide scope. See the [operator scope][operator-scope] documentation.

OLM will manage creation of most if not all resources required to run your operator, using a bit of setup from other operator-sdk commands. Check out the [OLM integration guide][quickstart-bundle].

[ansible-operator-install]: /docs/building-operators/ansible/installation
[ansible-developer-tips]: /docs/building-operators/ansible/development-tips/
[ansible-watches]: /docs/building-operators/ansible/reference/watches
[custom-resources]: https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/
[operator-scope]:https://v0-19-x.sdk.operatorframework.io/docs/legacy-common/operator-scope/
[layout-doc]:../reference/scaffolding
[docker-tool]:https://docs.docker.com/install/
[kubectl-tool]:https://kubernetes.io/docs/tasks/tools/install-kubectl/
[quickstart-bundle]: /docs/olm-integration/quickstart-bundle/
[operator_install]: /docs/installation/install-operator-sdk