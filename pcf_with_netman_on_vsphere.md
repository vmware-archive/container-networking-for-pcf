## 1-Click PCF 1.8 Deploy on vSphere

1. Navigate to the [Toolsmiths Environments App](http://environments.toolsmiths.cf-app.com/engineering_environments).
2. Find a Resource Pool where the Status is "Available".
3. Click the rocket image.
4. Choose the `1.8-stable` PCF version.
5. Choose `cf-container-networking@pivotal.io` from the team email list.
6. Choose how long you want the environment for (ex: a week).
7. Click Deploy.
8. Click the icon to download the environment manifest.
9. Navigate to the Ops Manager interface. You can find the fully qualified domain domain
(FQDN) in the manifest under `ops_manager.url`.
10. In the Installation Dashboard should be the Ops Manager Director for vSphere tile.
On the right side, click `INSTALL`. (This will take about a half hour...)
11. Magically, the Pivotal Elastic Runtime tile will show up in the Installation Dashboard.
`INSTALL` this too. (This will take about an hour...)

## Using the BOSH CLI to upload releases
1. In the downloaded environment manifest, use the `vm_password` field to ssh onto the
Ops Manager VM with: `ssh ubunut@pcf.ENVIRONMENT.cf-app.com`.
- Note: if you will require your SSH key on the VM, use the `-A` flag to forward your SSH agent.
2. From the Ops Manager interface, navigate to the Status tab and copy the Director IP address.
3. From the Ops Manager interface, navigate to the Credentials tab, click `Link to Credential` for th `Director Credentials` and copy the json.
4. Run `uaac target --ca-cert /var/tempest/worksapces/default/root_ca_certificate https://DIRECTOR_IP:8443`.
5. Run `bosh target DIRECTOR_IP`
6. Log into the BOSH Director with `bosh --ca-cert /var/tempest/workspaces/default/root_ca_certificate target DIRECTOR_IP ENVIRONMENT_NAME`
 - You will be prompted to enter the Director Credentials. (Email can be filled with the `identity` value.)
7. Run `bosh upload release https://bosh.io/d/github.com/cloudfoundry-incubator/netman-release`.


## Deployment manifest edits for PCF 1.8 on vSphere

```diff
+properties:
+  policy-server:
+    ca_cert: POLICY_SERVER_CA_CERT
+    server_cert: POLICY_SERVER_CERT
+    server_key: POLICY_SERVER__KEY
+    database:
+      type: postgres
+      connection_string: postgres://policy_server:DATABASE_PASSWORD@10.10.2.42:5524/policy_server?sslmode=disable
+    skip_ssl_validation: true
+    uaa_client_secret: POLICY_SERVER_UAA_CLIENT_SECRET
+    uaa_url: https://uaa.system.ENVIRONMENT_NAME.cf-app.com
+  system_domain: system.ENVIRONMENT_NAME.cf-app.com
+  uaa:
+    clients:
+      network-policy:
+        secret: NETWORK_POLICY_SECRET
+  cni-flannel:
+    etcd_endpoints:
+    - http://cf-etcd.service.cf.internal:4001
+    etcd_client_cert: ETCD_CLIENT_CERT
+    etcd_client_key: ETCD_CLIENT_KEY
+    etcd_ca_cert: ETCD_CA_CERT
+    flannel:
+      etcd:
+        require_ssl: ETCD_REQUIRE_SSL
+  vxlan-policy-agent:
+    policy_server_url: http://policy-server.service.cf.internal:4002
+    ca_cert: VXLAN_POLICY_AGENT_CA_CERT
+    client_cert: VXLAN_POLICY_AGENT_CLIENT_CERT
+    client_key: VXLAN_POLICY_AGENT_CLIENT_KEY
+  garden_properties:
+    network_plugin: "/var/vcap/packages/runc-cni/bin/garden-external-networker"
+    network_plugin_extra_args:
+    - "--configFile=/var/vcap/jobs/garden-cni/config/adapter.json"
+  garden-cni:
+    cni_config_dir: "/var/vcap/jobs/cni-flannel/config/cni"
+    cni_plugin_dir: "/var/vcap/packages/flannel/bin"
```


```diff
 releases:
+- name: netman
+  version: latest
```

```diff
 instance_groups:
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
+        servers:
+          lan:
+          - CONSUL_AGENT_SERVERS_LAN
+        services:
+          policy-server:
+            check:
+              interval: 5s
+              script: "/bin/true"
+            name: policy-server
+    nats:
+      user: NATS_USERNAME
+      password: NATS_PASSWORD
+      port: 4222
+      machines:
+      - NATS_MACHINES
+    route_registrar:
+      routes:
+      - name: policy-server
+        port: 4002
+        registration_interval: 20s
+        uris:
+        - api.ENVIRONMENT_NAME.cf-app.com/networking
+  templates:
+  - name: policy-server
+    release: netman
+  - name: route_registrar
+    release: cf
+  - name: consul_agent
+    release: cf
+  env:
+    bosh:
+      password: "ENV_BOSH_PASSWORD"
+  update:
+    serial: true
+    max_in_flight: 1
+  networks:
+  - name: default
+    default:
+    - dns
+    - gateway
+  persistent_disk_type: '1024'
+- instances: 1
+  name: flannel_etcd
+  azs:
+  - default
+  vm_type: medium.mem
+  stemcell: bosh-vsphere-esxi-ubuntu-trusty-go_agent
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
+        servers:
+          lan:
+          - CONSUL_AGENT_SERVERS_LAN
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
+      password: "ENV_BOSH_PASSWORD"
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


in uaa instance_group properties.clients:
```diff
         cf:
-          scope: cloud_controller.read,cloud_controller.write,openid,password.write,cloud_controller.admin,scim.read,scim.write,doppler.firehose,uaa.user,routing.router_groups.read
+          scope: cloud_controller.read,cloud_controller.write,openid,password.write,cloud_controller.admin,scim.read,scim.write,doppler.firehose,uaa.user,routing.router_groups.read,network.admin
+        network-policy:
+          authorities: uaa.resource
+          secret: NETWORK_POLICY_SECRET
```

in uaa instance_group properties.scim:
```diff
        users:
        - name: admin
          groups:
+         - network.admin
```

on every diego cell add the following jobs and update the stemcell:
```diff
+  - name: garden-cni
+    release: netman
+  - name: cni-flannel
+    release: netman
+  - name: vxlan-policy-agent
+    release: netman
+  - name: netmon
+    release: netman
```
