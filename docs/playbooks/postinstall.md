# Playbook: Postinstall

The postinstall playbook helps you perform a variety of post-installation tasks.

Examples of tasks that come with the playbook distribution include:
- Initializing Solr collections and configs.
- Setting the HDFS umask post-deployment.
- Initializing Kafka topics.
- Initializing HDFS folders and encryption zones.
- Initializing Ranger policies (including key policies).
- Creating KTS keys.

These tasks can be configured individually, using the `postinstall_args` variable.

The configuration options expected differ from task to task.

## Provisioning credentials

A key feature of the postinstall role is that it will provision principals and temporary ranger policies which can be used to run the postinstall tasks. This helps restrict the permissions granted to the postinstall tasks and leaves a more understandable audit trail once completed. The details of these credentials are given in `postinstall/common/defaults` and can be extended.

These include:

- `hbase_admin`
- `hdfs_admin`
- `hive_admin`
- `solr_admin`
- `kafka_admin`

To use these credentials, please set `run_as` to the required provisioning principal (check that the postinstall task supports the `run_as` param).

**Note:** Do not set `run_as` if you cluster does not have Ranger – the playbook will use service user identities for the tasks.

## Local user

Unless root is required, postinstall tasks will run each command using the local user set in the variable `postinstall_local_user` (default `provision_user`).

You must ensure this user exists on each node of the cluster.

## Configuring standard tasks

### HDFS umask

The `hdfs_umask` task resets the HDFS umask according to the `postinstall_args.hdfs_umask` variable, restarts all dependent services and redeploys the client configs.

e.g.

```yaml
postinstall_args:
  hdfs_umask:
    cluster_name: "CDP-DC Base Cluster"
    umask: "027"
```

### HDFS Init

The `hdfs_init` task creates folders according to the `postinstall_args.hdfs_init` variable.

The `run_as` variable allows you to run the folder creation task as a particular provisioning user. This helps leave an understandable audit trail and allows us to restrict the permissions given to the provisioner.By default, we will fall back to the `hdfs` principal.

__Note:__ `chown` is run using the `hdfs` principal regardless of `run_as`.

Either `cluster_name` or `delegate_host` is required.

`delegate_host` specifies the host where the HDFS commands are run.

The folders are given as an array of dictionaries with the following entries:
```
- path:  string required e.g. /user/test
- mode: string optional e.g. "700"
- owner: string optional e.g. "test"
- mode: string optional e.g. "mode"
- acls:  array  optional e.g. [ "user:root:rwx" ]
```

e.g.

```yaml
provisioning_args:
  hdfs_init:
    run_as: hdfs_admin
    cluster_name: "CDP-DC Base Cluster"
    folders:
    - path: /user/test
      chmod: "700"
      chown: "test:test"
      acls
      - [ "group:hr:rwx" ]
```

### Kafka Init

The `kafka_init` task creates folders according to the `postinstall_args.kafka_init` variable.

The `run_as` variable allows you to run the topic creation task as a particular provisioning user. This helps leave an understandable audit trail and allows us to restrict the permissions given to the provisioner.By default, we will fall back to the `kafka` principal.

Either `cluster_name` or `bootstrap_host` is required.

The `bootstrap_port` variable is required and specifies the port which the Kafka broker is listening on.

The `security_protocol` variable is required and specifies the security protocol the client should use to connect to Kafka.

The topics are given as an array of dictionaries with the following entries:
```
name:               string     required e.g. topic1
configs:            dictionary optional e.g. { "min.insync.replicas": 3 }
replication_factor: int default(1)
partitions:         int default(1)
```

e.g.

```yaml
postinstall_args:
  kafka_init:
    cluster_name: "CDP-DC Base Cluster"
    bootstrap_port: 9093
    security_protocol: SASL_SSL
    topics:
    - name: topic1
      replication_factor: 3
      configs:
        min.insync.replicas: 2
```

### Ranger Init

