---
tags:
  - openstack
  - heat
  - orchestration
  - iac
---

# Heat — Orchestration Service

Heat là **Infrastructure as Code** cho OpenStack — tương đương AWS CloudFormation. Dùng template để tạo và quản lý infrastructure.

## Kiến trúc

```
heat-api (REST) ──► heat-engine ──► OpenStack Services
heat-api-cfn (CloudFormation compat) │   (Nova, Neutron, Cinder...)
                                     │
                                  heat-api-cloudwatch (deprecated)
```

## Template Format: HOT (Heat Orchestration Template)

```yaml
heat_template_version: 2021-04-16  # hoặc 2018-08-31

description: Simple web server stack

parameters:
  image_name:
    type: string
    default: ubuntu-22.04
    description: Image to use for server
  flavor:
    type: string
    default: m1.medium
  key_name:
    type: string
    description: SSH key pair name
  network_name:
    type: string
    default: internal-net

resources:

  # Security Group
  web_sg:
    type: OS::Neutron::SecurityGroup
    properties:
      rules:
        - protocol: tcp
          port_range_min: 22
          port_range_max: 22
        - protocol: tcp
          port_range_min: 80
          port_range_max: 80
        - protocol: icmp

  # Server Instance
  web_server:
    type: OS::Nova::Server
    properties:
      name: web-01
      image: { get_param: image_name }
      flavor: { get_param: flavor }
      key_name: { get_param: key_name }
      networks:
        - network: { get_param: network_name }
      security_groups:
        - { get_resource: web_sg }
      user_data_format: RAW
      user_data: |
        #!/bin/bash
        apt-get update
        apt-get install -y nginx
        systemctl start nginx

  # Floating IP
  floating_ip:
    type: OS::Neutron::FloatingIP
    properties:
      floating_network: provider-net

  floating_ip_association:
    type: OS::Neutron::FloatingIPAssociation
    properties:
      floatingip_id: { get_resource: floating_ip }
      port_id: { get_attr: [web_server, addresses, { get_param: network_name }, 0, port] }

outputs:
  server_ip:
    description: Floating IP of web server
    value: { get_attr: [floating_ip, floating_ip_address] }
  server_private_ip:
    description: Private IP
    value: { get_attr: [web_server, first_address] }
```

## CLI Usage

```bash
# Validate template
openstack orchestration template validate -t stack.yaml

# Tạo stack
openstack stack create -t stack.yaml \
  --parameter key_name=my-key \
  --parameter flavor=m1.large \
  my-stack

# Xem trạng thái
openstack stack show my-stack
openstack stack list

# Xem events (debug khi stack fail)
openstack stack event list my-stack

# Xem resources trong stack
openstack stack resource list my-stack

# Update stack
openstack stack update -t stack-updated.yaml my-stack

# Delete stack (xóa TẤT CẢ resources)
openstack stack delete my-stack
```

## Resource Types phổ biến

| Resource Type | Mô tả |
|--------------|-------|
| `OS::Nova::Server` | VM instance |
| `OS::Neutron::Net` | Network |
| `OS::Neutron::Subnet` | Subnet |
| `OS::Neutron::Router` | Router |
| `OS::Neutron::FloatingIP` | Floating IP |
| `OS::Neutron::SecurityGroup` | Security Group |
| `OS::Cinder::Volume` | Block volume |
| `OS::Heat::AutoScalingGroup` | Auto scaling group |
| `OS::Heat::ScalingPolicy` | Scaling policy |
| `OS::Octavia::LoadBalancer` | Load balancer |

## Auto Scaling

```yaml
resources:
  asg:
    type: OS::Heat::AutoScalingGroup
    properties:
      min_size: 2
      max_size: 10
      resource:
        type: OS::Nova::Server
        properties:
          image: ubuntu-22.04
          flavor: m1.medium

  scale_up_policy:
    type: OS::Heat::ScalingPolicy
    properties:
      adjustment_type: change_in_capacity
      auto_scaling_group_id: { get_resource: asg }
      scaling_adjustment: 1

  scale_down_policy:
    type: OS::Heat::ScalingPolicy
    properties:
      adjustment_type: change_in_capacity
      auto_scaling_group_id: { get_resource: asg }
      scaling_adjustment: -1
```

## Stack Environments

```bash
# environment.yaml — override defaults, resource registry
openstack stack create -t stack.yaml \
  -e environment.yaml \
  my-stack
```

```yaml
# environment.yaml
parameters:
  flavor: m1.xlarge
  image_name: ubuntu-22.04-custom

resource_registry:
  OS::MyApp::Server: templates/server.yaml  # custom resource type
```

---
*Xem thêm: [[Nova - Compute]] | [[Neutron - Networking]] | [[Cinder - Block Storage]] | [[OpenStack]]*
