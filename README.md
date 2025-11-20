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
After mirroring we will get below.
```bash

Success
Update image:  service.example.com:8443/ocp4/openshift4:4.18.9-x86_64
Mirror prefix: service.example.com:8443/ocp4/openshift4
Mirror prefix: service.example.com:8443/ocp4/openshift4:4.18.9-x86_64

To use the new mirrored repository to install, add the following section to the install-config.yaml:

imageContentSources:
- mirrors:
  - service.example.com:8443/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-release
- mirrors:
  - service.example.com:8443/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev


To use the new mirrored repository for upgrades, use the following to create an ImageContentSourcePolicy:

apiVersion: operator.openshift.io/v1alpha1
kind: ImageContentSourcePolicy
metadata:
  name: example
spec:
  repositoryDigestMirrors:
  - mirrors:
    - service.example.com:8443/ocp4/openshift4
    source: quay.io/openshift-release-dev/ocp-release
  - mirrors:
    - service.example.com:8443/ocp4/openshift4
    source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
```
Now prepare the install-config.yaml
```bash
apiVersion: v1
baseDomain: example.com
compute:
  - hyperthreading: Enabled
    name: worker
    replicas: 0
controlPlane:
  hyperthreading: Enabled
  name: master
  replicas: 3
metadata:
  name: lab # Cluster name
networking:
  clusterNetwork:
    - cidr: 10.128.0.0/14
      hostPrefix: 23
  networkType: OVNKubernetes
  serviceNetwork:
    - 172.30.0.0/16
platform:
  none: {}
fips: false
pullSecret: '{"auths":{"service.example.com:8443":{"auth":"aW5pdDpQWEtKQUxuODB4NTJpb3BaNnp5NE5WOTdkMXd1STNHbQ=="}}}'
sshKey: "ssh-ed25519 AAAA..."
additionalTrustBundle: |
 -----BEGIN CERTIFICATE-----
 MIID3zCCAsegAwIBAgIULyC6sHIjYdLAZmMOEiDrrtyZIYQwDQYJKoZIhvcNAQEL
 BQAwbTELMAkGA1UEBhMCVVMxCzAJBgNVBAgMAlZBMREwDwYDVQQHDAhOZXcgWW9y
 azENMAsGA1UECgwEUXVheTERMA8GA1UECwwIRGl2aXNpb24xHDAaBgNVBAMME3Nl
 cnZpY2UuZXhhbXBsZS5jb20wHhcNMjUxMTIwMTU0OTA0WhcNMjgwOTA5MTU0OTA0
 WjBtMQswCQYDVQQGEwJVUzELMAkGA1UECAwCVkExETAPBgNVBAcMCE5ldyBZb3Jr
 MQ0wCwYDVQQKDARRdWF5MREwDwYDVQQLDAhEaXZpc2lvbjEcMBoGA1UEAwwTc2Vy
 dmljZS5leGFtcGxlLmNvbTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEB
 AMw/ezcDndUhbp274vW2+Gf5wLwRBhUj9F9LFTETVv9cNDl9DImcbbeFHy4DfHgP
 NIZQNKkPwWXV4mwUpCFUkHYHqh4eU9yk5gBUNJ+mX9Vv7D7owKuSbcxgrdGcBP9L
 pQYNaZP8qCZ9TLaoOCA+FUToqw3Pt6SE8sO5UAKZFvCzrbsju45pA/hi4WpeAXXb
 Ub56UbmpUw22qBB0zz8Yvu/0QEzWQKXXeDPOyXJ8kZlVIqRqQ2BV2B01k06NWr/r
 KmBuiwvq0gv+NV/r7hEjD1ixQwXPGoDPZ9jdbMc+JmdLIoXIeS6KtvixQq6XNvoG
 3gzSfSFMGlk4F4NUpcY0pfUCAwEAAaN3MHUwCwYDVR0PBAQDAgLkMBMGA1UdJQQM
 MAoGCCsGAQUFBwMBMB4GA1UdEQQXMBWCE3NlcnZpY2UuZXhhbXBsZS5jb20wEgYD
 VR0TAQH/BAgwBgEB/wIBATAdBgNVHQ4EFgQUaLr3eha/cR4jsLtCEMjdZdxTUPIw
 DQYJKoZIhvcNAQELBQADggEBADQzY97TkTeui5ku2OaaqccnAyqmDtNHTEY1BIpI
 upLMQx/U5bjuG7r5uj/TrQ/nyKtC5ToRo9tcu86K3fnD0SOxpCckRu/t2XBJdc9Z
 VNeQ6YQfueSzZPJUNOTAgJA6k365ChyXKfOfKGivVNUMjvpiCp3aiKMD+boOYj0z
 QSV+b95SjUzB1dPo4Kb9/hqS+jr/0yhmtGxwYx+/Z8ptq902mgF434x+CzVU0xDU
 ViG94okJIrDpoqgXlLdkwUz6+y8AQ6VCMJvdQu1XUAX3peDJHNGksFF4gzVf2SO9
 BYsiELJCaU899KU5N2FF8JBKjnbK3o0W1GHvy1dcdUxzFu4=
 -----END CERTIFICATE-----
imageContentSources:
- mirrors:
  - service.example.com:8443/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-release
- mirrors:
  - service.example.com:8443/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
```
