# Ansible Role for Infra Nodes

## Infra Nodes

Having Infra nodes allows you to run workloads that are not strictly part of the Control Plane, nor are they the Applications you want to run. The main items we mean when talking of such 'Infrastructure' is the handling of Routing, the Image Registry, Metrics, and Logging. Keeping these distinct from your applications gives a good and clear separation.

For more information, please refer to:

* [Creating infrastructure machine sets](https://docs.openshift.com/container-platform/4.6/machine_management/creating-infrastructure-machinesets.html)
* [Controlling pod placement using node taints](https://docs.openshift.com/container-platform/4.6/nodes/scheduling/nodes-scheduler-taints-tolerations.html)
* [Taints: Expectations vs. Reality](https://www.openshift.com/blog/taints-expectations-vs.-reality)

## Role Tasks

This role can be borkwn down into 2 main sections. The first is to get details of the cluster and the second is to use said details to create new machinesets based on the existing worker machinesets.

*Getting cluster details*:
```yaml
---

- name: Get cluster info
  community.kubernetes.k8s_info:
    api_version: config.openshift.io/v1
    name: cluster
    kind: Infrastructure
  register: clusterInfo

- name: Get cluster version
  community.kubernetes.k8s_info:
    api_version: config.openshift.io/v1
    kind: ClusterVersion
    name: version
  register: clusterVersion

- name: Get Machine Region Info
  community.kubernetes.k8s_info:
    api_version: machine.openshift.io/v1beta1
    kind: Machine
    namespace: openshift-machine-api
  register: machineRegion

- name: debug
  debug:
    msg: "{{ machineRegion.resources[0].spec.providerSpec.value.placement.availabilityZone }}"

- name: Get Worker MachineSetInfo fot AZ a
  community.kubernetes.k8s_info:
    api_version: machine.openshift.io/v1beta1
    name: "{{ clusterInfo.resources[0].status.infrastructureName }}-worker-{{ machineRegion.resources[0].spec.providerSpec.value.placement.region }}a"
    kind: MachineSet
    namespace: openshift-machine-api
  register: machineSetInfoA

- name: Get Worker MachineSetInfo fot AZ b
  community.kubernetes.k8s_info:
    api_version: machine.openshift.io/v1beta1
    name: "{{ clusterInfo.resources[0].status.infrastructureName }}-worker-{{ machineRegion.resources[0].spec.providerSpec.value.placement.region }}b"
    kind: MachineSet
    namespace: openshift-machine-api
  register: machineSetInfoB

- name: Get Worker MachineSetInfo fot AZ c
  community.kubernetes.k8s_info:
    api_version: machine.openshift.io/v1beta1
    name: "{{ clusterInfo.resources[0].status.infrastructureName }}-worker-{{ machineRegion.resources[0].spec.providerSpec.value.placement.region }}c"
    kind: MachineSet
    namespace: openshift-machine-api
  register: machineSetInfoC
```

These details are then used in our jinja templates located in the 'templates' directory of this role.
*Creating MachineSets from templates*:
```yaml
- name: Create manchinesets
  community.kubernetes.k8s:
    state: present
    definition: "{{ lookup('template', 'templates/{{ item }}.yaml.j2') }}"
  loop:
    - infra_a
    - infra_b
    - infra_c
```
