## Installing Mirror Registry on RHEL9

1. Download the mirror registry bundle from this page - https://console.redhat.com/openshift/downloads#tool-mirror-registry
2. Make sure login to the root user with *sudo su** instead of only **su**
3. After downloading extract the mirror registry files into root user's home directory.
4. Set hoatname and make an entry in /etc/hosts file for mirror registry. I am using bastion.example.com as registry's FQDN.
```bash
hostnamectl set-hostname service.example.com
echo "192.168.200.20 service.example.com" >> /etc/hosts
```
6. Then run the mirror-registry installation script.
```bash
./mirror-registry install   --quayHostname service.example.com   --quayRoot /mirror
```
7. After the installation we will get a username and random password to login on registry.
```bash
PLAY RECAP ***************************************************************************************************************************************
root@service.example.com   : ok=48   changed=25   unreachable=0    failed=0    skipped=16   rescued=0    ignored=0   

INFO[2025-11-20 10:49:39] Quay installed successfully, config data is stored in /mirror 
INFO[2025-11-20 10:49:39] Quay is available at https://service.example.com:8443 with credentials (init, PXKJALn80x52iopZ6zy4NV97d1wuI3Gm) 
```
Download the pull secret from Red hat and save as json
```bash
cat ~/pull-secret.txt | jq . > ~/pull-secret.json
```
Also save your registry credentials same pull-secret
```bash
podman login --authfile ~/pull-secret.json -u init -p PXKJALn80x52iopZ6zy4NV97d1wuI3Gm service.example.com:8443 --tls-verify=false
```
Copy registry certificate in anchors.
```bash
cp /mirror/quay-rootCA/rootCA.pem /etc/pki/ca-trust/source/anchors/
cat /mirror/quay-config/ssl.cert >> /etc/pki/ca-trust/source/anchors/rootCA.pem
update-ca-trust
```
Set below variables.
```bash
OCP_RELEASE=4.18.9
LOCAL_REGISTRY='service.example.com:8443'
PRODUCT_REPO='openshift-release-dev'
LOCAL_SECRET_JSON='/root/pull-secret.json'
RELEASE_NAME="ocp-release"
ARCHITECTURE=x86_64
LOCAL_REPOSITORY=ocp4/openshift4
```
Mirror the images
```bash
oc adm release mirror -a ${LOCAL_SECRET_JSON}       --from=quay.io/${PRODUCT_REPO}/${RELEASE_NAME}:${OCP_RELEASE}-${ARCHITECTURE}      --to=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY}      --to-release-image=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY}:${OCP_RELEASE}-${ARCHITECTURE} --dry-run
```
