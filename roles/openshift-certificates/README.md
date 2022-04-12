# Ansible Role for Certificates

## OpenShift Ingress Certificates

All Worker VMs are available in Load Balancer backend pool even if only two of them have OpenShift Router Pods running (they can be recreated on another VM after restart). Traffic is sent to the first available pool member VM.

Ingress Controller creates OpenShift Router Pods which are Ingress points for traffic to enter the cluster. By default, Ingress Controller attach Certificate Operator precreated Self-Signed certificate for Ingress to the OpenShift Router Pods. We can replace it with a Company signed Wildcard Certificate.

This certificate will be used if SSL is going to be terminated on the OpenShift Routers (default option called `Edge` in OpenShift).

## Ingress Certificate Role

The required variables are to be set, these are the file locations of the certficates to be used:

```yaml
root_ca: "/root/openshift-dev/apps-certs/apps-cp4i-usdev-org.pem"
ingress_tls_crt: "/root/openshift-dev/apps-certs/apps-cp4i-usdev-tech-chain.pem"
ingress_tls_key: "/root/openshift-dev/apps-certs/apps-cp4i-usdev-key.pem"
```

# Role tasks

This role performs the following tasks:

* *Create Config Map for CA Bundle* - Here we are giving ConfigMap the name - custom-ca
* *Patch Global Proxy with new trustedCA bundle* - Here we are setting new ca-bundle as trusted
* *Create a Secret with crt and key* - Here we are creating a Kubernetes Secret object containing our Centrificate and Private Key
* *Patch Ingress Controller* - Here we tell Ingress Controller to use our certificate Secret as a default Ingress certificate

```yaml
---
  - name: Replacing the default ingress certificate
    block:

      - name: Create Config Map for CA Bundle
        community.kubernetes.k8s:
          state: present
          kind: ConfigMap
          name: custom-ca
          namespace: openshift-config
          merge_type:
            - strategic-merge
            - merge
          definition:
            data:
              ca-bundle.crt: "{{ lookup('file', '{{ root_ca }}', rstrip=False) }}"

      - name: Patch Global Proxy with new trustedCA bundle
        community.kubernetes.k8s:
          state: present
          merge_type:
            - strategic-merge
            - merge
          kind: Proxy
          name: cluster
          definition:
            spec:
              trustedCA:
                name: custom-ca

      - name: Create a Secret with crt and key
        community.kubernetes.k8s:
          state: present
          kind: Secret
          name: ingress-certificate
          namespace: openshift-ingress
          definition:
            data:
              tls.crt: "{{ lookup('file', '{{ ingress_tls_crt }}', rstrip=False) | b64encode }}"
              tls.key: "{{ lookup('file', '{{ ingress_tls_key }}', rstrip=False) | b64encode }}"
            type: kubernetes.io/tls

      - name: Patch Ingress Controller
        community.kubernetes.k8s:
          state: present
          merge_type:
            - strategic-merge
            - merge
          kind: IngressController
          name: default
          namespace: openshift-ingress-operator
          definition:
            spec:
              defaultCertificate:
                name: ingress-certificate

      - name: Cert update Wait
        pause:
          minutes: 20
```

NOTE: If Ingress Certificate expiration day is aproaching, update the variable providing new Ingress Certificates. Good practice is to keep the same name and only add expiry data as suffix (e.g. ingress-certificate-20220330) so it can be updated without downtime.

Once the Ingress Certificate is updated, login to OpenShift and remove the old secret:

```bash
oc delete secret ingress-certificate -n openshift-ingress
```
