# Pre-check for CPD Upgrade

---

1. A client workstation RHEL 8 with internet to download OCP and CPD images, it can be bastion or infra node 

OS level and make sure OS is REHL 8

```
cat /etc/redhat-release

```


Test internet connection, and make sure the output from the target url and it can be connected successfully.

```
curl -v https://github.com/IBM

curl -v https://icr.io

curl -v https://mirror.openshift.com/pub

```

Prepare customer's IBM entitlement key

  1) Log in to https://myibm.ibm.com/products-services/containerlibrary on My IBM with the IBMid and password that are associated with the entitled software.
  2) On the Get entitlement key tab, select Copy key to copy the entitlement key to the clipboard.
  3) Save the API key in a text file.

Prepare Red Hat pull secret

  1) Log in to https://cloud.redhat.com/openshift/install/pull-secret
  2) Click “Copy pull secret” 
  3) Save it as pullsecret_config.json

Make sure free disk space more than 800 GB (to download images and pack the images into a tar ball)

```
df -lh

```

2. Collect OCP and CPD cluster information

Log into OCP cluster from bastion node

```
oc login -u kubeadmin -p <kubeadmin-password> <ocp-api-server-url>

```

Get OCP version

```
oc version

```

Get storage classes

```
oc get sc

```

Check out OCP cluster status

Make sure all nodes in ready status

```
oc get nodes

```

Make sure all mc in correct status, UPDATED all Ture, UPDATING all False, DEGRADED all False

```
oc get mcp

```

Make sure all co in correct status, AVAILABLE all True, PROGRESSING all False, DEGRADED all False

```
oc get co

```

