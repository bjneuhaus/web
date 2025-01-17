# Disconnected

## Disconnected installation notes

[Documentation](https://docs.openshift.com/container-platform/3.11/install/disconnected_install.html)

### Prepare OpenStack Env.

#### Default security without internet access

![](default-security-group.png)

#### Useful cloud-init config

```text
#cloud-config

# set the locale
locale: en_US.UTF-8

# timezone: set the timezone for this instance
timezone: UTC

hostname: instance
fqdn: instance
```

### Fetch data (images & rpms)

#### via Docker (on your MacBook)

##### Dockerfile 

```
FROM registry.redhat.io/rhel7
# docker build . --build-arg RH_ORG_ID=...
ARG RH_ORG_ID 
ARG RH_ACTIVATIONKEY
ARG RH_POOL_ID

RUN subscription-manager register --org=$RH_ORG_ID --activationkey=$RH_ACTIVATIONKEY  && \
    subscription-manager attach --pool=$RH_POOL_ID && \
    subscription-manager repos --disable="*" && \
    subscription-manager repos \
        --enable="rhel-7-server-rpms" \
        --enable="rhel-7-server-extras-rpms" \
        --enable="rhel-7-server-ose-3.11-rpms" \
        --enable="rhel-7-server-ansible-2.6-rpms" && \
    yum install -y skopeo openssl docker-distribution tmux lsof telnet yum-utils createrepo git && \
    subscription-manager unregister

VOLUME /work
WORKDIR /work

ENTRYPOINT bash

USER 0
```

##### Build the Image

```text
docker build -t ocp3-disconnected-fetcher --build-arg RH_ORG_ID=xxx --build-arg RH_ACTIVATIONKEY=xxx --build-arg RH_POOL_ID=xxx .
docker run -ti -v $(pwd):/work ocp3-disconnected-fetcher
./fetch-data.sh sync
```

![](fetch-data.png)

#### On an RHEL box

On an RHEL box with an openshift subscription and enabled repos

```text
rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
subscription-manager register

subscription-manager list --available --matches '*OpenShift*'
subscription-manager attach --pool=<pool_id>
subscription-manager repos --disable="*"

subscription-manager repos \
    --enable="rhel-7-server-rpms" \
    --enable="rhel-7-server-extras-rpms" \
    --enable="rhel-7-server-ose-3.11-rpms" \
    --enable="rhel-7-server-ansible-2.6-rpms"

yum install -y skopeo openssl docker-distribution tmux lsof telnet yum-utils createrepo git
```

##### Create fetch-data.sh & execute :

```bash
#!/usr/bin/env bash

###### Variables ######

SELF=$(basename $0)
VERSION=v3.11.117
MAJOR_VERSION_TAG=v3.11
declare -a IMAGES
IMAGES=(
    # Basis images
    "openshift3/apb-base:${VERSION}"
    "openshift3/apb-tools:${VERSION}"
    "openshift3/automation-broker-apb:${VERSION}"
    "openshift3/csi-attacher:${VERSION}"
    "openshift3/csi-driver-registrar:${VERSION}"
    "openshift3/csi-livenessprobe:${VERSION}"
    "openshift3/csi-provisioner:${VERSION}"
    "openshift3/grafana:${VERSION}"
    # Missing openshift3/image-inspector:${VERSION}
    "openshift3/image-inspector:${MAJOR_VERSION_TAG}"
    "openshift3/local-storage-provisioner:${VERSION}"
    "openshift3/manila-provisioner:${VERSION}"
    "openshift3/mariadb-apb:${VERSION}"
    "openshift3/mediawiki:${VERSION}"
    "openshift3/mediawiki-apb:${VERSION}"
    "openshift3/mysql-apb:${VERSION}"
    # Missing openshift3/ose-ansible:${VERSION}
    "openshift3/ose-ansible:${MAJOR_VERSION_TAG}"
    "openshift3/ose-ansible-service-broker:${VERSION}"
    "openshift3/ose-cli:${VERSION}"
    "openshift3/ose-cluster-autoscaler:${VERSION}"
    "openshift3/ose-cluster-capacity:${VERSION}"
    "openshift3/ose-cluster-monitoring-operator:${VERSION}"
    "openshift3/ose-console:${VERSION}"
    "openshift3/ose-configmap-reloader:${VERSION}"
    "openshift3/ose-control-plane:${VERSION}"
    "openshift3/ose-deployer:${VERSION}"
    "openshift3/ose-descheduler:${VERSION}"
    "openshift3/ose-docker-builder:${VERSION}"
    "openshift3/ose-docker-registry:${VERSION}"
    "openshift3/ose-efs-provisioner:${VERSION}"
    "openshift3/ose-egress-dns-proxy:${VERSION}"
    "openshift3/ose-egress-http-proxy:${VERSION}"
    "openshift3/ose-egress-router:${VERSION}"
    "openshift3/ose-haproxy-router:${VERSION}"
    "openshift3/ose-hyperkube:${VERSION}"
    "openshift3/ose-hypershift:${VERSION}"
    "openshift3/ose-keepalived-ipfailover:${VERSION}"
    "openshift3/ose-kube-rbac-proxy:${VERSION}"
    "openshift3/ose-kube-state-metrics:${VERSION}"
    "openshift3/ose-metrics-server:${VERSION}"
    "openshift3/ose-node:${VERSION}"
    "openshift3/ose-node-problem-detector:${VERSION}"
    "openshift3/ose-operator-lifecycle-manager:${VERSION}"
    "openshift3/ose-ovn-kubernetes:${VERSION}"
    "openshift3/ose-pod:${VERSION}"
    "openshift3/ose-prometheus-config-reloader:${VERSION}"
    "openshift3/ose-prometheus-operator:${VERSION}"
    "openshift3/ose-recycler:${VERSION}"
    "openshift3/ose-service-catalog:${VERSION}"
    "openshift3/ose-template-service-broker:${VERSION}"
    "openshift3/ose-tests:${VERSION}"
    "openshift3/ose-web-console:${VERSION}"
    "openshift3/postgresql-apb:${VERSION}"
    "openshift3/registry-console:${VERSION}"
    "openshift3/snapshot-controller:${VERSION}"
    "openshift3/snapshot-provisioner:${VERSION}"
    "rhel7/etcd:3.2.22"
    # Optional
    "openshift3/ose-efs-provisioner:${VERSION}"
    "openshift3/metrics-cassandra:${VERSION}"
    "openshift3/metrics-hawkular-metrics:${VERSION}"
    "openshift3/metrics-hawkular-openshift-agent:${VERSION}"
    "openshift3/metrics-heapster:${VERSION}"
    "openshift3/metrics-schema-installer:${VERSION}"
    "openshift3/oauth-proxy:${VERSION}"
    "openshift3/ose-logging-curator5:${VERSION}"
    "openshift3/ose-logging-elasticsearch5:${VERSION}"
    "openshift3/ose-logging-eventrouter:${VERSION}"
    "openshift3/ose-logging-fluentd:${VERSION}"
    "openshift3/ose-logging-kibana5:${VERSION}"
    "openshift3/prometheus:${VERSION}"
    # Missing openshift3/prometheus-alert-buffer:${VERSION}
    "openshift3/prometheus-alert-buffer:${MAJOR_VERSION_TAG}"
    "openshift3/prometheus-alertmanager:${VERSION}"
    "openshift3/prometheus-node-exporter:${VERSION}"
    "cloudforms46/cfme-openshift-postgresql"
    "cloudforms46/cfme-openshift-memcached"
    "cloudforms46/cfme-openshift-app-ui"
    "cloudforms46/cfme-openshift-app"
    "cloudforms46/cfme-openshift-embedded-ansible"
    "cloudforms46/cfme-openshift-httpd"
    "cloudforms46/cfme-httpd-configmap-generator"
    "rhgs3/rhgs-server-rhel7"
    "rhgs3/rhgs-volmanager-rhel7"
    "rhgs3/rhgs-gluster-block-prov-rhel7"
    "rhgs3/rhgs-s3-server-rhel7"
    # Builder images
    "jboss-amq-6/amq63-openshift:latest"
    "jboss-datagrid-7/datagrid71-openshift:latest"
    "jboss-datagrid-7/datagrid71-client-openshift:latest"
    "jboss-datavirt-6/datavirt63-openshift:latest"
    "jboss-datavirt-6/datavirt63-driver-openshift:latest"
    "jboss-decisionserver-6/decisionserver64-openshift:latest"
    "jboss-processserver-6/processserver64-openshift:latest"
    "jboss-eap-6/eap64-openshift:latest"
    "jboss-eap-7/eap71-openshift:latest"
    "jboss-webserver-3/webserver31-tomcat7-openshift:latest"
    "jboss-webserver-3/webserver31-tomcat8-openshift:latest"
    "openshift3/jenkins-2-rhel7:${VERSION}"
    "openshift3/jenkins-agent-maven-35-rhel7:${VERSION}"
    "openshift3/jenkins-agent-nodejs-8-rhel7:${VERSION}"
    "openshift3/jenkins-slave-base-rhel7:${VERSION}"
    "openshift3/jenkins-slave-maven-rhel7:${VERSION}"
    "openshift3/jenkins-slave-nodejs-rhel7:${VERSION}"
    "rhscl/mongodb-32-rhel7:latest"
    "rhscl/mysql-57-rhel7:latest"
    "rhscl/perl-524-rhel7:latest"
    "rhscl/php-56-rhel7:latest"
    "rhscl/postgresql-95-rhel7:latest"
    "rhscl/python-35-rhel7:latest"
    "redhat-sso-7/sso70-openshift:latest"
    "rhscl/ruby-24-rhel7:latest"
    "redhat-openjdk-18/openjdk18-openshift:latest"
    "redhat-sso-7/sso71-openshift:latest"
    "rhscl/nodejs-6-rhel7:latest"
    "rhscl/mariadb-101-rhel7:latest"
)


if [ "$1" == "" ] ; then
    echo "Please run $0 [sync|start-registry|sync-images|check-images|sync-rpms]"
    exit 1;
fi

if [ ! -d certs/ ] ; then mkdir certs/; fi;
if [ ! -f certs/localhost.crt ] ; then
    echo "Create certificate...."
    openssl req -newkey rsa:4096 -nodes -sha256 \
        -keyout certs/localhost.key -x509 -days 365 \
        -out certs/localhost.crt -subj "/CN=localhost"
fi;

if [ ! -f auth.json ] ; then
    echo "Please download auth.json to get access to registry.redhat.io"
    echo "  Go to: https://access.redhat.com/terms-based-registry/"
    exit 1;
fi;

function main {
    case $1 in 
        "start-registry")
            start-registry
        ;;
        "sync-images")
            sync-images
        ;;
        "check-images")
            check-images
        ;;
        "sync-rpms")
            sync-rpms
        ;;
        "sync")
            sync
        ;;
        *)
            echo "Please run $0 [sync|start-registry|sync-images|check-images|sync-rpms] ($1)"
            exit 1;
        ;;    
    esac
}


function start-registry { 
    #echo "start registry"; sleep 5; exit
    if [ ! -d registry/ ] ; then mkdir registry/; fi;
    export REGISTRY_HTTP_TLS_KEY=$PWD/certs/localhost.key
    export REGISTRY_HTTP_TLS_CERTIFICATE=$PWD/certs/localhost.crt
    export REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY=$PWD/registry
    registry serve /etc/docker-distribution/registry/config.yml
    exit;
}
function sync-images {
    #echo "sync-image"; sleep 5; exit
    for i in ${IMAGES[@]} ; do 
        IFS=':'; arrIN=($i); unset IFS;
        IMAGE=${arrIN[0]}
        TAG=${arrIN[1]:-latest}
        echo "# registry.redhat.io/$i -> localhost:5000/${IMAGE}:${TAG}";
        #continue;
        skopeo copy --dest-tls-verify=false \
          --authfile=auth.json \
          docker://registry.redhat.io/${IMAGE}:${TAG} \
          docker://localhost:5000/${IMAGE}:${TAG}

        echo "# localhost:5000/${IMAGE}:${TAG} -> localhost:5000/${IMAGE}:${MAJOR_VERSION_TAG}";
        skopeo copy --dest-tls-verify=false --src-tls-verify=false \
          --authfile=auth.json \
          docker://localhost:5000/${IMAGE}:${TAG} \
          docker://localhost:5000/${IMAGE}:${MAJOR_VERSION_TAG}
    done;
    echo "Check images in registry:"
    check-images
    exit;
}
function check-images { 
    #echo "Check image"; sleep 5; exit
    for i in ${IMAGES[@]} ; do 
        IFS=':'; arrIN=($i); unset IFS;
        IMAGE=${arrIN[0]}
        TAG=${arrIN[1]:-latest}
        curl -k --fail  -s -o /dev/null \
            https://localhost:5000/v2/$IMAGE/manifests/${TAG} \
            || echo "MISSING IN REGISTRY: ${IMAGE}:${TAG}"
    done;
    exit;
}
function sync-rpms { 
    #echo "Sync repo"; sleep 5; exit
    if [ ! -d repos/ ] ; then mkdir repos/; fi;
    SUDO=""
    if [ $UID != 0 ] ; then SUDO="sudo"; fi;
    for repo in \
        rhel-7-server-rpms \
        rhel-7-server-extras-rpms \
        rhel-7-server-ansible-2.6-rpms \
        rhel-7-server-ose-3.11-rpms
    do
        # ToDo: Please check maybe --newest-only is enough
        $SUDO reposync --gpgcheck -lm --newest-only --repoid=${repo} --download_path=repos/ 
        $SUDO createrepo -v repos/${repo} -o repos/${repo} 
    done
    exit;
}

function sync {
    if [ ! -z $TMUX ] ; then
        tmux split-window -t 0 -v 
	    tmux send-keys "./$SELF sync-rpms" C-m 
        tmux split-window -t 1 -h 
	    tmux send-keys "sleep 2; ./$SELF sync-images" C-m 
        ./$SELF start-registry
    else
        tmux new-session -d -s 'sync'
	    tmux send-keys "./$SELF start-registry" C-m 
        tmux split-window -t 0 -v 
	    tmux send-keys "./$SELF sync-rpms" C-m 
        tmux split-window -t 1 -h 
	    tmux send-keys "./$SELF sync-images" C-m 
        tmux attach-session -t 'sync'
    fi
    exit;
}

main $@
```

```text
chmod +x fetch-data.sh
./fetch-data.sh sync
```

Prepare package `tar cf ocp-3.11-data.tar registry/ repos/`

Go to the customer ;-\)

