# Ansible Role for Image Registry

## Image Registry

OpenShift Container Platform provides a built-in container image registry that runs as a standard workload on the cluster. The registry is configured and managed by an infrastructure Operator. It provides an out-of-the-box solution for users to manage the images that run their workloads, and runs on top of the existing cluster infrastructure. This registry can be scaled up or down like any other cluster workload and does not require specific infrastructure provisioning. In addition, it is integrated into the cluster user authentication and authorization system, which means that access to create and retrieve images is controlled by defining user permissions on the image resources.

For more information, please refer to:

* [Integrated OpenShift Container Platform registry](https://docs.openshift.com/container-platform/4.6/registry/architecture-component-imageregistry.html)
* [Controlling pod placement using node taints](https://docs.openshift.com/container-platform/4.6/nodes/scheduling/nodes-scheduler-taints-tolerations.html)

## Role Tasks

Below we are adding a node selector and toleration in order for our image registry pods to run on our 'infra' nodes.

*Creating Affinity to Infra Nodes*:
```yaml
- name: Move Registry to Infra nodes
  community.kubernetes.k8s:
    api_version: imageregistry.operator.openshift.io/v1
    state: present
    kind: Config
    name: cluster
    namespace: openshift-ingress-operator
    merge_type:
      - strategic-merge
      - merge
    definition:
      spec:
        nodeSelector:
          node-role.kubernetes.io/infra: ""               #1
        tolerations:                                      #2
        - key: "node.ocs.openshift.io/storage"
          value: "true"
          effect: NoSchedule
```

1. We are setting for the pod to select nodes with the specified label
2. We are adding a toleration to match the taint for the 'infra' nodes