The `ranger_init` task creates / updates policies according to the `postinstall_args.ranger_init` variable.

If `cluster_name` is specified, the url and credentials will be configured automatically. Otherwise, please set the following:

The `admin_url` variable is required and specifies the url of the Ranger admin server (including the port and the protocol).

The `admin_username` and `admin_password` variables are required and specifies the admin credentials that will be used to create / update the Ranger policies.

The policies are given as an array of dictionaries with the following entries:
```
name:     string required
plugin:   string required
resource: dictionary required
users:    dictionary optional
groups:   dictionary optional
```

Here, `plugin` stands for the name of the Ranger plugin corresponding to the service. By default, this tends to be the name of the service prefixed with `cm_` (e.g. `cm_hdfs`).

The format of `resource` differs from service to service. You can find some examples in `roles/postinstall/common/defaults` under `default_postinstall_policies`.

It may be helpful to construct the policies via the Ranger Web UI and then export the policies via the `Export` button on the main page. The `resources` dictionaries found in the exported file match the format required for this postinstall task.

The `users` and `groups` dictionaries are maps from the user or group to the required access. See the example below for more details....

e.g.

```yaml
postinstall_args:
  ranger_init:
    cluster_name: "CDP-DC Base Cluster"
    policies:
    - name: "etl dir"
      plugin: cm_hdfs
      resources:
        path:
          values: [ "/etl" ]
          isRecursive: true
      groups:
        etl:
          conditions: [] # (default [])
          delegateAdmin: false # (default false)
          accesses: [ "read", "write", "execute" ]
```

#### Example Policies

**Hadoop SQL**

```yaml
- name: External Database
  plugin: cm_hive
  resources:
    database:
      values:
        - "external"
    table:
      values:
        - "*"
    column:
      values:
        - "*"
  groups:
    developers:
      accesses:
        - select
        - update
        - create
        - drop
        - alter
        - read
        - write
        - refresh

- name: External Database URL
  plugin: cm_hive
  resources:
    url:
      values:
        - "hdfs:///external/*"
  groups:
    developers:
      accesses:
        - read
```

**Kafka**

```yaml
- name: Application Topic
  plugin: cm_kafka
  resources:
    topic:
      values:
        - "application_topic"
  groups:
    developers:
      accesses:
        - publish
        - consume
        - configure
        - describe
        - create
        - delete
        - describe_configs
        - alter_configs
        - alter
    etc-consumer:
      accesses:
        - consume
    etc-producer:
      accesses:
        - publish
- name: Application Topic Consumer Groups
  plugin: cm_kafka
  resources:
    consumergroup:
      values:
        - "application_topic_cg_*"
  groups:
    developers:
      accesses:
        - consume
    etc-consumer:
      accesses:
        - consume
```

**NiFi**

**Note:** The examples below make references to UUIDs. These are visible within the NiFi UI. If more granular privileges are required, please make use of the UUIDs in-place of the asterisk.