Make sure the unhealthypods have no pod unthealty, if there is, please open IBM support case (https://www.ibm.com/mysupport) to fix them.

```

oc get po --no-headers --all-namespaces -o wide| grep -Ev '1/1|2/2|3/3|4/4|5/5|6/6|7/7|8/8|9/9|10/10|Completed' > unhealthypods.txt

```

Get CPD installed projects

```
oc get pod -A | grep zen

```

Get CPD version and installed componenets

For CPD 4.5 and above

```
cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE}

```

For CPD 3.5

```
cpd-cli status --namespace <cpd-project>

```

Check the scheduling service, if it is installed but not in ibm-common-services project, uninstall it before upgrade

```
oc get scheduling -A

```

3. Bastion node and private container registry

Go to bastion node and check out private container registry

```
podman ps

```

Log into private container registry

```
podman login <private-registry-location> -u <user-name> -p <password>

```

Make sure bastion node has enough free disk to load images, more than 400 GB. If taking the air-gap installation approach, the free disk size need to be doubled to extract image tar ball.

```
df -lh

```

4. Walk through the cpd_vars.sh items and fill out each with correct value

```
#===============================================================================
# Cloud Pak for Data installation variables
#===============================================================================


# ------------------------------------------------------------------------------
# Client workstation
# ------------------------------------------------------------------------------

# export CPD_CLI_MANAGE_WORKSPACE=<enter a fully qualified directory>
# export OLM_UTILS_LAUNCH_ARGS=<enter launch arguments>


# ------------------------------------------------------------------------------
# Cluster
# ------------------------------------------------------------------------------

export OCP_URL=<ocp-api-server-url>  #e.g. https://api.example.ocp.ibm.com:6443
export OPENSHIFT_TYPE=self-managed
export IMAGE_ARCH=amd64
export OCP_USERNAME=kubeadmin
export OCP_PASSWORD=<kubeadmin-password>
# export OCP_TOKEN=<enter your token>


# ------------------------------------------------------------------------------
# Projects
# ------------------------------------------------------------------------------

export PROJECT_CPFS_OPS=ibm-common-services
export PROJECT_CPD_OPS=ibm-common-services
export PROJECT_CATSRC=openshift-marketplace
export PROJECT_CPD_INSTANCE=cpd-instance
# export PROJECT_TETHERED=<enter the tethered project>


# ------------------------------------------------------------------------------
# Storage for OCS
# Update them according to actually used storage classes
# ------------------------------------------------------------------------------

export STG_CLASS_BLOCK=ocs-storagecluster-ceph-rbd
export STG_CLASS_FILE=ocs-storagecluster-cephfs

# ------------------------------------------------------------------------------
# IBM Entitled Registry
# ------------------------------------------------------------------------------

export IBM_ENTITLEMENT_KEY=<customer-IBM-entitlement-key>

# ------------------------------------------------------------------------------
# Private container registry
# ------------------------------------------------------------------------------
# Set the following variables if you mirror images to a private container registry.
#
# To export these variables, you must uncomment each command in this section.

export PRIVATE_REGISTRY_LOCATION=<customer-private-registry>  #e.g. bastion.example.ocp.ibm.com:15000
export PRIVATE_REGISTRY_PUSH_USER=admin
export PRIVATE_REGISTRY_PUSH_PASSWORD=password
export PRIVATE_REGISTRY_PULL_USER=admin
export PRIVATE_REGISTRY_PULL_PASSWORD=password


# ------------------------------------------------------------------------------
# Cloud Pak for Data version
# ------------------------------------------------------------------------------

export VERSION=4.6.2

# ------------------------------------------------------------------------------
# Components
# ------------------------------------------------------------------------------
# Set the following variable if you want to install or upgrade multiple components at the same time.
#
# To export the variable, you must uncomment the command.

export COMPONENTS=cpfs,cpd_platform

```

5. Walk through the ocp_vars.sh 

```
# ------------------------------------------------------------------------------
# OCP Images Mirroring
# ------------------------------------------------------------------------------
# Collect these variables for mirroring OCP images
#

export REMOVABLE_MEDIA_PATH=/ibm/ocp/mirror
export PRODUCT_REPO="openshift-release-dev"
export RELEASE_NAME="ocp-release"
export OCP_RELEASE=4.10.51
export ARCHITECTURE=x86_64
export LOCAL_SECRET_JSON="/ibm/ocp/pullsecret_config.json"    # Customer's Red Hat pull secret
export LOCAL_REPOSITORY="ocp4/openshift4"
export PRIVATE_REGISTRY="registry.example.com:5000"       # Customer's private registry for OCP images

```

6. Apply ```cpd_vars.sh``` and ```ocp_vars.sh``` on client workstation or bastion node

```
source cpd_vars.sh
source ocp_vars.sh

```

Probe IBM registry

```
podman login cp.icr.io -u cp -p ${IBM_ENTITLEMENT_KEY}

```

Probe Red Hat OCP registry

```
podman login registry.redhat.io --authfile ${LOCAL_SECRET_JSON}

podman login quay.io --authfile ${LOCAL_SECRET_JSON}

```

7. Cluster network

During the upgrade, we takes MinIO (https://www.minio.io) to backup CPD cluster on bastion node, so port #9000 and #9001 need to be opened for bastion node ip address.


If you need download CPD CLI and OpenShift CLI in a restricted network, please add these urls into whitelist/allowlist.
  1. https://github.com/IBM/cpd-cli/releases/download : CPD CLI packages
  2. https://mirror.openshift.com/pub : OpenShift CLI packages
  3. https://docker.io : Images to setup Private Container Registry

If you need mirror CPD images into PCR (private container registry) in a restricted network, please add these urls into whitelist/allowlist.
  1. cp.icr.io/cp : Images that are pulled from the IBM Entitled Registry that require an entitlement key to download. Most of the IBM Cloud Pak for Data software uses this tag.
  2. icr.io/cpopen : Publicly available images that are provided by IBM and that don't require an entitlement key to download. The IBM Cloud Pak for Data operators use this tag.

If you need mirror OCP 4 images into PCR (private container registry) in a restricted network, please put these urls into whitelist/allowlist too.
  1. registry.redhat.io : provides core container images
  2. *.quay.io : provides core container images
  3. sso.redhat.com : the https://cloud.redhat.com/openshift site uses authentication from sso.redhat.com
  4. mirror.openshift.com : required to access mirrored installation content and images
  5. registry.connect.redhat.com : provides third-party images

---

End of document
