OpenShift Automation with Ansible
=================================

This repository contains a set of roles and playbooks to help with the day 2 configuration of OpenShift 4.

## Requirements

The following python3 packages are required:
- ansible
- jmespath
- openshift
- kubernetes
- passlib

```bash
pip3 install ansible
pip3 install jmespath
pip3 install openshift
pip3 install passlib
```

The following Ansible collections are required:
- community.kubernetes

```bash
ansible-galaxy collection install community.kubernetes
```

## Role Variables

The role variables are all set in a single file inside the vars directory - each cluster will have its own vars file. e.g.

```bash
ansible/vars/
├── config_usdev.yml
├── config_usuat.yml
└── config_usprod.yml
```

In addition to the variables files, sensitive variables are stored in Ansible Vault files.

For more information on Vault please see: [Encrypting content with Ansible Vault](https://docs.ansible.com/ansible/latest/user_guide/vault.html#encrypting-content-with-ansible-vault)

To encrypt a file to be a vault:

```bash
 ansible-vault encrypt vars/vault_usdev.yml
```

To then ammend that file:

```bash
 ansible-vault edit vars/vault_usdev.yml
```

## Playbook Usage and Configuration

To execute the automation run the playbook of the relevant cluster. The playbook points to the relevant kubeconfig and variables file(s).

If using a vault password file:
```bash
ansible-playbook playbooks/<cluster_config_file>.yaml --vault-password-file .vault_pass.txt
```

If you want a prommpt for the vault password:
```bash
ansible-playbook playbooks/<cluster_config_file>.yaml --ask-vault-pass
```

## Roles

There are multiple roles contained within this project with their documentation being available within the role structure. Clicking the links below will take you to the relevant docs for the role.

* [openshift-auth](roles/openshift-auth/README.md) - configures Authentication via OpenID and HTPasswd
* [openshift-certificates](roles/openshift-certificates/README.md) - configures Ingress Certificates the cluster
* [openshift-console](roles/openshift-console/README.md) - configures the OpenShift web console
* [openshift-etcd](roles/openshift-etcd/README.md) - encrypts the etcd database
* [openshift-etcd-backups](roles/openshift-etcd-backups/README.md) - creates an ETCD Backup CronJob
* [openshift-infra-nodes](roles/openshift-infra-nodes/README.md) - creates Infra nodes MachineSets
* [openshift-image-registry](roles/openshift-image-registry/README.md) - moves the image registry pods to the infra nodes
* [openshift-ingress](roles/openshift-ingress/README.md) - moves the ingress pods to the infra nodes
* [openshift-logging](roles/openshift-logging/README.md) - deploys openshift logging and places the relevant parts onto the infra nodes
* [openshift-monitoring](roles/openshift-monitoring/README.md) - moves the relevant parts of the monitoring stack onto the infra nodes
* [openshift-storage](roles/openshift-storage/README.md) - deploys openshift container storage and places the relevant parts onto the infra nodes
* [ibm-cp4i](roles/ibm-cp4i/README.md) - deploys IBM Cloud Paks

## General

This repository contains Ansible Roles for configuring the Red Hat Openshift Container Platform.

See the [official documentation for more information](https://docs.openshift.com/container-platform/4.6/) on installing and configuring the Openshift Container Platform (OCP).