```yaml
# write on /restricted-components allows the user to use restricted
# components – components that have elevated privliedges (marked
# with a shield).
- name: restricted components
  plugin: cdp-dc_cluster_nifi
  resources:
    nifi-resource:
      values:
        - "/restricted-components"
  groups:
    nifi-developer:
      accesses:
        - read
        - write
    nifi-flow-operator:
      accesses:
        - read
        - write
    nifi-cluster-operator:
      accesses:
        - read
        - write

# read on /provenance allows the user to view the provenance page.
# you must also give the user permissions to view the provenance data
# itself (see below).
- name: provenance
  plugin: cdp-dc_cluster_nifi
  resources:
    nifi-resource:
      values:
        - "/provenance"
  groups:
    nifi-developer:
      accesses:
        - read
    nifi-flow-operator:
      accesses:
        - read
    nifi-cluster-operator:
      accesses:
        - read

# all users must have read on /flow to view the ui.
- name: flow
  plugin: cdp-dc_cluster_nifi
  resources:
    nifi-resource:
      values:
        - "/flow"
  groups:
    nifi-developer:
      accesses:
        - read
    nifi-flow-operator:
      accesses:
        - read
    nifi-cluster-operator:
      accesses:
        - read

# read on /controller allows the user to view information about the
# nifi cluster, including controller services and reporing services,
# as well as information about the nifi node instances.
# write on /controller allows the user to modify the controller
# services, reporting services and nifi node state.
- name: controller
  plugin: cdp-dc_cluster_nifi
  resources:
    nifi-resource:
      values:
        - "/controller"
  groups:
    nifi-cluster-operator:
      accesses:
        - read
        - write

# read on /process-groups/<uuid> allows the user to view a process group
# and all of the processors and process groups enclosed
# write on /process-groups/<uuid> allows the user to modify the flow 
# contained within the process group and start and stop processors
# within
- name: all process groups
  plugin: cdp-dc_cluster_nifi
  resources:
    nifi-resource:
      values:
        - "/process-groups/*"
  groups:
    nifi-developer:
      accesses:
        - read
        - write
    nifi-flow-operator:
      accesses:
        - read
        - write
    nifi-cluster-operator:
      accesses:
        - read
        - write

# read on /data/process-groups/<uuid> allows the user to view the data
# within the process group. the user will be allowed to read data in 
# queues.
# write on /data/process-groups/<uuid> allows the user to modify data
# in the queue (e.g. clear the queue).
- name: all process group data
  plugin: cdp-dc_cluster_nifi
  resources:
    nifi-resource:
      values:
        - "/data/process-groups/*"
  groups:
    nifi-developer:
      accesses:
        - read
        - write
    nifi-flow-operator:
      accesses:
        - read
        - write
    nifi-cluster-operator:
      accesses:
        - read
        - write

# read on /controller-services/<uuid> allows the user to view the
# configuration of the controller service (e.g. ssl context).
# write on /controller-services/<uuid> allows the user to start and
# stop the controller service and modify its configuration.
# neither read nor write should be required to make use of the service
# within the flow.
- name: all controller services
  plugin: cdp-dc_cluster_nifi
  resources:
    nifi-resource:
      values:
        - "/controller-services/*"
  groups:
    nifi-flow-operator:
      accesses:
        - read
    nifi-cluster-operator:
      accesses:
        - read
        - write

# read on /provenance-data/process-groups/<uuid> is required, in
# addition to read on /provenance to read the process group's provenance
# information.
- name: all provenance data
  plugin: cdp-dc_cluster_nifi
  resources:
    nifi-resource:
      values:
        - "/provenance-data/process-groups/*"
  groups:
    developer:
      accesses:
        - read
    nifi-flow-operator:
      accesses:
        - read
    nifi-cluster-operator:
      accesses:
        - read
```

**Solr**

```yaml
- name: Test Collection
  plugin: cm_solr
  resources:
    collection:
      values:
        - "test_collection"
  groups:
    developer:
      accesses:
        - query
        - update
        - others
        - solr_admin
    etc-dashboard:
      accesses:
        - query
```

### Ranger KMS Init

The `ranger_kms_init` task creates / updates policies according to the `postinstall_args.ranger_kms_init` variable.

It is identical to the `ranger_init` task above, except it uses the key admin credentials instead of admin.

### Solr Init

The `solr_init` task creates Solr configs and collections according to the `postinstall_args.solr_init` variable.

The `run_as` variable allows you to run the config and collection creation tasks as a particular provisioning user. This helps leave an understandable audit trail and allows us to restrict the permissions given to the provisioner.By default, we will fall back to the `solr` principal.

Either `cluster_name` or `solr_server` is required.

The configs are given as an array of dictionaries with the following entries:
```
name: string required
base: string required
p:    string required
```