### Prepare bastion at customer

Extract repos into /var/www/html

```text
sudo tar -C /var/www/html -xf ocp-3.11-data.tgz repos/
```

Add repos into `/etc/yum.repos.d/ose.repo`

```text
[rhel-7-server-rpms]
name=rhel-7-server-rpms
baseurl=file:///var/www/html/repos/rhel-7-server-rpms
enabled=1
gpgcheck=0
[rhel-7-server-extras-rpms]
name=rhel-7-server-extras-rpms
baseurl=file:///var/www/html/repos/rhel-7-server-extras-rpms
enabled=1
gpgcheck=0
[rhel-7-server-ansible-2.6-rpms]
name=rhel-7-server-ansible-2.6-rpms
baseurl=file:///var/www/html/repos/rhel-7-server-ansible-2.6-rpms
enabled=1
gpgcheck=0
[rhel-7-server-ose-3.11-rpms]
name=rhel-7-server-ose-3.11-rpms
baseurl=file:///var/www/html/repos/rhel-7-server-ose-3.11-rpms
enabled=1
gpgcheck=0
```

Install webserver & docker registry

```text
sudo yum install -y httpd docker-distribution
```

Extract registry data:

```text
sudo tar -C /var/lib/registry/ --strip-components=1 -xf ocp-3.11-data.tgz registry/docker
```

