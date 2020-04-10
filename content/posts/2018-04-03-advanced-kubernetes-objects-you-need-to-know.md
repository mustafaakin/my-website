---
title: Advanced Kubernetes Objects You Need to Know
date: '2018-04-03T14:31:01.543Z'
---

> Originally appeared on [Opsgenie Engineering Blog](https://engineering.opsgenie.com/advanced-kubernetes-objects-53f5e9bc0c28)

[Kubernetes](https://kubernetes.io/) adoption is increasing each day. People are transforming both development and production environments to container-based deployments, and they are making use of Kubernetes to handle the operations more elegantly. Ability to do one-click zero downtime rolling deployment updates was a dream or required too many interventions by an operator or a custom in-house applications where they were heavily dependent on the specific platforms. However, like every tool, Kubernetes has a learning curve, so it can be overwhelming for starters.

In Kubernetes, Objects are persistent entities which can be queried and updated via APIs. Having a single endpoint makes the management of every object easier, because almost all the time, you supply Kubernetes an Object and want it to apply it, and the API server takes the required steps to fulfill your request by detecting the changes in the given object template.

When you start learning and experimenting with Kubernetes, you mostly use a small subset of the objects. The most basic one is _Pod_, which is a group of one or more containers. They are ephemeral entities, meaning they are not durable. So, if you deploy your applications by utilizing pods, you will need to take additional steps to ensure a number of your pods are running, and they are healthy. However, Kubernetes already has more complex objects such as _ReplicaSets_ and _Deployments_ which handle the lifetime of pods. There are also objects for service discovery, such as _Service_ and _Ingress_, and configuration objects _ConfigMap_ and _Secrets_. Although these objects will solve most of your problems, there are many more objects and controllers which can make your life easier while using Kubernetes. If you examine the Kubernetes API documentation, you will see many objects. The objects are categorized as:

1.  **Workloads:** Manage containers and their lifetimes
2.  **Discovery & Load Balancing:** Make your applications accessible to each other or external world
3.  **Config & Storage:** Bind data to your containers
4.  **Metadata:** Adjust the behavioral data for other objects
5.  **Cluster:** Managing cluster state and configurations

In this blog post, we will introduce you to some of the rarely used or not widely known objects and how are they used by Kubernetes API and controllers, and how they can improve your workloads and day-to-day operations.

If you are familiar with Kubernetes, you might already know basic objects and controllers like _Pod_, _Deployment_, _Volume_, _Service_ and _Ingress_. However, with default permissions, there is no limit on what users can request from Kubernetes API. A user can request unlimited number of replicas, volumes, any Docker image, arbitrary CPU and Memory resources. Having an uncontrolled environment would cause instability as pods will most likely be competing for resources. You might also need some enforcements on what can be run, so that deployments both conform your technical and business needs.

Admission controllers are used for intercepting the requests to Kubernetes API, such as creating a _Deployment_, a new _ConfigMap_, etc. In other words, any Kubernetes object can be caught with admission controllers and modified before persisting them in the database. They can also be used to reject the objects, and many admission controllers might be chained to perform a set of checks. In short, admission controllers can be categorized as “validating” which accepts or rejects an object, and they can also be “mutating” which modifies the object before persistence.

The ability to intercept the objects before saving them allows enforcing some rules. For instance, you might want to let only a single domain of Docker images to pull, or you might enforce a naming scheme for objects, prevent some labels from being used, or add sidecar containers to each of your containers. The main reason why admission controllers are useful is that you can continue interacting with API server with proper credentials, and you do not have to create an additional proxy layer or a handler, and you maintain a fewer number of and smaller components, and it becomes easier to modify and swap them.

### ResourceQuota

As you might already know, you can specify pods’ CPU and Memory requests and limits, and as Kubernetes already knows the pod placements, it can properly place your pods into such places that your requests are fulfilled. When a pod has memory requests set, your pod’s QoS (Quality of Service) class is **Guaranteed**, and when your limit is higher than requests, QoS class is **Burstable**. In other words, your pod gets at least the resources it desires, if there is space. However, limiting the total requests by namespace can be useful if you have many namespaces used by many projects or people so that namespaces get their fair shares. This is where _ResourceQuota_ helps, and it can be defined as a simple YAML file as follows:

```yaml
apiVersion: v1  
kind: ResourceQuota  
metadata:  
  name: my-cheap-namespace  
spec:  
  hard:  
    requests.cpu: "4"  
    requests.memory: 8Gi  
    limits.cpu: "16"  
    limits.memory: 16Gi
```

You can also limit the number of Kubernetes objects that a namespace can use. You can limit the total number of Pods to avoid scheduling overheads, the number of load balancers (which can be tied to a load balancer with an actual cost in a cloud provider, such as AWS Network Load Balancer) or the number of Persistent Volume Claims as they can be costly as well. An example configuration is as follows:

```yaml
apiVersion: v1  
kind: ResourceQuota  
metadata:  
  name: object-quota-demo  
spec:  
  hard:  
    persistentvolumeclaims: "4"  
    services.loadbalancers: "3"  
    services.nodeports: "1"
```

### PriorityClass

If you are running high-intensity workloads, it might be common for you to be out of space from time to time. Sure, you might add new worker kubelets with the auto-scalers. However, it might be too late. If some of your containers can tolerate eviction, such as background tasks that are not directly customer facing, there is no reason that they cannot be reduced to make room for your new deployment. _PriorityClass_ also allows the higher priority pods to be scheduled earlier than the lower priority ones.

Note that, _PodDistruptionBudgets_ try to guarantee a number or percentage of pods running at a time. However, if a high priority pod is submitted to the scheduler, the lower priority pod with a PodDistruptionBudget might not be guaranteed, and more than desired number of pods might be deleted.

A _PriorityClass_ is as defined as follows.

```yaml
apiVersion: scheduling.k8s.io/v1alpha1  
kind: PriorityClass  
metadata:  
  name: high-priority  
value: 1000000  
globalDefault: false  
description: "Wow this is very much you should have a good reason"
```

And you use it in a _Pod_ like the following:

```yaml
apiVersion: v1  
kind: Pod  
metadata:  
  name: my-very-important-container  
  labels:  
    important:   
spec:  
  containers:  
  - name: find-the-answer  
    image: return42:latest  
  priorityClassName: high-priority
```

### LimitRange

If you forget specifying CPU & Memory requests and limits for your pods, they will be in **BestEffort** QoS class. Specifying default limits and requests might be helpful for beginners of your Kubernetes adaptation. Limiting container resource usage is a good practice because rogue pods might deter the other ones. You can define a _LimitRange_ as follows:

```yaml
apiVersion: v1  
kind: LimitRange  
metadata:  
  name: cpu-limit-range  
spec:  
  limits:  
  - default:  
      cpu: 1  
    defaultRequest:  
      cpu: 0.5  
    type: Container
```

### PodSecurityPolicy

You might sometimes want to run untrusted pods and desire to secure your Kubernetes cluster against harmful intent as much as possible. Even if you are running trusted pools, your applications or the software you use might have some security vulnerabilities and might be exploited if they are facing the public networks.

On the other hand, you might have some trusted applications that might need extended privileges, and you would like to grant them new capabilities that generally regular containers do not possess. Containers might want to modify protected kernel variables and features, and would like some advanced system calls.

While you could do the above, You can modify both grants and restrictions in a centralized manner with Kubernetes. This is where _PodSecurityPolicy_ is helpful. You can configure the SELinux and AppArmor rules, drop and add Linux capabilities, modify namespace sharing for PID, network, IPC, enforce the user and group of the containers and even make the container read-only.

The below is an example of a security policy which is a restrictive policy, but it also allows some of the sysctl’s to be configured.


```yaml
apiVersion: policy/v1beta1  
kind: PodSecurityPolicy  
metadata:  
  name: restricted  
  Annotations:  
security.alpha.kubernetes.io/sysctls: 'net.ipv4.route.\*,kernel.msg\*'  
    apparmor.security.beta.kubernetes.io/allowedProfileNames: 'runtime/default'  
    apparmor.security.beta.kubernetes.io/defaultProfileName:  'runtime/default'  
spec:  
  privileged: false  
  allowPrivilegeEscalation: false  
  requiredDropCapabilities:  
    - ALL  
  volumes:  
    - 'configMap'  
    - 'emptyDir'  
    - 'secret'  
    - 'persistentVolumeClaim'  
  hostNetwork: false  
  hostIPC: false  
  hostPID: false  
  runAsUser:  
    rule: 'MustRunAsNonRoot'  
  seLinux:  
    rule: 'RunAsAny'  
  supplementalGroups:  
    rule: 'MustRunAs'  
    ranges:  
      - min: 1  
        max: 65535  
  fsGroup:  
    rule: 'MustRunAs'  
    ranges:  
      - min: 1  
        max: 65535  
  readOnlyRootFilesystem: false
```

### ImagePolicyWebhook and ImageReview

For a safe and controlled deployment, it is desirable to specify a policy for the images that can be used as containers. Running arbitrary containers on production might lead to some undesired situations if you are not careful. A classical cluster admin would configure HTTP proxy or SSL termination for all the nodes, and implement a custom firewall that will reject requests for unverified domains or paths. However, it is hard and prone to error. Good news is that it can be quickly done in Kubernetes with _ImagePolicyWebhook_. As you can understand from the name, Kubernetes API can perform a webhook to an external service to validate your image pull request. The external API is supposed to check the images and additional information like annotations and namespace and make a decision and return success or failure to the Kubernetes API server. The server can also return a reason string so that the user can be informed why a particular image was rejected. The responses can also be cached for a configurable time to increase the performance. An example _ImageReview_ object passed to the configured server can be seen as below:

```json
{    
  "apiVersion":"imagepolicy.k8s.io/v1alpha1",  
  "kind":"ImageReview",  
  "spec":{    
    "containers":[    
      {    
        "image":"myrepo/myimage:v1"  
      },  
      {    
        "image":"myrepo/myimage@sha256:alongstring"  
      }  
    ],  
    "annotations":[    
      "mycluster.image-policy.k8s.io/ticket-1234":"break-glass"  
    ],  
    "namespace":"mynamespace"  
  }  
}
```

### ValidatingAdmissionWebhook and MutatingAdmissionWebhook

Although you can use the default Kubernetes Admission controllers, which can enforce LimitRange, ResourceQuota or different known objects, you might have some non-trivial requirements and want to implement logic for such purposes. This is where ValidatingAdmissionWebhook comes to help, and as similar to ImagePolicyWebhook, it passes Kubernetes object requests you configure to the endpoint of your choice, and you are free to accept or reject the request. You can also configure which API calls you are interested, so you are only passed the ones you desire. In the below example, a ValidatingWebhookConfiguration is given with a single webhook which validates only the pods _CREATE_ requests. You can configure any Kubernetes object to be included for verification.

```yaml
apiVersion: admissionregistration.k8s.io/v1beta1  
kind: ValidatingWebhookConfiguration  
metadata:  
  name: validate-your-containers  
webhooks:  
\- name: verify-pod-creation  
  rules:  
  - apiGroups:  
    - ""  
    apiVersions:  
    - v1  
    operations:  
    - CREATE  
    resources:  
    - pods  
  clientConfig:  
    service:  
      namespace: default  
      name: name  
    caBundle: <pem-encoded>
```

_MutatingAdmissionWebhook_ is similar to _ValidatingWebhookConfiguration,_ you are again passed the Kubernetes object that is being created, however, instead of rejecting or accepting, your endpoint is supposed to modify the original Kubernetes object and return the modified request. You could use it to change the environment variables of a pod, append a default label, set a namespace if the user forgot to do so. Any custom corporate requirements can be done here!

### Available Admission Controllers

In addition to above admission controllers, there are various ones you can use, and they are summarized as follows (not including deprecated ones):

*   **AlwaysPullImages:** Modifies each pod’s _imagePullPolicy_ to always so that every time the latest Docker image is pulled. Note that might not be desirable to use latest in production since it would violate the stability.
*   **DefaultStorageClass:** If a volume request does not have a storage class, it assigns one, so the user does not care and get the default class.
*   **DefaultTolerationSeconds:** Sets the default toleration for pods for tolerating specific taints _not_ _ready:NoExecute_ and _unreachable:NoExecute_ for 5 minutes.
*   **DenyEscalatingExec:** This controller will deny the privileges to be escalated, meaning getting more rights on the host system, even when the container is running as running as privileged.
*   **EventRateLimit (alpha):** Protects the Kubernetes API server from request floods to prevent both accidental and intentional abuse.
*   **ExtendedResourceToleration:** Extended resources such as GPUs and FPGAs can be used in Kubernetes, but such nodes should be tainted with the extended resource’s name. This controller automatically adds such taints to pods if they require extended resources.
*   **ImagePolicyWebhook:** As explained above, performs a webhook for the image to be pulled to accept or reject.
*   **Initializers (alpha):** The initializer is a new feature, which is already similar to regular admission controllers. The downside of admission controllers is that they are compiled into the kubernetes binary, and you need to enable them explicitly. Initializers allow you to write custom controllers, and you can find more info on [this blog post](https://ahmet.im/blog/initializers/) by Ahmet Alp Balkan. There is also an example by Kelsey Hightower of injecting Envoy proxies with Initializers in each Deployment [in this repository](https://github.com/kelseyhightower/kubernetes-initializer-tutorial).
*   **InitialResources (experimental):** This controllers examine the pod’s resource usage history and come up with an estimation of the resources of a new pod creation request. While this is an experimental controller, it is promising and might make your environments more stable due to foreseeable resource requests and make the scheduling more comfortable and more correct.
*   **LimitPodHardAntiAffinityTopology:** This admission controller denies any pod that defines _AntiAffinity_ topology key other than _kubernetes.io/hostname_ in _requiredDuringSchedulingRequiredDuringExecution_. This ensures, if affinity is requested, pods are distributed according to hostname only.
*   **LimitRanger:** Enforces that the requested sources in the _LimitRange_ object are not exceeding available resources with new pod deployments.
*   **MutatingAdmissionWebhook (beta):** As explained above, modifies an object before an actual request to API.
*   **NamespaceAutoProvision:** Creates the namespaces automatically, when an object with a non-existing namespace arrives.
*   **NamespaceExists:** If a namespaced object is submitted without a namespace, the request is rejected. Might be helpful to enforce resource segregation properly.
*   **NamespaceLifecycle:** Used to prevent deletion of default system namespaces, _default_, _kube-system_ and _kube-public_. Also prevents adding new objects to a namespace which is being deleted. Deleting a namespace deletes each resource in it so that it might be useful.
*   **NodeRestriction:** Kubernetes worker nodes are called as kubelets, and this controller limits the kubelet’s ability to modify Node and Pod objects. This allows locking down the nodes even more, and in the future, it is expected to add more restrictions so that the kubelets are less harmful in the event of the compromise of a node.
*   **OwnerReferencesPermissionEnforcement:** This controller has a fancy name, but it protects the _metadata.ownerReferences_ of a Kubernetes object, so it is forbidden by a user to change it and be its owner if the user does not have delete permissions.
*   **PodNodeSelector:** This controller enforces what selectors can be used which node selectors might be used for pods.
*   **PersistentVolumeClaimResize:** This controller decides whether a volume can be resized. If enabled, storage classes must explicitly define _allowVolumeExpansion_ if they can be resized or the API will reject such requests.
*   **PodPreset:** PodPresets allows setting a template for your pods. Upon pod creation, the controller iterates over all PodPresets and if your pod matches, the preset is applied to your pod. It can be useful to provide a baseline for all pods or attaching specific ConfigMap or Volumes to pods.
*   **PodSecurityPolicy:** As explained above, this applies preset security configuration for pods to ensure a common security policy for your cluster.
*   **PodTolerationRestriction:** Kubernetes uses affinities and taints to attract or retract pods. When a pod has toleration, it can be assigned to a node that has taints. This controller manages if the pod can tolerate the taint. This can be useful if you want to lock down a node and make sure only the approved ones can schedule there. Or you can be sure it a pod is not allowed to tolerate fatal taints such as _node.kubernetes.io/out-of-disk_.
*   **Priority:** Uses the _priorityClassName_ field to set an integer value to a pod priority. If no priority class is found, the request is rejected. Making priorities mandatory might be helpful to ensure availability of your services.
*   **ResourceQuota:** As explained above, this controller allows ensuring the resource demands are not exceeding the node capacities.
*   **SecurityContextDeny:** Ensures only allowed _SecurityContext_ fields to be set by pods. For instance, you can reject a pod becoming root, or sharing host PID namespaces.
*   **ServiceAccount:** Implements the automation for service accounts, which are used by pods to communicate with Kubernetes API.
*   **Storage Object in Use Protection (beta):** Adds new finalizers to Persistent Volumes and Claims and protects PVs and PVCs that are in active use from premature removal.
*   **ValidatingAdmissionWebhook (beta):** As explained above, this controller allows validating submitted objects to the API before their persistence.

**Disclaimer:** Some of the examples are taken from the official [Kubernetes documentation](https://kubernetes.io/docs/home/?path=browse) and adapted to this blog.