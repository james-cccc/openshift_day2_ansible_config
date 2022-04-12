# Ansible Role to Setup Authentication

## OpenShift OAuth

The authentication operator is an OpenShift ClusterOperator. It installs and maintains the Authentication in a cluster and can be viewed with:

```bash
oc get clusteroperator authentication -o yaml
```

The identity providers in the role is for OpenID and HTPasswd.

For more about authentication and authorization, please refer to:

* [Understanding authentication](https://docs.openshift.com/container-platform/4.6/authentication/understanding-authentication.html)
* [Configuring a OpenID Connect identity provider](https://docs.openshift.com/container-platform/4.6/authentication/identity_providers/configuring-oidc-identity-provider.html)
* [Configuring a HTPasswd identity provider](https://docs.openshift.com/container-platform/4.6/authentication/identity_providers/configuring-htpasswd-identity-provider.html)
* [HTPASSWD](https://httpd.apache.org/docs/2.4/programs/htpasswd.html)

## Building the HTPASSWD file

Using the Ansible `htpasswd` module, we can create a `htpasswd` file with the users specified in the `groups_and_users` dictionary in the variables configuration. In the dictionary below, we are only using the `username` from each group.

```yaml
groups_and_users:
  cluster_admin:
    group_name: 'OpenShift Admins'
    username: cluster_admin_user
    password: redhat1
  cluster_reader:
    group_name: 'OpenShift ReadOnly'
    username: cluster_reader_user
    password: redhat1
  developer:
      group_name: 'OpenShift Developer'
      username: developer_user
      password: redhat1
  tester:
    group_name: 'OpenShift Tester'
    username: tester_user
    password: redhat1
```
For this example, we are using a password of `redhat1` for each username and saving the htpasswd file to `files/htpassword`.

```yaml
- name: Authentication via HTPasswd
  block:
    - name: Build HTPASSWD file
      htpasswd:
        state: present
        path: "{{ role_path }}/files/htpassword"
        name: "{{ item.value.username }}"
        password: "{{ item.value.password }}"
      with_dict: "{{ groups_and_users }}"
```

Next we are creating a Secret Object containing our `htpassword` file. The secret is holding base64 encoded data, so we are encoding our `htpassword` file content.

```yaml
- name: Create htpassword secret
  k8s:
    state: present
    api_version: v1
    kind: Secret
    namespace: openshift-config
    name: htpasswd
    definition:
      type: Opaque
      data:
        htpasswd: "{{ lookup('file', role_path + '/files/htpassword') | b64encode }}"
```

Next we are configuring the OAuth server to use the HTPasswd IdP from the secret by editing the spec of the cluster-wide `OAuth/cluster` object:

```yaml
- name: Create htpassword oauth config
  k8s:
    state: present
    api_version: config.openshift.io/v1
    kind: OAuth
    name: cluster
    definition:
      spec:
        identityProviders:
          - htpasswd:
              fileData:
                name: htpasswd
            mappingMethod: claim
            name: htpasswd
            type: HTPasswd
```

Finally, we'll clean up the role by removing the `htpasswd` file with our users and credentials. We can remove this as it's already been consumed and applied by Openshift.

```yaml
- name: Remove file from tmp directory
  file:
    path: "{{ role_path }}/files/htpassword"
    state: absent
```

The operator will now restart the OAuth server deployment and mount the new config. When the operator is available again (`oc get clusteroperator authentication`), you should be able to log in:

```yaml
oc login -u cluster_admin_user -p redhat1
```

At this stage, each user can log into the cluster, but they'll have no permissions and won't be able to interact with anything. This is due to the absence of RBAC (Role Based Access Control) roles being applied to each user.

## RBAC overview

Role-based access control (RBAC) objects determine whether a user is allowed to perform a given action within a project.

Cluster administrators can use the cluster roles and bindings to control who has various access levels to the OpenShift Container Platform platform itself and all projects.

Developers can use local roles and bindings to control who has access to their projects.

## Applying RBAC

To demonstrate RBAC we're going to apply 4 out-the-box Cluster Roles to groups that each user is a member of. Applying Cluster Roles to groups makes managing user permissions easier as we wouldn't have to configure granular access control for each user. Below is a table describing each user, the group they're a member of, the roles that'll be applied and an example of a user that'll use the specific Cluster Role. The description column provides an explanation of the Cluster Role being used.

User Type | Username | Group Name | Cluster Role | Description
--- | --- | --- | --- | ---
Cluster Admin | cluster_admin_user | OpenShift Admins | cluster_admin | A super-user that can perform any action in any project. When bound to a user with a local binding, they have full control over quota and every action on every resource in the project.
Cluster Reader | cluster_reader_user | OpenShift ReadOnly | cluster-reader | A user who can read, but not view, objects in the cluster.
Developer | developer_user | OpenShift Developer | admin | If used in a local binding, an admin user will have rights to view any resource in the project and modify any resource in the project except for quota.
Tester | tester_user | OpenShift Tester | view | A user who cannot make any modifications, but can see most objects in a project. They cannot view or modify roles or bindings.

### Creating Namespaces

This has not been implemented in the role tasks as of right now, however if you wish to add this functionality it will be explained below.

In this project we are binding the Cluster Roles to the deafult `openshift-config` namespace. However, when binding local Roles we bind them to the project we wish to give Groups or Users permissions for. We could add tasks to create test projects named `developer-project-(1-3)` which we'll bind our Roles to.

```yaml
# Create developer namespaces
- name: Create namespaces
  k8s:
    state: present
    kind: namespace
    name: '{{ item }}'
    merge_type:
      - strategic-merge
      - merge
  loop: '{{ project_names }}'          #1
```
1. Loops through the `project_names` list from variables and creates the projects

```yaml
# RBAC example developer namespaces
project_names:
    - developer-project-1
    - developer-project-2
    - developer-project-3
```

### Creating Groups

As we are binding our Roles to groups, we'll first need to create the groups. We create the `groups` specified in the `groups_and_users` dictionary.

```yaml
# Create groups
- name: Create cluster_admin_group user Group
  k8s:
    state: present
    api_version: user.openshift.io/v1
    kind: Group
    name: '{{ item.value.group_name }}'
    merge_type:
      - strategic-merge
      - merge
  with_dict: '{{ groups_and_users }}'             #1
```
1. Loops through the `groups_and_users` dictionary and creates the group based off `group_name`.

### Binding Cluster Roles to Groups

Below we'll bind our 4 Cluster Roles into the groups specified:

* Binding the `cluster_admin` Cluster Role to the `OpenShift Admins` group.

```yaml
- name: Bind cluster-admin role to the Openshift Admins group
  k8s:
    state: present
    api_version: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding                                    #1
    name: cluster-admin-cluster-admin-crb
    namespace: openshift-config                                 #2
    merge_type:
      - strategic-merge
      - merge
    definition:
      metadata:
        annotations:
          rbac.authorization.kubernetes.io/autoupdate: "true"   #3
      roleRef:
        apiGroup: rbac.authorization.k8s.io
        kind: ClusterRole
        name: cluster-admin                                     #4
      subjects:
      - apiGroup: rbac.authorization.k8s.io
        kind: Group
        name: '{{ groups_and_users.cluster_admin.group_name }}' #5
```

1. We use a ClusterRoleBinding because cluster-admin is cluster scope.
2. Deploy the ClusterRoleBinding to the `openshift-config` namespace.
3. Set autoupdate to `true` to ensure roles are reconciliated during a master restart.
4. Specify the ClusterRole `cluster-admin`.
5. Specify the `'Openshift Admins'` group name.

```yaml
- name: Bind cluster-reader role to the Cluster Reader
  k8s:
    state: present
    api_version: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding                                     #1
    name: cluster-admin-cluster-reader-crb
    namespace: openshift-config                                  #2
    merge_type:
      - strategic-merge
      - merge
    definition:
      metadata:
        annotations:
          rbac.authorization.kubernetes.io/autoupdate: "true"    #3
      roleRef:
        apiGroup: rbac.authorization.k8s.io
        kind: ClusterRole
        name: cluster-reader                                     #4
      subjects:
      - apiGroup: rbac.authorization.k8s.io
        kind: Group
        name: '{{ groups_and_users.cluster_reader.group_name }}' #5
```

1. We use a ClusterRoleBinding because cluster-reader is cluster scope.
2. Deploy the ClusterRoleBinding to the `openshift-config` namespace.
3. Set autoupdate to `true` to ensure roles are reconciliated during a master restart.
4. Specify the ClusterRole `cluster-reader` .
5. Specify the `'OpenShift ReadOnly'` group name.

```yaml
- name: Bind admin role to the Developer group
  k8s:
    state: present
    api_version: rbac.authorization.k8s.io/v1
    kind: RoleBinding                                           #1
    name: developer-group-rb
    namespace: '{{ item }}'                                     #2
    merge_type:
      - strategic-merge
      - merge
    definition:
      metadata:
        annotations:
          rbac.authorization.kubernetes.io/autoupdate: "true"   #3
      roleRef:
        apiGroup: rbac.authorization.k8s.io
        kind: ClusterRole
        name: admin                                             #4
      subjects:
      - apiGroup: rbac.authorization.k8s.io
        kind: Group
        name: '{{ groups_and_users.developer.group_name }}'     #5
  loop: '{{ project_names }}'
```
1. We use a RoleBinding as a developer user only needs a local namespace scope.
2. Apply the Role Binding to each `developer-project-(1-3)` namespace
3. Set autoupdate to `true` to ensure roles are reconciliated during a master restart.
4. Specify the ClusterRole `admin`. Although we're binding it to a local Role, the out-the-box Roles that come with Openshift are ClusterRoles.
5. Specify the `'OpenShift Developer'` group name.

```yaml
- name: Bind view role to the Tester group
  k8s:
    state: present
    api_version: rbac.authorization.k8s.io/v1
    kind: RoleBinding                                           #1
    name: tester-group-rb
    namespace: '{{ item }}'                                     #2
    merge_type:
      - strategic-merge
      - merge
    definition:
      metadata:
        annotations:
          rbac.authorization.kubernetes.io/autoupdate: "true"   #3
      roleRef:
        apiGroup: rbac.authorization.k8s.io
        kind: ClusterRole
        name: view                                              #4
      subjects:
      - apiGroup: rbac.authorization.k8s.io
        kind: Group
        name: '{{ groups_and_users.tester.group_name }}'        #5
  loop: '{{ project_names }}'
```

1. We use a RoleBinding as a developer user only needs a local project scope.
2. Apply the Role Binding to each `developer-project-(1-3)` project.
3. Set autoupdate to `true` to ensure roles are reconciliated during a master restart.
4. Specify the ClusterRole `view`. Although we're binding it to a local Role, the out-the-box Roles that come with Openshift are ClusterRoles.
5. Specify the `'OpenShift Tester'` group name.

## Binding our Users to Groups

Once each group has the correct permissions, we'll need to populate the groups. In this example we'll assign our users specified in the `groups_and_users` dictionary to their corresponding groups. However, once more users are made they can be added to any group and inherit that groups permissions. For example I can add the user `bob` to `'Openshift Admins'` and now `bob` will be a cluster admin.

```yaml
- name: Bind Users to Groups
  k8s:
    state: present
    api_version: user.openshift.io/v1
    name: "{{ item.value.group_name }}"      #1
    kind: Group
    merge_type:
      - strategic-merge
      - merge
    definition:
      users:
      - "{{ item.value.username }}"          #2
  with_dict: '{{ groups_and_users }}'        #3
```
1. Specifiy the group name.
2. Specify the User to add to the group.
3. Loop over the `'groups_and_users'` dictionary.