## \[NOTE\]

## If SELinux is enabled: `chcon -R -t httpd_sys_content_t /var/www/html/repos/`

Startup webserver and registry

```text
systemctl enable docker-distribution.service httpd.service
systemctl start docker-distribution.service httpd.service
systemctl status docker-distribution.service httpd.service
```

## \[NOTE\]

ToDo: Check authentication for docker-distribution, it looks like everyone can push and overwrite images!

```text
REGISTRY_AUTH_HTPASSWD_PATH=/var/lib/registry/htpasswd
REGISTRY_AUTH=htpasswd
REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm
```

====

### Startup instances

Maybe run your host prep.

Repos are set up by prerequisites.yml

### Install OpenShift

Please replace `bastion:5000` and `bastion`

```text
[OSEv3:children]
masters
nodes
etcd

[OSEv3:vars]
ansible_ssh_user=cloud-user
ansible_become=yes
openshift_deployment_type=openshift-enterprise

# --- Important part for disconnected ----

# Cluster Image Source (registry) configuration
# openshift-enterprise default is 'registry.redhat.io/openshift3/ose-${component}:${version}'
# origin default is 'docker.io/openshift/origin-${component}:${version}'
oreg_url=bastion:5000/openshift3/ose-${component}:${version}
# If oreg_url points to a registry other than registry.redhat.io we can
# modify image streams to point at that registry by setting the following to true
openshift_examples_modify_imagestreams=true
# Add insecure and blocked registries to global docker configuration
openshift_docker_insecure_registries=['bastion:5000']
openshift_docker_blocked_registries=['registry.access.redhat.com', 'docker.io', 'registry.fedoraproject.org', 'quay.io', 'registry.centos.org']
# You may also configure additional default registries for docker, however this
# is discouraged. Instead you should make use of fully qualified image names.
openshift_docker_additional_registries=['bastion:5000']

# OpenShift repository configuration
openshift_additional_repos=[{'id': 'rhel-7-server-rpms', 'name': 'rhel-7-server-rpms', 'baseurl': 'http://bastion/repos/rhel-7-server-rpms', 'enabled': 1, 'gpgcheck': 0},{'id': 'rhel-7-server-extras-rpms', 'name': 'rhel-7-server-extras-rpms', 'baseurl': 'http://bastion/repos/rhel-7-server-extras-rpms', 'enabled': 1, 'gpgcheck': 0},{'id': 'rhel-7-server-ansible-2.6-rpms', 'name': 'rhel-7-server-ansible-2.6-rpms', 'baseurl': 'http://bastion/repos/rhel-7-server-ansible-2.6-rpms', 'enabled': 1, 'gpgcheck': 0},{'id': 'rhel-7-server-ose-3.11-rpms', 'name': 'rhel-7-server-ose-3.11-rpms', 'baseurl': 'http://bastion/repos/rhel-7-server-ose-3.11-rpms', 'enabled': 1, 'gpgcheck': 0}]

# Important: docker_image_availability, maybe the skopoe check did not work with your repo
openshift_disable_check=disk_availability,memory_availability,docker_image_availability


# Don't work very well, becaude ose-pod-v3.11.69 is hardcoded
#openshift_image_tag=v3.11

# Arg, hardcoded registry.redhat.io/....
#    https://github.com/openshift/openshift-ansible/blob/master/roles/etcd/defaults/main.yaml#L15
osm_etcd_image=bastion:5000/rhel7/etcd:3.2.22

# --- Important part for disconnected ----

os_sdn_network_plugin_name='redhat/openshift-ovs-multitenant'

openshift_node_groups=[{'name': 'node-config-all-in-one', 'labels': ['node-role.kubernetes.io/master=true', 'node-role.kubernetes.io/infra=true', 'node-role.kubernetes.io/compute=true']}]

# htpasswd auth
openshift_master_identity_providers=[{'name': 'htpasswd_auth', 'login': 'true', 'challenge': 'true', 'kind': 'HTPasswdPasswordIdentityProvider'}]
# Defining htpasswd users
openshift_master_htpasswd_users={'admin': '$apr1$5slPL.BP$waLoQ10SWU6HYokq1wV5t1', 'dev': '$apr1$xATuF0Is$amNbjuDTUN1eQP0hwdMGC0'}

[masters]
instance

[etcd]
instance

[nodes]
# openshift_node_group_name should refer to a dictionary with matching key of name in list openshift_node_groups.
instance openshift_node_group_name="node-config-all-in-one"
```

