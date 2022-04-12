# Ansible Role for Cluster Logging Stack Deployment

## OpenShift Cluster Logging Stack

The cluster logging components are based upon Elasticsearch, Fluentd, and Kibana (EFK).

The collector, Fluentd, is deployed to each node in the OpenShift Container Platform cluster. It collects all node and container logs and writes them to Elasticsearch (ES).

Kibana is the centralized, web UI where users and administrators can create rich visualizations and dashboards with the aggregated data.

There are currently 5 different types of cluster logging components:

* **logStore** - This is where the logs will be stored. The current implementation is Elasticsearch.
* **collection** - This is the component that collects logs from the node, formats them, and stores them in the `logStore`. The current implementation is Fluentd.
* **visualization** - This is the UI component used to view logs, graphs, charts, and so forth. The current implementation is Kibana.
* **curation** - This is the component that trims logs by age. The current implementation is Curator.
* **event routing** - This component forwards OpenShift Container Platform events to cluster logging. The current implementation is Event Router.

For more about Cluster Logging, please refer to: [Understanding cluster logging](https://docs.openshift.com/container-platform/4.6/logging/cluster-logging.html)

## Role Tasks

We are installing cluster logging by deploying the Elasticsearch and Cluster Logging Operators.

* The Elasticsearch Operator creates and manages the Elasticsearch cluster used by cluster logging.
* The Cluster Logging Operator creates and manages the components of the logging stack.

Elasticsearch and Cluser Logging Operators are being deployed by creating Subscription objects for both:

*Elasticsearch Operator Installation*:
```yaml
- name: Set Namespace for elasticsearch
  community.kubernetes.k8s:
    state: present
    api_version: v1
    kind: Namespace
    merge_type:
      - strategic-merge
      - merge
    definition:
      metadata:
        name: openshift-operators-redhat
        annotations:
          openshift.io/node-selector: ""
        labels:
          openshift.io/cluster-monitoring: "true"

- name: Set Operator Group for elasticsearch
  community.kubernetes.k8s:
    state: present
    api_version: operators.coreos.com/v1
    kind: OperatorGroup
    namespace: openshift-operators-redhat
    merge_type:
      - strategic-merge
      - merge
    definition:
      metadata:
        name: openshift-operators-redhat
      spec: {}

- name:  Ensure subscription existence for elasticsearch
  community.kubernetes.k8s:
    state: present
    api_version: operators.coreos.com/v1alpha1
    kind: Subscription
    name: elasticsearch-operator
    namespace: openshift-operators-redhat
    merge_type:
      - strategic-merge
      - merge
    definition:
      spec:
        channel: "{{ logging_channel }}"
        installPlanApproval: Automatic
        name: elasticsearch-operator
        source: redhat-operators
        sourceNamespace: openshift-marketplace

- name: Verify Elasticseach subscription existence
  community.kubernetes.k8s_info:
    kind: Subscription
    name: elasticsearch-operator
    namespace: openshift-operators-redhat
  register: ES_SUB_EXISTENCE
  until: "ES_SUB_EXISTENCE.failed == false"
  retries: 30
  delay: 10

- name: Get Elasticseach subscription
  community.kubernetes.k8s_info:
    api_version: operators.coreos.com/v1alpha1
    kind: Subscription
    name: elasticsearch-operator
    namespace: openshift-operators-redhat
  register: ES_SUB_STATUS
  until: ES_SUB_STATUS.resources[0].status.installedCSV is defined
  retries: 30
  delay: 10

- name: Display ES CSV version
  debug:
    msg: "{{ ES_SUB_STATUS.resources[0].status.installedCSV }}"

- name: Wait for Elasticseach Operator to Install
  community.kubernetes.k8s_info:
    api_version: operators.coreos.com/v1alpha1
    kind: ClusterServiceVersion
    namespace: openshift-operators-redhat
    name: "{{ ES_SUB_STATUS.resources[0].status.installedCSV }}"
  register: ES_STATUS
  until: "'InstallSucceeded' in ES_STATUS | json_query('resources[].status.reason | []')"
  retries: 90
  delay: 2
  ignore_errors: true

```

NOTE: Here we are deploying Elasticsearch Operator. We are deploying Elasticsearch Operator to `openshift-operators` Namespace which is already present on the cluser.

*Cluster Logging Operator Installation*:
```yaml

- name: Set Namespace for cluster logging
  community.kubernetes.k8s:
    state: present
    api_version: v1
    kind: Namespace
    merge_type:
      - strategic-merge
      - merge
    definition:
      metadata:
        name: openshift-logging
        annotations:
          openshift.io/node-selector: ""
        labels:
          openshift.io/cluster-monitoring: "true"

- name: Set Operator Group for cluster logging
  community.kubernetes.k8s:
    state: present
    api_version: operators.coreos.com/v1
    kind: OperatorGroup
    namespace: openshift-logging
    merge_type:
      - strategic-merge
      - merge
    definition:
      metadata:
        name: openshift-logging
      spec:
        targetNamespaces:
        - openshift-logging

- name: Ensure subscription existence for cluster logging operator
  community.kubernetes.k8s:
    state: present
    api_version: operators.coreos.com/v1alpha1
    kind: Subscription
    name: cluster-logging
    namespace: openshift-logging
    merge_type:
      - strategic-merge
      - merge
    definition:
      spec:
        channel: "{{ logging_channel }}"
        installPlanApproval: Automatic
        name: cluster-logging
        source: redhat-operators
        sourceNamespace: openshift-marketplace

- name: Verify cluster logging operator is created
  community.kubernetes.k8s_info:
    kind: Subscription
    namespace: openshift-logging
    name: cluster-logging
  register: CLO_SUB_EXISTENCE
  until: "CLO_SUB_EXISTENCE.failed == false"
  retries: 30
  delay: 10

- name: Get CLO Sub
  community.kubernetes.k8s_info:
    api_version: operators.coreos.com/v1alpha1
    kind: Subscription
    name: cluster-logging
    namespace: openshift-logging
  register: CLO_SUB_STATUS
  until: CLO_SUB_STATUS.resources[0].status.installedCSV is defined
  retries: 30
  delay: 10

- name: Display CSV version
  debug:
    msg: "{{ CLO_SUB_STATUS.resources[0].status.installedCSV }}"

- name: Wait for CLO Operator to Install
  community.kubernetes.k8s_info:
    api_version: operators.coreos.com/v1alpha1
    kind: ClusterServiceVersion
    namespace: openshift-logging
    name: "{{ CLO_SUB_STATUS.resources[0].status.installedCSV }}"
  register: CLO_STATUS
  until: "'InstallSucceeded' in CLO_STATUS | json_query('resources[].status.reason | []')"
  retries: 20
  delay: 10
  ignore_errors: true

```

NOTE: For creating the Namespace for Cluster Logging. Cluster Logging must be created in `openshift-logging` Namespace. We are also setting the label `openshift.io/cluster-monitoring: "true"`. This will include logging namespace in Cluster Monitoring list.

Once both Operators are deployed, we create ClusterLogging instance which will kick off EFK stack components creation:

*Cluster Logging Instance Creation*:
```yaml
- name: Deploy ClusterLogging
  community.kubernetes.k8s:
    state: present
    api_version: logging.openshift.io/v1
    kind: ClusterLogging
    name: "instance"                                      #1
    namespace: "openshift-logging"
    merge_type:
      - strategic-merge
      - merge
    definition:
      spec:
        managementState: "Managed"                        #2
        logStore:
          retentionPolicy:
            application:
              maxAge: 1d
            infra:
              maxAge: 7d
            audit:
              maxAge: 7d
          type: "elasticsearch"                           #3
          elasticsearch:
            nodeCount: 3                                  #4
            storage:
              storageClassName: "gp2"                     #5
              size: 200G
            resources:
              requests:
                memory: "8Gi"
            proxy:
              resources:
                limits:
                  memory: 256Mi
                requests:
                   memory: 256Mi
            redundancyPolicy: "SingleRedundancy"
            nodeSelector:
              node-role.kubernetes.io/infra: ""
            tolerations:
            - key: "node.ocs.openshift.io/storage"
              value: "true"
              effect: NoSchedule
        visualization:
          type: "kibana"                                 #6
          kibana:
            replicas: 1
            nodeSelector:
              node-role.kubernetes.io/infra: ""
            tolerations:
            - key: "node.ocs.openshift.io/storage"
              value: "true"
              effect: NoSchedule
        curation:
          type: "curator"                                #7
          curator:
            schedule: "30 1 * * *"
            nodeSelector:
              node-role.kubernetes.io/infra: ""
            tolerations:
            - key: "node.ocs.openshift.io/storage"
              value: "true"
              effect: NoSchedule
        collection:
          logs:
            type: "fluentd"                             #8
            fluentd:
              tolerations:
              - key: "node.ocs.openshift.io/storage"
                value: "true"
                effect: NoSchedule
```

1. The name must be `instance`. <br>
2. The cluster logging management state. In some cases, if you change the cluster logging defaults, you must set this to `Unmanaged`. However, an unmanaged deployment does not receive updates until the cluster logging is placed back into a managed state.<br>
3. Settings for configuring Elasticsearch. Using the Custom Resource (ClusterLogging), you can configure shard replication policy and persistent storage.<br>
4. Specify the number of Elasticsearch nodes. See the note that follows this list.<br>
5. Specify that each Elasticsearch node in the cluster is bound to a Persistent Volume Claim.<br>
6. Settings for configuring Kibana. Using the Custom Resource (ClusterLogging), you can scale Kibana for redundancy and configure the CPU and memory for your Kibana nodes.<br>
7. Settings for configuring Curator. Using the Custom Resource (ClusterLogging), you can set the Curator schedule.<br>
8. Settings for configuring Fluentd. Using the Custom Resource (ClusterLogging), you can configure Fluentd CPU and Memory limits.<br>

## Useage

This is the list of Pods created by Cluster Logging Stack. As mentioned previously, Fluentd is a DaemonSet running pod on each VM, we have 2 Elasticsearch pods and 1 Kibana pod:

```bash
$> oc get pods -n openshift-logging

NAME
cluster-logging-operator-cb795f8dc-xkckc

elasticsearch-cdm-b3nqzchd-1-5c6797-67kfz
elasticsearch-cdm-b3nqzchd-2-6657f4-wtprv

fluentd-2c7dg
fluentd-9z7kk
fluentd-br7r2
fluentd-fn2sb
fluentd-pb2f8
fluentd-zqgqx
fluentd-rf4er

kibana-7fb4fd4cc9-bvt4p
```
