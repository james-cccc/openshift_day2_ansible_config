# Ansible Role for OCS

## OpenShift Container Storage

Red Hat OpenShift Container Storage is a provider of agnostic persistent storage for OpenShift Container Platform supporting file, block, and object storage, either in-house or in hybrid clouds. As a Red Hat storage solution, Red Hat OpenShift Container Storage is completely integrated with OpenShift Container Platform for deployment, management, and monitoring.

For more information please refer to:

* [Red Hat OpenShift Container Storage](https://docs.openshift.com/container-platform/4.6/storage/persistent_storage/persistent-storage-ocs.html)

## Role Tasks

Creating the namespace, operatorgroup and subscription will install the operator.

*Operator Installation*:
```yaml
---
  - name: Ensure namespace existence for the Storage Operator
    community.kubernetes.k8s:
        state: present
        api_version: v1
        kind: Namespace
        merge_type:
          - strategic-merge
          - merge
        definition:
          metadata:
            name: openshift-storage

  - name: Ensure operator group existence for the Storage Operator
    community.kubernetes.k8s:
        state: present
        api_version: operators.coreos.com/v1
        kind: OperatorGroup
        merge_type:
          - strategic-merge
          - merge
        definition:
          metadata:
            name: openshift-storage-operatorgroup
            namespace: openshift-storage
          spec:
            targetNamespaces:
              - openshift-storage

  - name: Ensure subscription existence for the Storage Operator
    community.kubernetes.k8s:
        state: present
        api_version: operators.coreos.com/v1alpha1
        kind: Subscription
        merge_type:
          - strategic-merge
          - merge
        definition:
          metadata:
            name: ocs-subscription
            namespace: openshift-storage
          spec:
            channel: "{{ ocs_channel }}"
            installPlanApproval: Automatic
            name:  ocs-operator
            source: redhat-operators
            sourceNamespace: openshift-marketplace
```

The below tasks will wait for OCS to fully install before the play continues.

*Installation Verificaton*:
```yaml
---
- name: Verify that the OCS operator subscription is created
  community.kubernetes.k8s_info:
    kind: Subscription
    namespace: openshift-storage
    name: ocs-subscription
  register: ocs_sub
  until: "ocs_sub.failed == false"
  retries: 30
  delay: 10

- name: Get OCS CSV
  community.kubernetes.k8s_info:
    api_version: operators.coreos.com/v1alpha1
    kind: Subscription
    name: ocs-subscription
    namespace: openshift-storage
  register: OCS_SUB_STATUS
  until: OCS_SUB_STATUS.resources[0].status.installedCSV is defined
  retries: 30
  delay: 10

- name: Display OCS CSV version
  debug:
    msg: "{{ OCS_SUB_STATUS.resources[0].status.installedCSV }}"

- name: Wait for OCS Operator to Install
  community.kubernetes.k8s_info:
    api_version: operators.coreos.com/v1alpha1
    kind: ClusterServiceVersion
    namespace: openshift-storage
    name: "{{ OCS_SUB_STATUS.resources[0].status.installedCSV }}"
  register: OCS_STATUS
  until: "'InstallSucceeded' in OCS_STATUS | json_query('resources[].status.reason | []')"
  retries: 90
  delay: 2
  ignore_errors: true
  ```

The below task will create the AWS credentials required for Nooba.

*Nooba Secret*:
```yaml
- name: Create Noobaa secret
  community.kubernetes.k8s:
    state: present
    kind: Secret
    name: noobaa-aws-cloud-creds-secret
    namespace: openshift-storage
    definition:
      data:
        aws_access_key_id: "{{ nooba_aws_access_key_id }}"
        aws_secret_access_key: "{{ nooba_aaws_secret_access_key }}"
```

The below tasks will wait for the OCS operators to all be up and available.

*Wait for OCS operators to come up*:
```yaml
- name: Wait for OpenShift Storage Operator
  community.kubernetes.k8s_info:
    api_version: v1
    kind: Pod
    namespace: openshift-storage
    label_selectors:
      - name = ocs-operator
  register: pod
  retries: 72
  delay: 5
  until:
    - pod.resources[0].status.phase == "Running"

- name: Wait for Rook/Ceph Operator
  community.kubernetes.k8s_info:
    api_version: v1
    kind: Pod
    namespace: openshift-storage
    label_selectors:
      - app = rook-ceph-operator
  register: pod
  retries: 72
  delay: 5
  until:
    - pod.resources[0].status.phase == "Running"

- name: Wait for Noobaa Operator
  community.kubernetes.k8s_info:
    api_version: v1
    kind: Pod
    namespace: openshift-storage
    label_selectors:
      - noobaa-operator = deployment
  register: pod
  retries: 2
  delay: 5
  until:
    - pod.resources[0].status.phase == "Running"
```

Using `K8s` we can specifiy the manifests path using `definition` and set the `state` to `present`.

*Create the storage instance from a template and wait until it sucessfully creates*:
```yaml
- name: Create StorageCluster Resource
  community.kubernetes.k8s:
    state: present
    definition: "{{ lookup('template', 'templates/storage_cluster.j2') }}"

- name: Wait 20 Minutes for Storage Cluster to Become Available
  community.kubernetes.k8s_info:
    api_version: ocs.openshift.io/v1
    kind: StorageCluster
    name: ocs-storagecluster
    namespace: openshift-storage
  retries: 300
  delay: 5
  register: sc
  until:
    - sc.resources[0].status.phase == "Ready"
```

The `gp2` storageclass is created during the installation of Openshift when the cloud-provider is AWS. It can be used for provisioning gp2 as persistent volumes. Since we are using OCS the `gp2` StorageClass shouldn't be default for us. Thus we can switch the **`storageclass.kubernetes.io/is-default-class`** key to `false` for `gp2`. This ensures that the previously created `ocs-storagecluster-cephfs` StorageClass will have precedence when creating new PVCs when no storageclass is set.

*Setting default storageclass*:

```yaml
- name: Ensure GP2 is non defualt
  community.kubernetes.k8s:
    state: present
    kind: StorageClass
    api_version: storage.k8s.io/v1
    merge_type:
      - strategic-merge
      - merge
    definition:
      metadata:
        name: gp2
        annotations:
          storageclass.kubernetes.io/is-default-class: 'false'
      provisioner: kubernetes.io/aws-ebs
      parameters:
        encrypted: 'true'
        type: gp2
      reclaimPolicy: Delete
      allowVolumeExpansion: true
      volumeBindingMode: WaitForFirstConsumer

- name: Ensure OCS SC is configured as default
  community.kubernetes.k8s:
    state: present
    kind: StorageClass
    api_version: storage.k8s.io/v1
    name: ocs-storagecluster-cephfs
    merge_type:
      - strategic-merge
      - merge
    definition:
      metadata:
        name: ocs-storagecluster-cephfs
        annotations:
          storageclass.kubernetes.io/is-default-class: 'true'
      provisioner: openshift-storage.cephfs.csi.ceph.com
      parameters:
        clusterID: openshift-storage
        csi.storage.k8s.io/controller-expand-secret-name: rook-csi-cephfs-provisioner
        csi.storage.k8s.io/controller-expand-secret-namespace: openshift-storage
        csi.storage.k8s.io/node-stage-secret-name: rook-csi-cephfs-node
        csi.storage.k8s.io/node-stage-secret-namespace: openshift-storage
        csi.storage.k8s.io/provisioner-secret-name: rook-csi-cephfs-provisioner
        csi.storage.k8s.io/provisioner-secret-namespace: openshift-storage
        fsName: ocs-storagecluster-cephfilesystem
      reclaimPolicy: Delete
      allowVolumeExpansion: true
      volumeBindingMode: Immediate
```
After applying these 2 tasks we should see that the OCS storageclass `ocs-storagecluster-cephfs` has been set to `default` when observing the Storage Classes.
