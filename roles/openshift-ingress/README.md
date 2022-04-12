# Ansible Role for Ingress

## Ingress

The Ingress Operator implements the ingresscontroller API and is the component responsible for enabling external access to OpenShift Container Platform cluster services. The Operator makes this possible by deploying and managing one or more HAProxy-based Ingress Controllers to handle routing. You can use the Ingress Operator to route traffic by specifying OpenShift Container Platform Route and Kubernetes Ingress resources.

For more information, please refer to:

* [Ingress Operator in OpenShift Container Platform](https://docs.openshift.com/container-platform/4.6/networking/ingress-operator.html)
* [Controlling pod placement using node taints](https://docs.openshift.com/container-platform/4.6/nodes/scheduling/nodes-scheduler-taints-tolerations.html)

## Role Tasks

Below we are adding a node selector and toleration in order for our Ingress pods to run on our 'infra' nodes.

*Creating Affinity to Infra Nodes and keeping the Internal LB using the AWS NLB*:
```yaml
- name: Ensure ingress controllers affinity to infra nodes and new ingress certificate is defined
  community.kubernetes.k8s:
    api_version: operator.openshift.io/v1
    state: present
    kind: IngressController
    name: default
    namespace: openshift-ingress-operator
    merge_type:
      - strategic-merge
      - merge
    definition:
      spec:
        replicas: 2
        unsupportedConfigOverrides:
          externalTrafficPolicy: Cluster
        endpointPublishingStrategy:
          loadBalancer:
            scope: Internal
            providerParameters:
              type: AWS
              aws:
                type: NLB
          type: LoadBalancerService
        nodePlacement:
          nodeSelector:
            matchLabels:
              node-role.kubernetes.io/infra: ""           #1
          tolerations:                                    #2
          - key: "node.ocs.openshift.io/storage"
            value: "true"
            effect: NoSchedule
```

1. We are setting for the pod to select nodes with the specified label
2. We are adding a toleration to match the taint for the 'infra' nodes

*Wait for the ingress pods to startup*:
```yaml
- name: Verify the IngressController Pods is present
  community.kubernetes.k8s_info:
    kind: Pod
    namespace: openshift-ingress-operator
  register: ingress_operator
  until: "ingress_operator | json_query('resources[].status.containerStatuses[?ready==`true`][]') | length == 2"
  retries: 30
  delay: 10
```
