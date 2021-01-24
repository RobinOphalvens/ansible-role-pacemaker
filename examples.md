# Playbook examples

## Active-active chrooted BIND DNS server

``` yaml
    ---
    - name: Configure DNS cluster
      hosts: dns-servers
      vars:
        pcmk_password: hunter2
        pcmk_cluster_name: named
        pcmk_cluster_options:
          stonith-enabled: false
        pcmk_resources:
          - id: dns-ip
            class: ocf
            provider: heartbeat
            type: IPaddr2
            options:
              ip: 10.0.0.1
              cidr_netmask: 8
            op:
              - name: monitor
                interval: 5s
              
          - id: dns-clone
            type: clone
            resources:
              named:
                class: service
                type: named-chroot
                op:
                  - name: monitor
                    interval: 5s
        pcmk_constraints:
          type: order
          first: dns-ip
          then: dns-clone
```

### Active-active Squid proxy

```yaml
    ---
    - name: Configure Squid cluster
      hosts: proxy-servers
      vars:
        pcmk_password: hunter2
        pcmk_cluster_name: squid
        pcmk_cluster_options:
          stonith-enabled: false
        pcmk_resources:
          - id: squid-ip
            class: ocf
            provider: heartbeat
            type: IPaddr2
            options:
              ip: 192.168.0.200
              cidr_netmask: 24
            op:
              - name: monitor
                interval: 5s
        pcmk_advanced_resource:
          id: squid
            type: clone
            resources:
              squid-service:
                class: service
                type: squid
                op:
                  - name: monitor
                    interval: 5s
        pcmk_constraint:
          type: order
          first: squid-ip
          then: squid
```

### Nginx, web application, and master-slave Postgres

The cluster runs two Postgres nodes with synchronous replication. Wherever master is, a virtual IP
address is running, where NAT is pointing at. Nginx and the webapp are running at the same node, but
not the other, in order to save resources. Based on [the example from Clusterlabs
wiki](https://wiki.clusterlabs.org/wiki/PgSQL_Replicated_Cluster).

```yaml
    ---
    - hosts:
        - alpha
        - bravo
      vars:
        pcmk_pretty_xml: true
        pcmk_cluster_name: example
        pcmk_password: hunter2
        pcmk_cluster_options:
          no-quorum-policy: ignore
          stonith-enabled: false
        pcmk_resource_defaults:
          resource-stickiness: INFINITY
          migration-threshold: 1
        pcmk_resources:
          - id: coolapp
            class: service
            type: coolapp
          - id: nginx
            class: service
            type: nginx
          - id: virtual-ip
            class: ocf
            provider: heartbeat
            type: IPaddr2
            options:
              ip: 10.0.0.23
            meta:
              migration-threshold: 0
            op:
              - name: start
                timeout: 60s
                interval: 0s
                on-fail: restart
              - name: monitor
                timeout: 60s
                interval: 10s
                on-fail: restart
              - name: stop
                timeout: 60s
                interval: 0s
                on-fail: restart
        pcmk_advanced_resource:
          id: postgres
          type: master
          meta:
            master-max: 1
            master-node-max: 1
            clone-max: 2
            clone-node-max: 1
            notify: true
          resources:
            postgres-replica-set:
              class: ocf
              provider: heartbeat
              type: pgsql
              options:
                pgctl: /usr/pgsql-9.4/bin/pg_ctl
                psql: /usr/pgsql-9.4/bin/psql
                pgdata: /var/lib/pgsql/9.4/data
                rep_mode: sync
                node_list: "{{ ansible_play_batch | join(' ') }}"
                restore_command: cp /var/lib/pgsql/9.4/archive/%f %p
                master_ip: 10.0.0.23
                restart_on_promote: "true"
                repuser: replication
              op:
                - name: start
                  timeout: 60s
                  interval: 0s
                  on-fail: restart
                - name: monitor
                  timeout: 60s
                  interval: 4s
                  on-fail: restart
                - name: monitor
                  timeout: 60s
                  interval: 3s
                  on-fail: restart
                  role: Master
                - name: promote
                  timeout: 60s
                  interval: 0s
                  on-fail: restart
                - name: demote
                  timeout: 60s
                  interval: 0s
                  on-fail: stop
                - name: stop
                  timeout: 60s
                  interval: 0s
                  on-fail: block
                - name: notify
                  timeout: 60s
                  interval: 0s
        pcmk_constraints:
          - type: colocation
            rsc: virtual-ip
            with-rsc: postgres
            with-rsc-role: Master
            score: INFINITY
          - type: colocation
            rsc: nginx
            with-rsc: virtual-ip
            score: INFINITY
          - type: colocation
            rsc: coolapp
            with-rsc: virtual-ip
            score: INFINITY
          - type: order
            first: postgres
            first-action: promote
            then: virtual-ip
            then-action: start
            symmetrical: false
            score: INFINITY
          - type: order
            first: postgres
            first-action: demote
            then: virtual-ip
            then-action: stop
            symmetrical: false
            score: 0
```