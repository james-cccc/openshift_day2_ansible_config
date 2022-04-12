# Ansible Role for ETCD Encryption

## ETCD Encryption and Verification

ETCD is the key-value store for OpenShift Container Platform, which persists the state of all resource objects.

For more information, please refer to: [Encrypting etcd data](https://docs.openshift.com/container-platform/4.6/security/encrypting-etcd.html)

## Role Tasks

Below we are setting the etcd encryption to true:

*Encrypting ETCD*:
```yaml
- name: etcd encryption
  block:

    - name: Ensure etcd is encrypted
      community.kubernetes.k8s:
        state: present
        merge_type:
          - strategic-merge
          - merge
        kind: APIServer
        name: cluster
        definition:
          spec:
            encryption:
              type: aescbc
```

NOTE: The aescbc type means that AES-CBC with PKCS#7 padding and a 32 byte key is used to perform the encryption.

The encryption process starts. It can take 20 minutes or longer for this process to complete, depending on the size of your cluster. Thus the role has some validation logic to verify encrypted has completed:

*Verification*:
```yaml
    - name: Wait for OpenShift API server encryption to complete
      community.kubernetes.k8s_info:
        api_version: v1
        kind: OpenShiftAPIServer
        name: cluster
      register: OPENSHIFT_ENCRYPTION_STATUS
      until: "'EncryptionCompleted' in OPENSHIFT_ENCRYPTION_STATUS | json_query('resources[].status.conditions[?type==`Encrypted`].reason | []')"
      retries: 90
      delay: 90
      ignore_errors: true

    - name: Wait for Kubernetes API server encryption to complete
      community.kubernetes.k8s_info:
        api_version: v1
        kind: KubeAPIServer
        name: cluster
      register: KUBE_ENCRYPTION_STATUS
      until: "'EncryptionCompleted' in KUBE_ENCRYPTION_STATUS | json_query('resources[].status.conditions[?type==`Encrypted`].reason | []')"
      retries: 90
      delay: 90
      ignore_errors: true
```

The below code will just display that encypted has been comepleted in the Ansible play.

*Display Encrypted Status*:
```yaml
- name: Get encryption status
  block:

    - name: Retrieve OpenShift API server encryption status
      community.kubernetes.k8s_info:
        api_version: v1
        kind: OpenShiftAPIServer
        name: cluster
      register: OPENSHIFT_ENCRYPTION_STATUS

    - name: Retrieve Kubernetes API server encryption status
      community.kubernetes.k8s_info:
        api_version: v1
        kind: KubeAPIServer
        name: cluster
      register: KUBE_ENCRYPTION_STATUS

    - name: Display OpenShift API server encryption status
      debug:
        msg: "{{ OPENSHIFT_ENCRYPTION_STATUS | json_query('resources[].status.conditions[?type==`Encrypted`].reason ') }}'"

    - name: Display Kubernetes API server encryption status
      debug:
        msg: "{{ KUBE_ENCRYPTION_STATUS | json_query('resources[].status.conditions[?type==`Encrypted`].reason ') }}'"
```
