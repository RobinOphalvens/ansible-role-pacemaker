# Pacemaker role for Ansible

## About

This role is a fork from [devgateway's original pacemaker role](https://github.com/devgateway/ansible-role-pacemaker). The main goal is to make the role a bit more easier to handle by introducing a couple of quality-life-changes. The original role relied on the user to manually include tasks and provide them with the required variables. Now these tasks are automatically called if the required variable is defined.

## Introduction

This role configures Pacemaker cluster by dumping the configuration (CIB), adjusting the XML, and
reloading it. The role is idempotent, and supports check mode.

It has been redesigned to configure individual elements (cluster defaults, resources, groups,
constraints, etc) rather than the whole state of the cluster and all the services. This allows you
to focus on specific resources, without interfering with the rest.

## Requirements

No specific role dependencies. Although using hostnames or FQDNs with `pcmk_cluster_nodes` requires DNS to be configured which this role does not handle.  

This role has been tested for Centos 7. Binary compatible clones such as RHEL and Scientific Linux should therefore also work.

## Role variables

| Variable                                  | Default                        | Comment                                                                                                |
|:------------------------------------------|:-------------------------------|:-------------------------------------------------------------------------------------------------------|
| `pcmk_cluster_name`                       | `hacluster`                    | The name of the cluster                                                                                |
| `pcmk_password`                           | `ansible_machine_id \| to_uuid`| The password used to authenticate with the cluster nodes. See note below regarding the default         |
| `pcmk_user`                               | `hacluster`                    | The system user to authenticate PCS nodes with.                                                        |
| `pcmk_cluster_nodes`                      |                                | **[REQUIRED]** A list of either the hostnames or the IP's of the cluster nodes                         |
| `pcmk_cluster_options`                    |                                | Dictionary with [cluster-wide options](https://clusterlabs.org/pacemaker/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/s-cluster-options.html) (optional). |
| `pcmk_votequorum`                         |                                | Dictionary with votequorum options (optional). See `votequorum(5)`. Boolean values accepted.           |
| `pcmk_resource_defaults`                  |                                | Dictionary of resource defaults (optional).
| `pcmk_resources`                          |                                | A dictionary with the resources that cluster should manage.                                            |
| `pcmk_advanced_resources`                 |                                | A dictionary with advanced resource types that the cluster should manage.                              |
| `pcmk_constraints`                        |                                | A dictionary with pacemaker constraints

## `pcmk_password`

The plaintext password for the cluster user (optional). If omitted, will be derived from ansible_machine_id of the first host in the play batch. This password is only used in the initial authentication of the nodes.

## A note regarding `pcmk_cluster_nodes`

The upstream role uses the `ansible_play_batch` variable to populate the `pcs cluster auth` and `pcs cluster setup` commands. This however has the potential to break most setups that use the Ansible provisioner within Vagrant. While manually specifying the nodes is less dynamic, it provides a more fool-proof way that the command will succeed.

## Pacemaker resources

Dictionary describing a simple (*primitive*) resource. Contains the following members:

* `id`: resource identifier; mandatory for simple resources;
* `class`, `provider`, and `type`: resource agent descriptors; `provider` may be omitted, e.g. when
  `type` is `service`;
* `options`: optional dictionary of resource-specific attributes, e.g. address and netmask for
  *IPaddr2*;
* `op`: optional list of operations; each operation is a dictionary with required `name` and
  `interval` members, and optional arbitrary members;
* `meta`: optional dictionary of meta-attributes.

Example:

```yaml
pcmk_resources:
  - id: virtual-ip
    class: ocf
    provider: heartbeat
    type: IPaddr2
    options:
      ip: 192.168.100.100
      cidr: 24
    meta:
      migration-threshold: 0
    op:
      - name: monitor
        timeout: 60s
        interval: 10s
        on-fail: restart
  - id: haproxy
    class: service
    type: haproxy
```

## Resource groups

As its name suggests, resources can be grouped into a collective resource.  

Dictionary with two members:

* `id` is the group identifier
* `resources` is a dictionary where keys are resource IDs, and values have the same format as
  `pcmk_resources` (except for `id` of the resources being optional).

## Constraints

Configure a constraint.  

Dictionary defining a single constraint. The following members are required:

* `type`: one of: `location`, `colocation`, or `order`;
* `score`: constraint score (signed integer, `INFINITY`, or `-INFINITY`).

Depending on the value of `type`, the following members are also required:

* `location` requires `rsc` and `node`;
* `colocation` requires `rsc` and `with-rsc`;
* `order` requires `first` and `then`;

The dictionary may contain other members, e.g. `symmetrical`.

Example:

``` yaml
pcmk_constraints:
  - type: colocation
    rsc: virtual-ip
    with-rsc: haproxy
    score: INFINITY
```

## Example playbook

### Two Haproxy loadbalancers in a cluster

``` yaml
    - name: Configure HAPROXY cluster
      hosts: lb01,lb02
      become: true
      roles:
        - ansible-role-pacemaker
      vars:
        pcmk_password: secret
        pcmk_cluster_nodes:
          - 192.168.100.7
          - 192.168.100.8
        pcmk_cluster_name: loadbalancer
        pcmk_cluster_options:
          stonith-enabled: false
        pcmk_resources:
          - id: virtual-ip
            class: ocf
            provider: heartbeat
            type: IPaddr2
            options:
              ip: 192.168.100.100
              cidr: 24
            meta:
              migration-threshold: 0
            op:
              - name: monitor
                timeout: 60s
                interval: 10s
                on-fail: restart
          - id: haproxy
            class: service
            type: haproxy
        pcmk_constraints:
          - type: colocation
            rsc: virtual-ip
            with-rsc: haproxy
            score: INFINITY
```

Other playbook examples, inspired by [Development Gateway](https://github.com/devgateway) can be found [here](examples.md)


## See also

- [The official Pacemaker documentation](http://clusterlabs.org/doc/)

## License

Licensed under GPL v3+.
