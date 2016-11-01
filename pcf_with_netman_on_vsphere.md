## Prerequisites
You have an OpsManager and Elastic Runtime Tile 1.8 deployed.  You know how
to SSH to the OpsMan VM and run BOSH commands against the BOSH director.

## Deployment manifest edits for PCF 1.8 on vSphere


### Releases
```diff
+- name: netman
+  version: latest
```

### Instance Groups
In `mysql.properties.cf_mysql.mysql.seeded_databases:` add a new database:
```diff
+        - name: policy_server
+          username: policy_server
+          password: DATABASE_PASSWORD
```

In `uaa.properties.uaa.clients.cf`, modify the list of `scopes` to include `network.admin`:
```diff
-          scope: cloud_controller.read,cloud_controller.write,openid,password.write,cloud_controller.admin,scim.read,scim.write,doppler.firehose,uaa.user,routing.router_groups.read,routing.router_groups.write
+          scope: cloud_controller.read,cloud_controller.write,openid,password.write,cloud_controller.admin,scim.read,scim.write,doppler.firehose,uaa.user,routing.router_groups.read,routing.router_groups.write,network.admin
```

In `uaa.properties.uaa.clients`, add a new client called `network-policy`:
```diff
+        network-policy:
+          authorities: uaa.resource
+          secret: NETWORK_POLICY_SERVER_UAA_CLIENT_SECRET
```


In `uaa.properties.uaa.scim.users`, modify the `admin` user to belong to a new group called `network.admin`:
```diff
           - console.admin
           - console.support
           - doppler.firehose
+          - network.admin
           - notification_preferences.read
           - notification_preferences.write
           - notifications.manage
```

Add a new instance group called `policy-server` which includes the `consul` job with the standard `consul` encryption key and certificates.

```diff
+- instances: 1
+  name: policy-server
+  vm_type: medium.mem
+  lifecycle: service
+  stemcell: bosh-vsphere-esxi-ubuntu-trusty-go_agent
+  azs:
+  - default
+  properties:
+    consul:
+      encrypt_keys:
+      - CONSUL_ENCRYPT_KEY
+      ca_cert: CONSUL_CA_CERT
+      agent_cert: CONSUL_AGENT_CERT
+      agent_key: CONSUL_AGENT_KEY
+      server_cert: CONSUL_SERVER_CERT
+      server_key: CONSUL_SERVER_KEY
+      agent:
+        domain: cf.internal
+        services:
+          policy-server:
+            check:
+              interval: 5s
+              script: "/bin/true"
+            name: policy-server
+        servers:
+          lan:
+          - CONSUL_LAN_IP
+    nats:
+      user: NATS_USERNAME
+      password: NATS_PASSWORD
+      port: 4222
+      machines:
+      - NATS_IP
+    route_registrar:
+      routes:
+      - name: policy-server
+        port: 4002
+        registration_interval: 20s
+        uris:
+        - api.SYSTEM_DOMAIN/networking
+    policy-server:
+      ca_cert: POLICY_SERVER_CA_CERT
+      server_cert: POLICY_SERVER_SERVER_CERT
+      server_key: POLICY_SERVER_SERVER_KEY
+      database:
+        connection_string: policy_server:DATABASE_PASSWORD@tcp(MYSQL_PROXY_IP:3306)/policy_server
+        type: mysql
+      skip_ssl_validation: true
+      uaa_client_secret: NETWORK_POLICY_SERVER_UAA_CLIENT_SECRET
+      uaa_url: https://uaa.SYSTEM_DOMAIN
+  templates:
+  - name: policy-server
+    release: netman
+  - name: route_registrar
+    release: cf
+  - name: consul_agent
+    release: cf
+  env:
+    bosh:
+      password: ENV_BOSH_PASSWORD
+  update:
+    serial: true
+    max_in_flight: 1
+  networks:
+  - name: default
+    default:
+    - dns
+    - gateway
+  persistent_disk_type: '1024'
```

In the `diego_cell` instance group...

1. Add these jobs:

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
  ```

2. Edit `properties.garden` to include:

  ```diff
  +      network_plugin: "/var/vcap/packages/runc-cni/bin/garden-external-networker"
  +      network_plugin_extra_args:
  +      - "--configFile=/var/vcap/jobs/garden-cni/config/adapter.json"
  ```

3. Add to `properties`
  ```diff
  +    cni-flannel:
  +      etcd_ca_cert:
  +      etcd_client_cert:
  +      etcd_client_key:
  +      etcd_endpoints:
  +      - ETCD_IP
  +      flannel:
  +        etcd:
  +          require_ssl: false
  +    vxlan-policy-agent:
  +      policy_server_url: https://policy-server.service.cf.internal:4003
  +      ca_cert: VXLAN_POLICY_AGENT_CA_CERT
  +      client_cert: VXLAN_POLICY_AGENT_CLIENT_CERT
  +      client_key: VXLAN_POLICY_AGENT_CLIENT_KEY
  +    garden-cni:
  +      cni_config_dir: "/var/vcap/jobs/cni-flannel/config/cni"
  +      cni_plugin_dir: "/var/vcap/packages/flannel/bin"
  ```

  NOTE: The client certificate and key that the `vxlan-policy-agent` presents to the `policy-server` must be generated from the same `ca_cert` that is used to generate the server certificate and key for the `policy-server`. There is a [script in netman-release](https://github.com/cloudfoundry-incubator/netman-release/blob/develop/scripts/generate-certs) to generate certs/keys for this.

