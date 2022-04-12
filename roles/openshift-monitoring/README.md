# Ansible Role for Monitoring Configuration

## OpenShift Monitoring

OpenShift Container Platform includes a pre-configured, pre-installed, and self-updating monitoring stack that provides monitoring for core platform components. OpenShift Container Platform delivers monitoring best practices out of the box. A set of alerts are included by default that immediately notify cluster administrators about issues with a cluster. Default dashboards in the OpenShift Container Platform web console include visual representations of cluster metrics to help you to quickly understand the state of your cluster.

For more information please refer to:

* [Understanding the monitoring stack](https://docs.openshift.com/container-platform/4.6/monitoring/understanding-the-monitoring-stack.html)
* [Configuring the monitoring stack](https://docs.openshift.com/container-platform/4.6/monitoring/configuring-the-monitoring-stack.html)

## Role Tasks

*Monitoring Configuraton*:
```yaml
---
- name: Configure Monitoring and Move Monitoring to Infra nodes
  community.kubernetes.k8s:
    api_version: v1
    state: present
    kind: ConfigMap
    name: cluster-monitoring-config
    namespace: openshift-monitoring
    merge_type:
      - strategic-merge
      - merge
    definition:
      data:
        config.yaml: |+
          enableUserWorkload: true
          alertmanagerMain:
            volumeClaimTemplate:
              spec:
                storageClassName: gp2                           #1
                volumeMode: Filesystem
                resources:
                  requests:
                    storage: 40Gi                               #2
            nodeSelector:
              node-role.kubernetes.io/infra: ""                 #3
            tolerations:
            - key: "node.ocs.openshift.io/storage"              #4
              value: "true"
              effect: NoSchedule
          prometheusK8s:
            retention: 12h                                      #5
            volumeClaimTemplate:
              spec:
                storageClassName: gp2                           #6
                volumeMode: Filesystem
                resources:
                  requests:
                    storage: 200Gi                              #7
            nodeSelector:
              node-role.kubernetes.io/infra: ""
            tolerations:
            - key: "node.ocs.openshift.io/storage"
              value: "true"
              effect: NoSchedule
          prometheusOperator:
            nodeSelector:
              node-role.kubernetes.io/infra: ""
            tolerations:
            - key: "node.ocs.openshift.io/storage"
              value: "true"
              effect: NoSchedule
          grafana:
            nodeSelector:
              node-role.kubernetes.io/infra: ""
            tolerations:
            - key: "node.ocs.openshift.io/storage"
              value: "true"
              effect: NoSchedule
          k8sPrometheusAdapter:
            nodeSelector:
              node-role.kubernetes.io/infra: ""
            tolerations:
            - key: "node.ocs.openshift.io/storage"
              value: "true"
              effect: NoSchedule
          kubeStateMetrics:
            nodeSelector:
              node-role.kubernetes.io/infra: ""
            tolerations:
            - key: "node.ocs.openshift.io/storage"
              value: "true"
              effect: NoSchedule
          telemeterClient:
            nodeSelector:
              node-role.kubernetes.io/infra: ""
            tolerations:
            - key: "node.ocs.openshift.io/storage"
              value: "true"
              effect: NoSchedule
          openshiftStateMetrics:
            nodeSelector:
              node-role.kubernetes.io/infra: ""
            tolerations:
            - key: "node.ocs.openshift.io/storage"
              value: "true"
              effect: NoSchedule
          thanosQuerier:
            nodeSelector:
              node-role.kubernetes.io/infra: ""
            tolerations:
            - key: "node.ocs.openshift.io/storage"
              value: "true"
              effect: NoSchedule
```

1. The name of the storage class for alertmanager to use.<br>
2. The size of the persistent volumes that alertmanager will create.<br>
3. The setting for the pod to select nodes with the specified label.<br>
4. The toleration to match the taint for the 'infra' nodes.<br>
5. Prometheus log retention period.<br>
6. The name of the storage class for Prometheus to use.<br>
7. The size of the persistent volumes that Prometheus will create.<br>