The collections are given as an array of dictionaries with the following entries:
```
name:               string required
config:             string required
shards:             int    required
replication_factor: int    required
opts:               string optional
```

e.g.

```yaml
postinstall_args:
  solr_init:
    cluster_name: "CDP-DC Base Cluster"
    run_as: solr_admin
    configs:
    - name: config1
      base: schemalessTemplateSecure
    collections:
    - name: collection1
      config: config1
      shards: 1
      replication_factor: 1
```

### Hive Init

The `hive_init` task executes HQL statements via `beeline` according to the `postinstall_args.hive_init` variable.

The `run_as` variable allows you to run the HQL statements a particular provisioning user. This helps leave an understandable audit trail and allows us to restrict the permissions given to the provisioner.By default, we will fall back to the `hive` principal.

Either `cluster_name` or `hs2_host` is required.

You can also set `delegate_host` if you want to run the beeline command on a separate host.

Each HQL statement is given as an entry of the `ddl` array (see below).

Finally, you must set `kerberos` and `tls` (both booleans).

Additional variables include:

- `hs2_port`
- `truststore`
- `truststore_password`

e.g.

```yaml
postinstall_args:
  hive_init:
    cluster_name: "CDP-DC Base Cluster"
    ddl:
    - "create table hive_test (fname STRING, lname STRING);"
    - "create table hive_test_2 (fname STRING, lname STRING);"
    kerberos: true
    tls: true
    run_as: hive_admin
```

### Impala Init

The `impala_init` task executes HQL statements via `impala-shell` according to the `postinstall_args.impala_init` variable.

It is very similar to the `hive_init` task.

The `run_as` variable allows you to run the HQL statements a particular provisioning user. This helps leave an understandable audit trail and allows us to restrict the permissions given to the provisioner.By default, we will fall back to the `hive` principal.

Either `cluster_name` or `impala_daemon` is required.

You can also set `delegate_host` if you want to run the `impala-shell` command on a separate host.

Each HQL statement is given as an entry of the `ddl` array (see below).

Finally, you must set `kerberos` and `tls` (both booleans) and the `truststore` location, if TLS is enabled (the `impala-shell` will not use the OS truststore).

Additional variables include:

- `port`

e.g.

```yaml
postinstall_args:
  impala_init:
    cluster_name: "CDP-DC Base Cluster"
    ddl:
    - "create table impala_test (fname STRING, lname STRING);"
    - "create table impala_test_2 (fname STRING, lname STRING);"
    kerberos: true
    tls: true
    truststore: /etc/ipa/ca.crt
    run_as: hive_admin
```

### Hue Init

The `hue_init` task allows you to setup users and groups – without having to rely on the initial superuser.

Either `cluster_name` or `hue_server_host` is required.

**Note:** It must be the Hue server and not a load balancer.

You'll need to provide `database_password` – the Hue database password and optionally, `ldap_password` – the LDAP bind password and `key_password` – the TLS key password.

The users and groups to be imported are set using the following variables:

- `local_users`
- `ldap_users`
- `ldap_groups`

`local_users` is an array of dictionaries with the following entries:

- `name: string`
- `password: string`
- `is_superuser: bool`

**Note:** The password set above will be visible in the Ansible output.

**Note:** You must setup the Hue LDAP configurations before attempting to import LDAP users and groups.

`ldap_users` is an array of dictionaries with the following entries:

- `name: string`
- `sync_groups: bool`
- `is_superuser: bool`

`ldap_groups` is an array of dictionaries with the following entries:

- `name: string`
- `import_members: bool`
- `permissions: array of dictionaries`
  * `app: string`
  * `action: string`

Additional variables include:

- `hue_server_port`

e.g.

```yaml
postinstall_args:
  hue_init:
    cluster_name: "CDP-DC Base Cluster"
    database_password: changeme
    ldap_password: changeme
    local_users:
    - name: test
      password: test
      is_superuser: true
    - name: test_read
      password: test
      is_superuser: false
    ldap_users:
    - name: cmadmin
      sync_groups: true
      is_superuser: true
    ldap_groups:
    - name: ipausers
      import_members: true
      permissions: []
```

