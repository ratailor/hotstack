<!--
// Assisted by watsonx Code Assistant
// Code generated by WCA@IBM in this programming language is not approved for
// use in IBM product development.
-->

# Heat Templates in HotStack Scenarios

Heat templates are used in HotStack scenarios to define and manage the
infrastructure as code. They are essentially YAML files that describe the
resources required to deploy and configure the OpenStack resources. This
document provides an overview of Heat templates used in HotStack scenarios for
deploying and configuring OpenStack resources, specifically for OpenShift
Container Platform (OCP) and Red Hat OpenStack Platform (RHOSO).

- [Heat Template Parameters in HotStack Scenarios](#heat-template-parameters-in-hotstack-scenarios)
- [Heat Template Resources in HotStack Scenarios](#heat-template-resources-in-hotstack-scenarios)
  - [Network Resources (Networks, Subnets and Routers)](#network-resources-networks-subnets-and-routers)
  - [Heat Template Resources for Controller Node in HotStack Scenarios](#heat-template-resources-for-controller-node-in-hotstack-scenarios)
  - [Heat Template Resources for OCP Master Nodes in HotStack Scenarios](#heat-template-resources-for-ocp-master-nodes-in-hotstack-scenarios)
  - [Heat Template Resources for pre-provisioned Dataplane Nodes](#heat-template-resources-for-pre-provisioned-dataplane-nodes)
  - [Heat Template Resources for Dataplane Baremetal Hosts](#heat-template-resources-for-dataplane-baremetal-hosts)
  - [Heat Template Resources for Ironic Nodes in HotStack Scenarios](#heat-template-resources-for-ironic-nodes-in-hotstack-scenarios)
- [Heat Template Output Values in HotStack Scenarios](#heat-template-output-values-in-hotstack-scenarios)
  - [Output Values](#output-values)

## Heat Template Parameters in HotStack Scenarios

This section explains the typical parameters defined in a Heat Orchestration
Template (HOT) used in a HotStack scenario. These parameters allow users to
customize the deployment to their specific requirements. Some scenarios will
have additional parameters for other node types, similar to the
`compute_params` below, other scenarios may have the same for other node types,
for example: `networker_params`, `ironic_params`.

- **dns_servers**
  - **Type**: comma_delimited_list
  - **Default**: ["8.8.8.8", "8.8.4.4"]
  - **Description**: A comma-delimited list of DNS servers.

- **ntp_servers**
  - **Type**: comma_delimited_list
  - **Default**: []
  - **Description**: A comma-delimited list of NTP servers.

- **controller_ssh_pub_key**
  - **Type**: string
  - **Description**: The public SSH key for the Controller node.

- **dataplane_ssh_pub_key**
  - **Type**: string
  - **Description**: The public SSH key for the dataplane nodes.

- **controller_params**
  - **Type**: json
  - **Default**:

    ```json
    {
      "image": "hotstack-controller",
      "flavor": "hotstack.small"
    }
    ```

  - **Description**: Parameters specific to the Controller node.

- **ocp_master_params**
  - **Type**: json
  - **Default**:

    ```json
    {
      "image": "ipxe-boot-usb",
      "flavor": "hotstack.xxlarge"
    }
    ```

  - **Description**: Parameters specific to the OCP master nodes.

- **compute_params**
  - **Type**: json
  - **Default**:

    ```json
    {
      "image": "CentOS-Stream-GenericCloud-9",
      "flavor": "hotstack.large"
    }
    ```

  - **Description**: Parameters specific to the Compute nodes.

- **router_external_network**
  - **Type**: string
  - **Default**: "public"
  - **Description**: The name of the external network for the router.

- **floating_ip_network**
  - **Type**: string
  - **Default**: "public"
  - **Description**: The name of the network for floating IPs.

- **net_value_specs**
  - **Type**: json
  - **Default**: {}
  - **Description**: Network value specifications.

## Heat Template Resources in HotStack Scenarios

This section explains the Openstack networking related resources defined in a
Heat Orchestration Template (HOT) used in a HotStack scenario. Different
scenarios can have additional resources depending on the specific needs, the
resources below will be in most scenarios.

### Network Resources (Networks, Subnets and Routers)

- **Networks**
  - **machine-net**: A Neutron network with port security disabled, using the
    `net_value_specs` parameter for additional configuration.
  - **ctlplane-net**: A Neutron network with port security disabled, using the
    `net_value_specs` parameter for additional configuration.
  - **internal-api-net**: A Neutron network with port security disabled, using
    the `net_value_specs` parameter for additional configuration.
  - **storage-net**: A Neutron network with port security disabled, using the
    `net_value_specs` parameter for additional configuration.
  - **tenant-net**: A Neutron network with port security disabled, using the
    `net_value_specs` parameter for additional configuration.

- **Subnets**
  - **machine-subnet**: A Neutron subnet connected to the `machine-net`
    network, with a CIDR of 192.168.32.0/24, enabling DHCP, and specifying
    DNS nameservers.
  - **ctlplane-subnet**: A Neutron subnet connected to the `ctlplane-net`
    network, with a CIDR of 192.168.122.0/24, disabling DHCP, and specifying
    an allocation pool and DNS nameservers.
  - **internal-api-subnet**: A Neutron subnet connected to the
    `internal-api-net` network, with a CIDR of 172.17.0.0/24, disabling DHCP,
    and specifying an allocation pool.
  - **storage-subnet**: A Neutron subnet connected to the `storage-net`
    network, with a CIDR of 172.18.0.0/24, disabling DHCP, and specifying an
    allocation pool.
  - **tenant-subnet**: A Neutron subnet connected to the `tenant-net` network,
    with a CIDR of 172.19.0.0/24, disabling DHCP, and specifying an allocation
    pool.

- **Routers**
  - **router**: A Neutron router with an external gateway connected to the
    `router_external_network`.
  - **machine-net-router-interface**: A Neutron router interface connecting
    the `router` to the `machine-subnet`.
  - **ctlplane-net-router-interface**: A Neutron router interface connecting
    the `router` to the `ctlplane-subnet`.

### Heat Template Resources for Controller Node in HotStack Scenarios

This section explains the resources defined in a Heat Orchestration Template
(HOT) used in a HotStack scenario to create a Neutron port, a floating IP, and
a Nova server for the Controller node.

- **controller-machine-port**
  - **Type**: OS::Neutron::Port
  - **Purpose**: Creates a Neutron port with a fixed IP address (192.168.32.3)
    for the Controller node.

- **controller-floating-ip**
  - **Type**: OS::Neutron::FloatingIP
  - **Purpose**: Associates a floating IP with the `controller-machine-port`
    resource, allowing external access to the Controller node.

- **controller**
  - **Type**: OS::Nova::Server
  - **Purpose**: Creates a Nova server for the Controller node using the
    specified image, flavor, and network interface. The user data for the Nova
    server includes the combined configuration of user management, file
    writing, and run commands.

#### Cloud configuration resources for the Controller node

This section explains the resources defined in a Heat Orchestration Template
(HOT) used in a HotStack scenario to configure the Controller node. These
resources include user management, file writing, run commands, and multipart
configuration.

These resources are typically identical in all scenarios, with the
`controller-write-files` and `controller-runcmd` being the exception. Depending
on the scenario the host records and wildcard records in dnsmasq will be
different. For any multi node OCP scenario haproxy is also configured, enabled
and started on the Controller node.

- **controller_users**
  - **Type**: OS::Heat::CloudConfig
  - **Purpose**: Creates a new user named "zuul" with specific settings, such
    as the SSH public key.

- **controller-write-files**
  - **Type**: OS::Heat::CloudConfig
  - **Purpose**: Configures the dnsmasq service, sets DNS forwarders, host
    records, wildcard records, and updates the resolv.conf, NetworkManager
    and HAProxy configuration files.

- **controller-runcmd**
  - **Type**: OS::Heat::CloudConfig
  - **Purpose**: Enables and starts the dnsmasq, HAProxy and httpd services,
    sets the SELinux enforcing mode to permissive, and modifies the Apache
    HTTP server configuration.

- **controller-init**
  - **Type**: OS::Heat::MultipartMime
  - **Purpose**: Combines the user management, file writing, and run commands
  into a single configuration that is applied during the Controller node's
  initialization.

### Heat Template Resources for OCP Master Nodes in HotStack Scenarios

This section explains the resources defined in a Heat Orchestration Template
(HOT) used in a HotStack scenario to configure the OCP master nodes. These
resources include DHCP options, Neutron ports, a trunk port, Cinder volumes,
and a Nova server. For multi node scenarios a single shared
`extra-dhcp-opts-value` resource can be used.

- **extra-dhcp-opts-value**
  - **Type**: OS::Heat::Value
  - **Purpose**: Defines DHCP options for the OCP master node, including an
    option to set the HTTPClient for DHCP discovery and an option to provide
    the location of the OCP Agent Installers iPXE boot script.

- **masterX-machine-port**
  - **Type**: OS::Neutron::Port
  - **Purpose**: Creates a Neutron port with a fixed IP address for the OCP
    master node, applying the DHCP options defined in the
    `extra-dhcp-opts-value` resource.

- **masterX-ctlplane-trunk-parent-port**
  - **Type**: OS::Neutron::Port
  - **Purpose**: Creates a Neutron port with a fixed IP address for the OCP
    master node's control plane interface.

- **masterX-internal-api-port**
  - **Type**: OS::Neutron::Port
  - **Purpose**: Creates a Neutron port with a fixed IP address for the OCP
    master node's internal API interface.

- **masterX-storage-port**
  - **Type**: OS::Neutron::Port
  - **Purpose**: Creates a Neutron port with a fixed IP address for the OCP
    master node's storage interface.

- **masterX-tenant-port**
  - **Type**: OS::Neutron::Port
  - **Purpose**: Creates a Neutron port with a fixed IP address for the OCP
    master node's tenant interface.

- **masterX-trunk0**
  - **Type**: OS::Neutron::Trunk
  - **Purpose**: Combines multiple network interfaces (internal API, storage
    and tenant) into a single trunk port, allowing the OCP master node to
    communicate with different networks using VLAN segmentation.

- **masterX-ironic-port**
  - **Type**: OS::Neutron::Port
  - **Purpose**: Creates a Neutron port for the OCP master node's Ironic
    interface.

- **masterX-lvms-vol0**, **masterX-cinder-vol0**, **masterX-cinder-vol1**,
  and **masterX-cinder-vol2**
  - **Type**: OS::Cinder::Volume
  - **Purpose**: Creates Cinder volumes to be attached to the OCP master node
    as block devices.

- **masterX**
  - **Type**: OS::Nova::Server
  - **Purpose**: Creates a Nova server for the OCP master node using the
    specified image, flavor, and network interfaces. The server attaches the
    Cinder volumes as block devices and connects to the respective networks.

### Heat Template Resources for pre-provisioned Dataplane Nodes

This section explains the resources defined in a Heat Orchestration Template
(HOT) used in a HotStack scenario to configure the Dataplane nodes. These
resources include user management, hostname configuration, Neutron ports, a
trunk port, and a Nova server. This example is for a compute node, but the same
pattern is used for Networker nodes in the scenarios that include that node
type. For scenarios featuring multiple Dataplane nodes a single shared
`dataplane_users` resource can be used.

- **dataplane_users**
  - **Type**: OS::Heat::CloudConfig
  - **Purpose**: Creates a new user named "cloud-admin" with specific
    settings, such as the SSH public key.

- **computeX_hostname**
  - **Type**: OS::Heat::CloudConfig
  - **Purpose**: Sets the hostname and fully qualified domain name for the
    Dataplane node.

- **computeX_init**
  - **Type**: OS::Heat::MultipartMime
  - **Purpose**: Combines the user management and hostname configuration into
    a single configuration.

- **computeX-ctlplane-trunk-parent-port**
  - **Type**: OS::Neutron::Port
  - **Purpose**: Creates a Neutron port with a fixed IP address for the
    Dataplane node's control plane interface.

- **computeX-internal-api-port**
  - **Type**: OS::Neutron::Port
  - **Purpose**: Creates a Neutron port with a fixed IP address for the
    Dataplane node's internal API interface.

- **computeX-storage-port**
  - **Type**: OS::Neutron::Port
  - **Purpose**: Creates a Neutron port with a fixed IP address for the
    Dataplane node's storage interface.

- **computeX-tenant-port**
  - **Type**: OS::Neutron::Port
  - **Purpose**: Creates a Neutron port with a fixed IP address for the
    Dataplane node's tenant interface.

- **computeX-trunk0**
  - **Type**: OS::Neutron::Trunk
  - **Purpose**: Combines multiple network interfaces (internal API, storage,
    and tenant) into a single trunk port, allowing the Dataplane node to
    communicate with different networks using VLAN segmentation.

- **computeX**
  - **Type**: OS::Nova::Server
  - **Purpose**: Creates a Nova server for the Dataplane node using the
    specified image, flavor, and network interfaces. The server uses a config
    drive for additional configuration data and connects to the respective
    networks through the trunk port.

### Heat Template Resources for Dataplane Baremetal Hosts

This section explains the resources defined in a Heat Orchestration Template
(HOT) used in a HotStack scenario to configure Dataplane Baremetal Hosts. These
resources include Neutron ports, a trunk port, and a Nova server for the
Baremetal Host.

- **bmhX-ctlplane-trunk-parent-port**, **bmhX-internal-api-port**,
  **bmhX-storage-port**, **bmhX-tenant-port**, and **bmhX-provisioning-port**
  - **Type**: OS::Neutron::Port
  - **Purpose**: Creates Neutron ports for the Baremetal Host, connecting them
    to the respective networks (e.g., `ctlplane-net-1`, `internal-api-net`,
    `storage-net`, `tenant-net`, or `provisioning-net-0`) with port security
    disabled.

- **bmhX-trunk0**
  - **Type**: OS::Neutron::Trunk
  - **Purpose**: Combines multiple network interfaces (internal API, storage,
    tenant, and provisioning) into a single trunk port, allowing the Baremetal
    Host to communicate with different networks using VLAN segmentation.

- **bmhX**
  - **Type**: OS::Nova::Server
  - **Purpose**: Creates a Nova server for the Baremetal Host using the
    specified flavor and block devices. The block devices include a root disk
    and a CD-ROM device, with the root disk using the image specified in the
    `image` property of the `bmh_params` dictionary and the CD-ROM device
    using the image specified in the `cd_image` property of the `bmh_params`
    dictionary. The server connects to the respective networks through the
    trunk port and the `bmhX-ctlplane-trunk-parent-port` Neutron port.

### Heat Template Resources for Ironic Nodes in HotStack Scenarios

This section explains the resources defined in a Heat Orchestration Template
(HOT) used in a HotStack scenario to configure the Ironic nodes. These
resources include Neutron ports and Nova servers for the Ironic nodes.

- **ironicX-port**
  - **Type**: OS::Neutron::Port
  - **Purpose**: Creates Neutron port for the Ironic node, connecting it to
    the `ironic-net` network with port security disabled.

- **ironicX**
  - **Type**: OS::Nova::Server
  - **Purpose**: Creates Nova server for the Ironic node using the specified
    flavor and block devices. The block devices include a root disk and a
    CD-ROM device, with the root disk using the image specified in the `image`
    property of the `ironic_params` dictionary and the CD-ROM device using the
    image specified in the `cd_image` property of the `ironic_params`
    dictionary. The server connect to the respective networks through the
    Neutron port.

## Heat Template Output Values in HotStack Scenarios

This section explains the output values generated by a Heat Orchestration
Template (HOT) in a HotStack scenario. The output values provide essential
information about the deployed resources, which are used by hotstack to
configure and manage the infrastructure deployed by heat on Openstack.

### Output Values

- **controller_floating_ip**
  - **Description**: The floating IP address of the Controller node.
  - **Value**: The actual floating IP address of the Controller node.

- **sushy_emulator_uuids**
  - **Description**: UUIDs of instances that can be managed with sushy-tools,
    which is a tool for interacting with RedFish-enabled virtual BMCs.
  - **Value**: The UUIDs of instances intended for virtual BMCs management.

- **ocp_install_config**
  - **Description**: The OCP install-config.yaml file, which is used to
    configure the OCP cluster during installation.
  - **Value**: Contains details such as the base domain, control plane and
  compute configurations, networking settings, and platform information.

- **ocp_agent_config**
  - **Description**: The OCP agent-config.yaml file, which is used to
    configure the OCP agent during installation.
  - **Value**: Contains details such as the rendezvous IP, additional NTP
    sources, boot artifacts base URL, and host configurations.

- **controller_ansible_host**
  - **Description**: The Ansible host information for the Controller node.
    This can be passed to the `ansible.builtin.add_host` module.
  - **Value**: Includes the hostname, SSH user, IP address, and SSH common
    arguments.

- **ansible_inventory**
  - **Description**: The Ansible inventory file for the OCP cluster.
  - **Value**: Defines the hosts and groups for the controller, OCPs, and
    other nodes, along with their respective Ansible connection, user, and
    private key file information.
