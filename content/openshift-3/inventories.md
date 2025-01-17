# Example inventories
## simple-2-node-cluster.inventory

```ini
# Create an OSEv3 group that contains the masters and nodes groups
[OSEv3:children]
masters
nodes
etcd

# Set variables common for all OSEv3 hosts
[OSEv3:vars]
ansible_ssh_user=ec2-user
ansible_become=yes

# https://github.com/openshift/openshift-ansible/blob/master/DEPLOYMENT_TYPES.md
deployment_type=openshift-enterprise
oreg_url=registry.redhat.io/openshift3/ose-${component}:${version}
oreg_auth_user=XXXXXXXXXXXX
oreg_auth_password=XXXXXXXXXXXX

containerized=false

# Skip env validation
openshift_disable_check=disk_availability,memory_availability

# Configure usage of openshift_clock role.
openshift_clock_enabled=true

# Set upgrade restart mode for full system restarts
openshift_rolling_restart_mode=system

# Enable cockpit
osm_use_cockpit=false
osm_cockpit_plugins=['cockpit-kubernetes', 'cockpit-pcp', 'setroubleshoot-server']

# Docker / Registry Configuration
openshift_docker_disable_push_dockerhub=True
openshift_docker_options="--log-driver=journald --log-level=warn --ipv6=false"
openshift_docker_insecure_registries=docker-registry.default.svc,docker-registry.default.svc.cluster.local

# Native high availability cluster method with optional load balancer.

openshift_master_cluster_method=native
openshift_master_cluster_hostname=ec2-xx-xx-13-10.eu-central-1.compute.amazonaws.com
openshift_master_cluster_public_hostname=ec2-xx-xx-13-10.eu-central-1.compute.amazonaws.com
openshift_master_api_port=8443
openshift_master_console_port=8443


# Configure nodeIP in the node config
# This is needed in cases where node traffic is desired to go over an
# interface other than the default network interface.

# Configure the multi-tenant SDN plugin (default is 'redhat/openshift-ovs-subnet')
os_sdn_network_plugin_name=redhat/openshift-ovs-multitenant

# Configure SDN cluster network and kubernetes service CIDR blocks. These
# network blocks should be private and should not conflict with network blocks
# in your infrastructure that pods may require access to. Can not be changed
# after deployment.
osm_cluster_network_cidr=10.1.0.0/16
openshift_portal_net=172.30.0.0/16
osm_host_subnet_length=8


# htpasswd auth
openshift_master_identity_providers=[{'name': 'htpasswd_auth', 'login': 'true', 'challenge': 'true', 'kind': 'HTPasswdPasswordIdentityProvider'}]

# Provide local certificate paths which will be deployed to masters
openshift_master_overwrite_named_certificates=true

# Install the openshift examples
openshift_install_examples=true
openshift_examples_modify_imagestreams=true

# default subdomain to use for exposed routes
openshift_master_default_subdomain=apps.xx.xx.13.10.nip.io

# Openshift Registry Options
openshift_hosted_registry_storage_kind=hostpath
openshift_hosted_registry_replicas=1
openshift_hosted_registry_storage_hostpath_path=/var/openshift-registry-local-storage
#Fix SELinux: chcon -R unconfined_u:object_r:svirt_sandbox_file_t:s0 /var/openshift-registry-local-storage/
#OCS

# Metrics deployment
openshift_metrics_install_metrics=false

# Logging deployment
openshift_logging_install_logging=false

# Prometheus
openshift_cluster_monitoring_operator_install=true

# Operator Lifecycle Manager
openshift_enable_olm=true
# openshift_additional_registry_credentials=[{'host':'registry.connect.redhat.com','user':'your_user','password':'your_pwd','test_image':'mongodb/enterprise-operator:0.3.2'}]




[masters]
ec2-xx-xx-13-10.eu-central-1.compute.amazonaws.com openshift_node_group_name="node-config-master-infra"

[etcd]
ec2-xx-xx-13-10.eu-central-1.compute.amazonaws.com openshift_node_group_name="node-config-master-infra"


[nodes]
ec2-xx-xx-13-10.eu-central-1.compute.amazonaws.com openshift_node_group_name="node-config-master-infra"
ec2-3-122-43-141.eu-central-1.compute.amazonaws.com openshift_node_group_name='node-config-compute'

[bastion]
ec2-xx-xx-13-10.eu-central-1.compute.amazonaws.com openshift_node_group_name="node-config-master-infra"

```