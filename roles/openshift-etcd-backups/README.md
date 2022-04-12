# Ansible Role for ETCD Backups

## ETCD Backup CronJob Task

ETCD is the key-value store for OpenShift Container Platform, which persists the state of all resource objects.

Back up your cluster’s etcd data regularly and store in a secure location ideally outside the OpenShift Container Platform environment. Do not take an etcd backup before the first certificate rotation completes, which occurs 24 hours after installation, otherwise the backup will contain expired certificates. It is also recommended to take etcd backups during non-peak usage hours, as it is a blocking action.

Once you have an etcd backup, you can restore to a previous cluster state.

You can perform the etcd data backup process on any master host that has connectivity to the etcd cluster.

For more about ETCD Backup and Restore, please refer to [Backing up etcd](https://docs.openshift.com/container-platform/4.6/backup_and_restore/backing-up-etcd.html) and [Restoring to a previous cluster state](https://docs.openshift.com/container-platform/4.6/backup_and_restore/disaster_recovery/scenario-2-restoring-cluster-state.html).

In order to have etcd snapshots daily, we are creating a scheduled pod on OpenShift with privileges to run as root and mounting persistent storage where we are saving the snapshots. Snapshots can be used at a later time if you need to restore etcd.

You should only save a backup from a single master host. You do not need a backup from each master host in the cluster.

## Role Tasks

Below we are creating and Granting privileges to Service Account `approver` which will run the Backup Pod. We give this Service Account `cluster-admin` role and granting permisions to run containers as user `root` via SecurityContextConstraints.

*Setting up Role Based Access Control and Security Context Constraints*:
```yaml
    - name: Create Service Account approver
      community.kubernetes.k8s:
        state: present
        kind: ServiceAccount
        namespace: openshift-config
        name: etcd-backup

    - name: Create Cluster Role with SCC updates for Service Account
      community.kubernetes.k8s:
        state: present
        api_version: rbac.authorization.k8s.io/v1
        kind: ClusterRole
        name: use-privileged-scc-role
        merge_type:
          - strategic-merge
          - merge
        definition:
          rules:
          - apiGroups:
            - security.openshift.io
            resourceNames:
            - privileged
            resources:
            - securitycontextconstraints
            verbs:
            - use

    - name: Bind use-privileged-scc-role role to the approver Service Account
      community.kubernetes.k8s:
        state: present
        api_version: rbac.authorization.k8s.io/v1
        kind: RoleBinding
        name: use-privileged-scc-rolebinding
        namespace: openshift-config
        merge_type:
          - strategic-merge
          - merge
        definition:
          roleRef:
            apiGroup: rbac.authorization.k8s.io
            kind: ClusterRole
            name: use-privileged-scc-role
          subjects:
          - kind: ServiceAccount
            name: etcd-backup
            namespace: openshift-config
```

* **Create Service Account approver** - Ensures the`approver` Service Account exists
* **Create Cluster Role with SCC updates for Service Account** - Creates the Cluster Role with SCC changes for our `approver` SA
* **Bind use-privileged-scc-role role to the approver Service Account** - Binds the Cluster Role to the Service Account `approver`

*Setting up ETCD Backup PersistentVolume and ConfigMap*:
```yaml
    - name: Create ETCD PVC
      community.kubernetes.k8s:
        state: present
        kind: PersistentVolumeClaim
        api_version: v1
        name: etcd-backup
        namespace: openshift-config
        merge_type:
          - strategic-merge
          - merge
        definition:
          spec:
            accessModes:
              - ReadWriteOnce
            resources:
              requests:
                storage: 20Gi
            storageClassName: gp2
            volumeMode: Filesystem

    - name: Create ETCD Backup ConfigMap
      community.kubernetes.k8s:
        state: present
        kind: ConfigMap
        api_version: v1
        definition:
          metadata:
            name: etcd-backup-script
            namespace: openshift-config
          data:
            etcd-backup.sh: |
              #!/bin/bash

              DATE=$(date +%Y%m%dT%H%M%S)
              PATH=$PATH:/usr/bin/oc

              cp /usr/local/bin/cluster-backup.sh /tmp/
              ls -l /tmp/
              sed 's|dl_etcdctl|#dl_etcdctl|g' -i /tmp/cluster-backup.sh
              /tmp/cluster-backup.sh /assets/backup
              if [ $? -eq 0 ]; then
                  mkdir /etcd-backup/${DATE}
                  cp -r /assets/backup/*  /etcd-backup/${DATE}/
                  echo 'Copied backup files to PVC mount point.'
                  exit 0
              fi

              echo "Backup attempts failed. Please FIX !!!"
              exit 1
```

* **Create ETCD PVC** - Ensures a persistent volume exists that will be used to store the etcd backups
* **Create ETCD Backup ConfigMap** - Creates a shell script that the pod will use to execute the ETCD backup

*Setting up ETCD Backup CronJob*:
```yaml
    - name: Create ETCD Backup CronJob
      community.kubernetes.k8s:
        state: present
        kind: CronJob
        api_version: batch/v1beta1
        definition:
          metadata:
            name: cronjob-etcd-backup
            namespace: openshift-config
            labels:
              purpose: etcd-backup
          spec:
            schedule: '15 01 * * *'                                                           #1
            startingDeadlineSeconds: 200
            concurrencyPolicy: Forbid                                                         #2
            suspend: false
            jobTemplate:
              spec:
                backoffLimit: 0
                template:
                  spec:
                    nodeSelector:                                                             #3
                      node-role.kubernetes.io/master: ''
                    restartPolicy: Never
                    activeDeadlineSeconds: 200
                    serviceAccountName: etcd-backup
                    hostNetwork: true
                    containers:
                      - resources:
                          requests:
                            cpu: 300m
                            memory: 250Mi
                        terminationMessagePath: /dev/termination-log
                        name: etcd-backup
                        command:                                                              #4
                          - /bin/sh
                          - '-c'
                          - >-
                            /root/etcd-backup.sh && ls -1 /etcd-backup/* | sort -r | tail -n +6 | xargs rm -rf > /dev/null 2>&1
                        securityContext:                                                      #5
                          privileged: true
                        imagePullPolicy: IfNotPresent
                        volumeMounts:                                                         #6
                          - name: certs
                            mountPath: /etc/ssl/etcd/
                          - name: conf
                            mountPath: /etc/etcd/
                          - name: kubeconfig
                            mountPath: /etc/kubernetes/
                          - name: etcd-backup-script
                            mountPath: /root/etcd-backup.sh
                            subPath: etcd-backup.sh
                          - name: etcd-backup
                            mountPath: /etcd-backup
                          - name: scripts
                            mountPath: /usr/local/bin
                          - name: oc
                            mountPath: /usr/bin/oc
                        terminationMessagePolicy: FallbackToLogsOnError
                        image: "{{ etcd_image_tag.resources[0].spec.containers[0].image }}"   #7
                    serviceAccount: etcd-backup                                               #8
                    tolerations:                                                              #9
                      - operator: Exists
                        effect: NoSchedule
                      - operator: Exists
                        effect: NoExecute
                    volumes:                                                                 #10
                      - name: certs
                        hostPath:
                          path: /etc/kubernetes/static-pod-resources/etcd-member
                          type: ''
                      - name: conf
                        hostPath:
                          path: /etc/etcd
                          type: ''
                      - name: kubeconfig
                        hostPath:
                          path: /etc/kubernetes
                          type: ''
                      - name: scripts
                        hostPath:
                          path: /usr/local/bin
                          type: ''
                      - name: oc
                        hostPath:
                          path: /usr/bin/oc
                          type: ''
                      - name: etcd-backup
                        persistentVolumeClaim:
                          claimName: etcd-backup
                      - name: etcd-backup-script
                        configMap:
                          name: etcd-backup-script
                          defaultMode: 493
```

1. Crontab like schedule -> https://crontab.guru
2. If concurrencyPolicy is set to Forbid and a CronJob was attempted to be scheduled when there was a previous schedule still running, then it would count as missed. We do not want to run two etcd backups at the same time. Especially if current one is failing.
3. We are setting nodeSelector to schedule the pod only on the node with the label `node-role.kubernetes.io/master: ''`
4. in the `command` section we are providing the shell command which will be executed by the pod. In this case, we are running the etcd backup and keeping last 6 backup snapshots.
5. We are setting securityContext to `privileged: true` allowing the pod user to run as `root`.
6. We are mounting etcd configs and certificates from the Master host to the container keeping the same mount points pretending we are running etcd backup command directly on the Master host.
7. We are using etcd container image because it contains all required tools like `etcdctl`
8. We are setting the service account to be `approver` which has privileges to run containers as `root` user.
9. Since Master hosts have taints (required so that Schedule wouldn't schedule application pods on them) we need to tolerate those taints.
10. We are mounting directories (hostPath) from the Master host onto container.
