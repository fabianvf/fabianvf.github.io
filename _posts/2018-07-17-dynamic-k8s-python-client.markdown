---
layout: post
title:  "Dynamic Kubernetes client for Python and Ansible"
date:   2018-07-17 9:00:00
categories: blog
---

[Originally posted on the Ansible blog](https://www.ansible.com/blog/dynamic-kubernetes-client-for-ansible)

## tl;dr

We condensed the Python Kubernetes/OpenShift client from 400,000 lines of code to 500, while adding features and closing nearly all known bugs. The new Kubernetes modules shipping in Ansible 2.6 support all resources the Kubernetes server supports, and fix nearly all the bugs that were in the 2.5 k8s_raw and openshift_raw modules. If you want to control your Kubernetes infrastructure with Ansible, now is a very good time to give it a try.

## Previous Approaches

For anyone who has not followed the process of adding Kubernetes support to Ansible, this is actually our third attempt. With this iteration we have finally worked out a lot of the kinks that made the modules difficult to use. Here's a brief synopsis of the history of the project:

### Generated client, generated modules

Our first iteration was backed by a generated [OpenShift Python client](https://github.com/openshift/openshift-restclient-python), based on the existing [Kubernetes Python client](https://github.com/kubernetes-client/python). This Python client ingested the OpenAPI spec for the OpenShift/Kubernetes API and generated one or more modules per resource type. Due to the size of the API, this resulted in ~400,000 lines of generated code.

The [Ansible Kubernetes modules](https://github.com/ansible/ansible-kubernetes-modules) were in turn generated from the generated client, so for each Python module a corresponding Ansible module was created. We also took the spec for the object and flattened all the parameters to make it look like a normal Ansible module. That resulted in a lot of parameters that looked like this:
```
spec_template_spec_affinity_node_affinity_preferred_during_scheduling_ignored_during_execution

```

For obvious reasons, we decided that this approach was not going to be the winner.

### Generated client, static modules

The second attempt, spearheaded by Chris Houseknecht, focused on greatly simplifying the experience of the Ansible users. Rather than generate an Ansible module for every resource type, we had a single module, named `k8s_raw` (with a corresponding `openshift_raw` for OpenShift resources). This module allowed a user to pass in their valid yaml definition of a Kubernetes object, and then translated the definition into calls to the generated OpenShift Python client. One of the major issues here was actually caused by needing to translate the camelCased Kubernetes definition into snake_cased Python calls. Despite its apparent simplicity it was remarkably prone to error, especially in cases where something like `xxxxIP` was turned into `xxxx_i_p`. The other large source of bugs in this approach came from overeager client side validation, because the OpenAPI spec was imprecise in parts and specified validation that would reject valid objects. This was difficult to fix in a generic way because we relied on the spec to generate all of our logic, and didn't have much control over what the generation process did.

## Problem

During the process of building out the Python client and Ansible modules, there was one issue that was raised repeatedly that fell outside of the set of problems that the generated client could solve. The core of the issue is that Kubernetes is moving away from extending the core API, and is instead focusing on adding systems that allow cluster admins to extend the runtime API of Kubernetes through two approaches: The first, CustomResourceDefinitions, allow you to define custom API types; the second, aggregated API servers, allow you to define an API server that accepts a set of resources and processes them outside of the main Kubernetes API server. Because the generated client and modules were based entirely on a static OpenAPI spec, there was no way we could account for these arbitrary runtime resources.

## Solution

Fixing this problem required starting over from the ground up. Rather than statically generating code from the OpenAPI spec, we needed to contact the server and determine the available resources, and then generically handle operations on those resources. Luckily this is a problem that Kubernetes was already architected to solve.

Kubernetes exposes a discovery API, `/apis`. From `/apis`, you can find all API groups, versions, and kinds that the server supports, as well as metadata about those resources, most importantly the actions supported, and whether the resource is namespaced.

With this resource collection in hand, all that's left to do for the client is send a request to the proper endpoint. Kubernetes has a remarkably consistent set of API routes, which allows you to determine the exact URL for a request based only on the method, APIGroup, APIVersion, and resource Kind. This means that the dynamic client is actually little more than the API discovery logic + a simple URL templater. In fact, the whole dynamic client works out to about 500 lines of code, which is not bad compared to the 400,000 we started with.

The basic flow of the library looks like this:

1. Client instantiated
1. Discovery API traversed and cached
1. User queries client for a resource type, for example, `apps/v1 Deployment`. If resource type exists in cache, API object is returned.
1. User calls `get|create|delete|patch|replace` on that resource (providing `name`, `namespace`, `body` as required)
1. Client templates a URL and makes a request based on the GroupVersionKind, method, and whether the resource is namespaced.
  - Example: `POST /apis/apps/v1/namespaces/default/deployments` to `create` a `Deployment` in the `default` namespace.

## Usage and Examples

### Python
[Main documentation](https://github.com/openshift/openshift-restclient-python/#openshift-python-client)

The following block of code will create a `v1 Service` object in the `default` namespace.

```python
import kubernetes
from openshift.dynamic import DynamicClient

# Because there were no issues in the generated REST client we did not
# bother reimplementing the configuration/request portions of the
# Kubernetes library. Instead, the Dynamic client simply translates
# requests for arbitrary resources into calls to the generated REST
# client and parses the raw responses. As a result, the only
# argument needed to instantiate the DynamicClient is a Kubernetes
# client object from the Kubernetes python library
k8s_client = kubernetes.config.new_client_from_config()
dyn_client = DynamicClient(k8s_client)

# Here, we request a v1 Service object. The client searches the
# cached list of resources, and because v1.Service is present,
# will return a corresponding Resource object
v1_services = dyn_client.resources.get(api_version='v1', kind='Service')

# This is a standard Kubernetes definition for a Service
service = {
    "kind": "Service",
    "apiVersion": "v1",
    "metadata": {
        "name": "my-service"
    },
    "spec": {
        "selector": {
            "app": "MyApp"
        },
        "ports": [{
            "protocol": "TCP",
            "port": 8080,
            "targetPort": 9376,
        }]
    }
}

# This will POST the serialized body to
#     /api/v1/namespaces/default/services
resp = v1_services.create(body=service, namespace='default')

# resp is a ResourceInstance object
print(resp.metadata)
```

### Ansible
[Main documentation](https://docs.ansible.com/ansible/devel/modules/k8s_module.html)

For anyone that has used `k8s_raw` or `openshift_raw` in Ansible 2.5, this section should look familiar. Technically 2.6 will be including a new module, `k8s`, while deprecating `k8s_raw` and `openshift_raw`, but the API for `k8s` should match `k8s_raw` and `openshift_raw`, and `k8s_raw` and `openshift_raw` will just execute the `k8s` module, so there should be little impact on existing code. There were two main drivers for the rename:
1. People were intimidated by `_raw`. Many interpreted `_raw` to mean "internal and dangerous, do not touch."
1. There is no longer a meaningful difference in code for interacting with OpenShift and Kubernetes resources, so we were at least going to deprecate `openshift_raw`.


The following Ansible snippet has the same function as the Python example:
``` yaml
- name: Create a Service object from an inline definition
  k8s:
    state: present
    definition:
      apiVersion: v1
      kind: Service
      metadata:
        name: my-service
        namespace: default
      spec:
        selector:
          app: MyApp
        ports:
        - protocol: TCP
          targetPort: 9376
          port: 8080
```


## Special thanks
This project would never have made it off the ground were it not for the help and guidance of:

- [Adam Miller](https://github.com/maxamillion)
- [Chris Houseknecht](https://github.com/chouseknecht)
- [Clayton Coleman](https://github.com/smarterclayton)
- [David Zager](https://github.com/djzager)
- [Devan Goodwin](https://github.com/dgoodwin)
- [Haowei Cai](https://github.com/roycaihw)
- [James Cammarata](https://github.com/jimi-c)
- [Justin Detiberus](https://github.com/detiber)
- [Mehdy Bohlool](https://github.com/mbohlool)