### CDSW

The `cdsw_init` task allows you to setup LDAP for CDSW.

It makes use of the Playbook's auth providers.

You'll need to set the following variables:

- `cluster_name` or `delegate_host`
- `cdsw_admin_group`
- `auth_provider` (when `service_auth_provider` is not defined)

And optionally:

- `smtp_host`
- `smtp_port`
- `smtp_from`

The delegate host will be used to execute the required commands via `kubbectl`.

e.g.

```yaml
postinstall_args:
  cdsw_init:
    cluster_name: "CDP-DC Base Cluster"
    cdsw_admin_group: cdsw_admins
    auth_provider: freeipa
```

## Complete example

```yaml
---
cluster_definition: /path/to/definition
cloudera_manager_version: 7.1.3
cloudera_manager_repo_username: "{{ vault__cloudera_manager_repo_username }}"
cloudera_manager_repo_password: "{{ vault__cloudera_manager_repo_password }}"
cloudera_manager_license_type: enterprise
cloudera_manager_license_file: testing/license.txt
cloudera_manager_options:
  CUSTOM_BANNER_HTML: "Cloudera Ansible Playbook v2 - {{ cluster_definition }}"

postinstall_default_cluster: "CDP-DC Base Cluster"
postinstall_args:
  impala_init:
    ddl:
    - "create table impala_test (fname STRING, lname STRING);"
    - "create table impala_test_2 (fname STRING, lname STRING);"
    kerberos: true
    tls: true
    truststore: /etc/ipa/ca.crt
    run_as: hive_admin
  hdfs_init:
    folders:
    - path: "/test"
      mode: "700"
      owner: test
      group: test
    - path: "/external/test"
      mode: "750"
      group: "test"
      key: "test_key"
  hdfs_umask:
    umask: "027"
  hive_init:
    ddl:
    - "create table hive_test (fname STRING, lname STRING);"
    - "create table hive_test_2 (fname STRING, lname STRING);"
    kerberos: true
    tls: true
    run_as: hive_admin
  hue_init:
    database_password: changeme
    ldap_password: changeme
    local_users:
    - name: test
      password: test
      is_superuser: true
    - name: test_read
      password: test
      is_superuser: false
    ldap_users:
    - name: cmadmin-5c88ea6d
      sync_groups: true
      is_superuser: true
    ldap_groups:
    - name: ipausers
      import_members: true
      permissions: []
  kafka_init:
    bootstrap_port: 9093
    security_protocol: SASL_SSL
    topics:
    - name: test1
      replication_factor: 2
      partitions: 1
      configs:
        min.insync.replicas: 2
    - name: test2
      replication_factor: 1
      partitions: 2
  key_init:
    run_as: key_admin
    enc_keys:
    - name: test_key
  ranger_init:
    policies:
    - name: "hdfs testing"
      plugin: cm_hdfs
      resources:
        path:
          values: ["/test/*" ]
          isRecursive: true
      users:
        test:
          accesses: ["read", "write", "execute"]
      groups:
        hr: 
          accesses: ["read", "execute"]
  ranger_kms_init:
    policies:
    - name: Test key policy
      plugin: cm_kms
      resources:
        keyname:
          values:
          - "test_key"
      users:
        admin:
          accesses: ["decrypteek"]
  solr_init:
    configs:
    - name: config1
      base: managedTemplateSecure
    - name: config2
      base: managedTemplateSecure
    collections:
    - name: collection1
      config: config1
      shards: 1
      replication_factor: 1
    - name: collection2
      config: config2
      shards: 1
      replication_factor: 2
```

## Configuring custom tasks

You may wish to run a postinstall task from outside of the playbook distribution.

Each external postinstall task should come with its own documentation on how to integrate it with the playbook.
