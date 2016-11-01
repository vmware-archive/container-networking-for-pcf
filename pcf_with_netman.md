## Deploying netman to PCF 1.8

1. Deploy OpsMan 1.8
2. Deploy the ERT 1.8 tile
   - enable TCP routing
3. Follow [these instructions](https://docs.pivotal.io/pivotalcf/1-7/customizing/trouble-advanced.html) for accessing the BOSH director from the OpsMan VM.
4. Upload a stemcell running Linux kernel 4.4 to the BOSH director.  Versions >= 3262.5 should work.
5. Apply the neccessary changes to the deployment manifest (see below)
6. Try deploying, see that certain releases ([netman-release](https://github.com/cloudfoundry-incubator/netman-release) and [garden-runc-release](http://bosh.io/releases/github.com/cloudfoundry-incubator/garden-runc-release?all=1)) are not found. Upload them: be sure to grab release versions matching those specified in your manfiest.
7. Try deploying again should now see errors because certain releases are not compiled for the new stemcell (cf, diego, cflinuxfs...). Grab the releases from bosh.io and upload.  Be sure to get the right versions for your manifest.
8. Deploy should succeed

```diff
+properties:
+  policy-server:
+    database:
+      databases:
+      - name: policy_server
+        tag: whatever
+      db_scheme: postgres
+      port: 5432
+      roles:
+      - name: policy_server
+        password: REPLACE-policy-server-db-password
+        tag: admin
+    database_password: REPLACE-policy-server-db-password
+    server:
+      database:
+        host: policy-server-db.service.cf.internal
+        password: REPLACE-policy-server-db-password
+        type: postgres
+      skip_ssl_validation: true
+      uaa_client_secret: REPLACE-policy-server-uaa-client-secret
+      uaa_url: https://uaa.system.fez.c2c.cf-app.com
+  system_domain: system.fez.c2c.cf-app.com
+  uaa:
+    clients:
+      network-policy:
+        secret: REPLACE-policy-server-uaa-client-secret
+  cni-flannel:
+    etcd_endpoints:
+    - http://netman-db.service.cf.internal:4001
+  vxlan-policy-agent:
+    policy_server_url: http://policy-server.service.cf.internal:4002
+  garden:
+    allow_host_access:
+    allow_networks:
+    default_container_grace_time: 0
+    deny_networks:
+    - 0.0.0.0/0
+    destroy_containers_on_start: true
+    disk_quota_enabled:
+    graph_cleanup_threshold_in_mb: 0
+    insecure_docker_registry_list:
+    listen_address:
+    listen_network:
+    log_level: debug
+    network_mtu:
+    network_plugin: "/var/vcap/packages/runc-cni/bin/garden-external-networker"
+    network_plugin_extra_args:
+    - "--configFile=/var/vcap/jobs/garden-cni/config/adapter.json"
+    persistent_image_list:
+    - "/var/vcap/packages/cflinuxfs2/rootfs"
+  garden-cni:
+    cni_config_dir: "/var/vcap/jobs/cni-flannel/config/cni"
+    cni_plugin_dir: "/var/vcap/packages/flannel/bin"
+    netman_url: http://127.0.0.1:4007
```


```diff
 releases:
-- name: garden-linux
-  version: 0.339.0
+- name: garden-runc
+  version: latest
+- name: netman
+  version: latest
```

```diff
stemcells:
 - alias: bosh-aws-xen-hvm-ubuntu-trusty-go_agent
   os: ubuntu-trusty
   version: '3262.2'
+- alias: bosh-aws-xen-ubuntu-trusty-go_agent
+  os: ubuntu-trusty
+  version: '3262.5'
```

```diff
 instance_groups:
+- instances: 1
+  name: policy-server-db
+  vm_type: m3.medium
+  lifecycle: service
+  stemcell: bosh-aws-xen-hvm-ubuntu-trusty-go_agent
+  azs:
+  - us-west-2a
+  properties:
+    consul:
+      encrypt_keys:
+      - REPLACE-consul-encrypt-key
+      ca_cert: |
+        -----BEGIN CERTIFICATE-----
        REPLACE-consul-ca-cert
+        -----END CERTIFICATE-----
+      agent_cert: |
+        -----BEGIN CERTIFICATE-----
        REPLACE-consul-agent-cert
+        -----END CERTIFICATE-----
+      agent_key: |
+        -----BEGIN RSA PRIVATE KEY-----
        REPLACE-consul-agent-key
+        -----END RSA PRIVATE KEY-----
+      server_cert: |
+        -----BEGIN CERTIFICATE-----
        REPLACE-consul-server-cert
+        -----END CERTIFICATE-----
+      server_key: |
+        -----BEGIN RSA PRIVATE KEY-----
        REPLACE-consul-server-key
+        -----END RSA PRIVATE KEY-----
+      agent:
+        domain: cf.internal
+        servers:
+          lan:
+          - 10.0.16.15
+        services:
+          policy-server-db:
+            check:
+              interval: 5s
+              script: "/bin/true"
+            name: policy-server-db
+  templates:
+  - name: postgres
+    release: netman
+  - name: consul_agent
+    release: cf
+  env:
+    bosh:
+      password: "REPLACE-env-bosh-passowrd"
+  update:
+    serial: true
+    max_in_flight: 1
+  networks:
+  - name: REPLACE-network-name
+    default:
+    - dns
+    - gateway
+  persistent_disk_type: '1024'
+- instances: 1
+  name: policy-server
+  vm_type: m3.medium
+  lifecycle: service
+  stemcell: bosh-aws-xen-hvm-ubuntu-trusty-go_agent
+  azs:
+  - us-west-2a
+  properties:
+    consul:
+      encrypt_keys:
+      - REPLACE-consul-encrypt-key
+      ca_cert: |
+        -----BEGIN CERTIFICATE-----
        REPLACE-consul-ca-cert
+        -----END CERTIFICATE-----
+      agent_cert: |
+        -----BEGIN CERTIFICATE-----
        REPLACE-consul-agent-cert
+        -----END CERTIFICATE-----
+      agent_key: |
+        -----BEGIN RSA PRIVATE KEY-----
        REPLACE-consul-agent-key
+        -----END RSA PRIVATE KEY-----
+      server_cert: |
+        -----BEGIN CERTIFICATE-----
        REPLACE-consul-server-cert
+        -----END CERTIFICATE-----
+      server_key: |
+        -----BEGIN RSA PRIVATE KEY-----
        REPLACE-consul-server-key
+        -----END RSA PRIVATE KEY-----
+      agent:
+        domain: cf.internal
+        servers:
+          lan:
+          - 10.0.16.15
+        services:
+          policy-server:
+            check:
+              interval: 5s
+              script: "/bin/true"
+            name: policy-server
+    nats:
+      user: nats
+      password: REPLACE-nats-password
+      port: 4222
+      machines:
+      - 10.0.16.16
+    route_registrar:
+      routes:
+      - name: policy-server
+        port: 4002
+        registration_interval: 20s
+        uris:
+        - api.fez.c2c.cf-app.com/networking
+  templates:
+  - name: policy-server
+    release: netman
+  - name: route_registrar
+    release: cf
+  - name: consul_agent
+    release: cf
+  env:
+    bosh:
+      password: "REPLACE-env-bosh-passowrd"
+  update:
+    serial: true
+    max_in_flight: 1
+  networks:
+  - name: auniquenameforthisnetwork
+    default:
+    - dns
+    - gateway
+  persistent_disk_type: '1024'
+- instances: 1
+  name: flannel_etcd
+  azs:
+  - us-west-2a
+  vm_type: m3.medium
+  stemcell: bosh-aws-xen-hvm-ubuntu-trusty-go_agent
+  properties:
+    consul:
+      encrypt_keys:
+      - REPLACE-consul-encrypt-key
+      ca_cert: |
+        -----BEGIN CERTIFICATE-----
        REPLACE-consul-ca-cert
+        -----END CERTIFICATE-----
+      agent_cert: |
+        -----BEGIN CERTIFICATE-----
        REPLACE-consul-agent-cert
+        -----END CERTIFICATE-----
+      agent_key: |
+        -----BEGIN RSA PRIVATE KEY-----
        REPLACE-consul-agent-key
+        -----END RSA PRIVATE KEY-----
+      server_cert: |
+        -----BEGIN CERTIFICATE-----
        REPLACE-consul-server-cert
+        -----END CERTIFICATE-----
+      server_key: |
+        -----BEGIN RSA PRIVATE KEY-----
        REPLACE-consul-server-key
+        -----END RSA PRIVATE KEY-----
+      agent:
+        domain: cf.internal
+        servers:
+          lan:
+          - 10.0.16.15
+        services:
+          netman-db:
+            check:
+              interval: 5s
+              script: "/bin/true"
+            name: netman-db
+    etcd:
+      machines:
+      - netman-db.service.cf.internal
+      peer_require_ssl: false
+      require_ssl: false
+  templates:
+  - name: consul_agent
+    release: cf
+  - name: etcd
+    release: etcd
+  env:
+    bosh:
+      password: "REPLACE- the right password for env bosh??"
+  update:
+    serial: true
+    max_in_flight: 1
+  networks:
+  - name: REPLACE-your-network-name
+    default:
+    - dns
+    - gateway
+  persistent_disk_type: '1024'
```


in uaa instance_group properties.clients:
```diff
         cf:
           id: cf
           override: true
           authorities: uaa.none
           authorized-grant-types: password,refresh_token
-          scope: cloud_controller.read,cloud_controller.write,openid,password.write,cloud_controller.admin,scim.read,scim.write,doppler.firehose,uaa.user,routing.router_groups.read
+          scope: cloud_controller.read,cloud_controller.write,openid,password.write,cloud_controller.admin,scim.read,scim.write,doppler.firehose,uaa.user,routing.router_groups.read,network.admin
           access-token-validity: 7200
           refresh-token-validity: 1209600
+        network-policy:
+          authorities: uaa.resource
+          secret: REPLACE-for-network-policy-client
```

in uaa instance_group properties.scim:
```diff
        users:
        - name: admin
          groups:
+        - network.admin
```

replace all instances of `garden-linux` with `garden-runc`

on every diego cell add the following jobs and update the stemcell:
```diff
+  - name: garden-cni
+    release: netman
+    consumes: {}
+    provides: {}
+  - name: cni-flannel
+    release: netman
+    consumes: {}
+    provides: {}
+  - name: vxlan-policy-agent
+    release: netman
+    consumes: {}
+    provides: {}
+  - name: netmon
+    release: netman
+    consumes: {}
+    provides: {}
   vm_type: m3.2xlarge
-  stemcell: bosh-aws-xen-hvm-ubuntu-trusty-go_agent
+  stemcell: bosh-aws-xen-ubuntu-trusty-go_agent
   properties:
     diego:
       executor:
```
