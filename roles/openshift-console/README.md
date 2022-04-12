# Ansible Role for OpenShift Console Customisations

## OpenShift

For more information, please refer to: [Customizing the web console in OpenShift Container Platform](https://docs.openshift.com/container-platform/4.6/web_console/customizing-the-web-console.html)

## Role Task

You can create custom branding by adding a custom logo and/or a custom product name. The example below just ilustrates adding the browsers tab display name.

*Adding a custom product name*:
```yaml
---
- name: Set Custom Console Banner name
  community.kubernetes.k8s:
    api_version: operator.openshift.io/v1
    state: present
    merge_type:
      - strategic-merge
      - merge
    kind: Console
    name: cluster
    definition:
      spec:
        customization:
          customProductName: "{{ cluster_name }}"
```

Notification banners can help easily distinguish the cluster you are logged into by setting the cluster name in the banner along with colour coding enviornments e.g. PROD=Red, UAT=Blue, DEV=Green etc.  

*Creating a custom notification banner bar*:
```yaml
- name: Set custom notification banner
  community.kubernetes.k8s:
    api_version: console.openshift.io/v1
    state: present
    merge_type:
      - strategic-merge
      - merge
    kind: ConsoleNotification
    definition:
      metadata:
        name: example
      spec:
        text: "{{ cluster_name }}"
        location: BannerTop
        color: "{{ text_colour }}"
        backgroundColor: "{{ banner_colour }}"
```

There are additional console customizations which can be done, such as customizing the login page. If you wish to add these please refer to the above link to the official documentattion and add additional Ansible tasks to this role in the same format.
