# oc-mirror

```
echo "# oc-mirror" >> README.md
git init
git add README.md
git commit -m "first commit"
git branch -M main
git remote add origin https://github.com/alpha-wolf-jin/oc-mirror.git

git config --global credential.helper 'cache --timeout 72000'
git push -u origin main

git add . ; git commit -a -m "update README" ; git push -u origin main
```
**Reference:**

https://docs.openshift.com/container-platform/4.11/installing/disconnected_install/installing-mirroring-disconnected.html#installing-mirroring-disconnected

https://zhimin-wen.medium.com/openshift-4-10-image-mirroring-for-airgap-environment-f6bed61ea719

## Installing the oc-mirror OpenShift CLI plugin

**Download oc-mirror and openshift-install**
https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.11.18/

```
$ tar xvzf oc-mirror.tar.gz

$ chmod +x oc-mirror

$ sudo mv oc-mirror /usr/local/bin/.

$ oc version
Client Version: 4.11.18
Kustomize Version: v4.5.4
Server Version: 4.11.18
Kubernetes Version: v1.24.6+5658434

$ oc mirror help
...
Examples:
  # Mirror to a directory
  oc-mirror --config mirror-config.yaml file://mirror
  
  # Mirror to a directory without layer and image differential operations
  oc-mirror --config mirror-config.yaml file://mirror --ignore-history
  
  # Mirror to mirror publish
  oc-mirror --config mirror-config.yaml docker://localhost:5000
  
  # Publish a previously created mirror archive
  oc-mirror --from mirror_seq1_000000.tar docker://localhost:5000
  
  # Publish to a registry and add a top-level namespace
  oc-mirror --from mirror_seq1_000000.tar docker://localhost:5000/namespace
  
  # Generate manifests for previously created mirror archive
  oc-mirror --from mirror_seq1_000000.tar docker://localhost:5000/namespace --manifests-only
  
  # Skip metadata check during imageset publishing. This example shows a two-step process.
  # A differential imageset would have to be created with --ignore-history to be
  # successfully published with --skip-metadata-check.
  oc-mirror --config mirror-config.yaml file://mirror --ignore-history
  oc-mirror --from mirror_seq2_000000.tar docker://localhost:5000/namespace --skip-metadata-check
...
```

## Configuring credentials that allow images to be mirrored

1. Download your registry.redhat.io pull secret from:

https://console.redhat.com/openshift/install/pull-secret

2. Make a copy of your pull secret in JSON format:
```
$ cat ./pull-secret | jq . > <path>/<pull_secret_file_in_json> 
```

3. Save the file either as ~/.docker/config.json (or $XDG_RUNTIME_DIR/containers/auth.json.)

```
# cat ~/.docker/config.json | jq . >/tmp/pull-secret.json

# cat /tmp/pull-secret.json
{
  "auths": {
    "cloud.openshift.com": {
      "auth": "b3BlbnNoaW...",
      "email": "root@localhost"
    },
    "quay.io": {
      "auth": "b3Bl...",
      "email": "root@localhost"
    },
    "registry.connect.redhat.com": {
      "auth": "NTEzMTc...",
      "email": "root@localhost"
    },
    "registry.redhat.io": {
      "auth": "NTEzMTc5...",
      "email": "root@localhost"
    }
  }
}

```

4. Generate the base64-encoded user name and password or token for your mirror registry:


```
# echo -n 'init:0469w' | base64 -w0
aW5pdDowNDY5dw==
```

5. Edit the JSON file and add a section that describes your registry to it

```
# vi /tmp/pull-secret.json
{
  "auths": {
    "cloud.openshift.com": {
      "auth": "b3BlbnNoaW...",
      "email": "root@localhost"
    },
    "quay.io": {
      "auth": "b3Bl...",
      "email": "root@localhost"
    },
    "registry.connect.redhat.com": {
      "auth": "NTEzMTc...",
      "email": "root@localhost"
    },
    "registry.redhat.io": {
      "auth": "NTEzMTc5...",
      "email": "root@localhost"
    },
    "helper.example.com:8443": {
      "auth": "aW5pdDowNDY5dw==",
      "email": "root@localhost"
    }
  }
}

# cp /tmp/pull-secret.json ~/.docker/config.json
```

6. create online for value of pullSecret in install-config.yaml
```
# cat ~/.docker/config.json | jq -c
{"auths":{"cloud.openshift.com":{"auth":"b3Blbn...,"helper.example.com:8443":{"auth":"aW5pdDowNDY5dw==","email":"root@localhost"}}}
```

7. Run the oc mirror command to mirror the images from the specified image set configuration to a specified registry:
```
  # Output OpenShift release versions
  oc-mirror list releases
  
  # Output all OpenShift release channels list for a release
  oc-mirror list releases --version=4.11
  
  # List all OpenShift versions in a specified channel
  oc-mirror list releases --channel=stable-4.11
  
  # List all OpenShift channels for a specific version
  oc-mirror list releases --channels --version=4.11

# oc-mirror list releases --version=4.11
Listing stable channels. Use --channel=<channel-name> to filter.
Use oc-mirror list release --channels to discover other channels.

Channel: stable-4.11
Architecture: amd64
4.10.3
4.10.4
...
4.11.18
...

# cat imageset-platform.yaml
kind: ImageSetConfiguration
apiVersion: mirror.openshift.io/v1alpha2
archiveSize: 4
storageConfig:
  local:
    path: /root/labs/media/ocp4.11.18
mirror:
  platform:
    architectures:
    - amd64
    channels:
    - name: stable-4.11
      type: ocp
      minVersion: 4.11.18
      maxVersion: 4.11.18
    graph: true

# oc mirror --config=imageset-platform.yaml file:///root/labs/media/ocp4.11.18
...
uploading: file://openshift/release-images sha256:48748a1b43a9ce550e97c4fd9441f5918a1262a378b11da3f8331ca6c99166f4 857.9KiB
sha256:22e149142517dfccb47be828f012659b1ccf71d26620e6f62468c264a7ce7863 file://openshift/release-images:4.11.18-x86_64
info: Mirroring completed in 51m55.47s (4.836MB/s)
Creating archive /root/labs/media/ocp4.11.18/mirror_seq1_000000.tar
Creating archive /root/labs/media/ocp4.11.18/mirror_seq1_000001.tar
Creating archive /root/labs/media/ocp4.11.18/mirror_seq1_000002.tar
Creating archive /root/labs/media/ocp4.11.18/mirror_seq1_000003.tar
```

## Mirroring from disk to mirror

```
# cd /root/labs/media/ocp4.11.18/
# tree
.
├── mirror_seq1_000000.tar
├── mirror_seq1_000001.tar
├── mirror_seq1_000002.tar
├── mirror_seq1_000003.tar
├── oc-mirror-workspace
└── publish

# oc mirror --from=./ docker://helper.example.com:8443
...
info: Planning completed in 310ms
uploading: helper.example.com:8443/openshift/release sha256:f903aebcb9693cbb029800b01ee767ec906f9261535fbbcd09429eb8b5928069 22.94MiB
uploading: helper.example.com:8443/openshift/release sha256:7fc593bf62a0d2d13f0d9f8c24bc8c90e4ed3c49f6c9933d27adf157c092929c 17.19KiB
sha256:119575c2aa477a5216fc18144f23ff48666e950ad7bca70d13af93cc6f2a54d2 helper.example.com:8443/openshift/release:4.11.18-x86_64-telemeter
info: Mirroring completed in 6.42s (3.748MB/s)
Wrote release signatures to oc-mirror-workspace/results-1674629926
Writing image mapping to oc-mirror-workspace/results-1674629926/mapping.txt
Writing UpdateService manifests to oc-mirror-workspace/results-1674629926
Writing ICSP manifests to oc-mirror-workspace/results-1674629926

# tree 
.
├── mirror_seq1_000000.tar
├── mirror_seq1_000001.tar
├── mirror_seq1_000002.tar
├── mirror_seq1_000003.tar
├── oc-mirror-workspace
│   ├── publish
│   └── results-1674629926
│       ├── charts
│       ├── imageContentSourcePolicy.yaml
│       ├── mapping.txt
│       ├── release-signatures
│       │   └── signature-sha256-22e149142517dfcc.json
│       └── updateService.yaml
└── publish
```

```
# cat oc-mirror-workspace/results-1674629926/imageContentSourcePolicy.yaml
---
apiVersion: operator.openshift.io/v1alpha1
kind: ImageContentSourcePolicy
metadata:
  name: generic-0
spec:
  repositoryDigestMirrors:
  - mirrors:
    - helper.example.com:8443/ubi8
    source: registry.access.redhat.com/ubi8
---
apiVersion: operator.openshift.io/v1alpha1
kind: ImageContentSourcePolicy
metadata:
  name: release-0
spec:
  repositoryDigestMirrors:
  - mirrors:
    - helper.example.com:8443/openshift/release
    source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
  - mirrors:
    - helper.example.com:8443/openshift/release-images
    source: quay.io/openshift-release-dev/ocp-release
```

6 directories, 8 files


```


# Mirror settings for operator images

1. List available operator index for this version
```
# oc mirror list operators --catalogs --version 4.11
Available OpenShift OperatorHub catalogs:
OpenShift 4.11:
registry.redhat.io/redhat/redhat-operator-index:v4.11
registry.redhat.io/redhat/certified-operator-index:v4.11
registry.redhat.io/redhat/community-operator-index:v4.11
registry.redhat.io/redhat/redhat-marketplace-index:v4.11
```

2. List the operators in the index

```
# oc mirror list operators --catalog=registry.redhat.io/redhat/redhat-operator-index:v4.11
NAME                                          DISPLAY NAME                                                                 DEFAULT CHANNEL
3scale-operator                               Red Hat Integration - 3scale - Managed Application Services                  threescale-mas
advanced-cluster-management                   Advanced Cluster Management for Kubernetes                                   release-2.6
amq-broker-rhel8                              Red Hat Integration - AMQ Broker for RHEL 8 (Multiarch)                      7.10.x
amq-online                                    Red Hat Integration - AMQ Online                                             stable
...
local-storage-operator                        Local Storage                                                                stable
...
```

3. List individual operator’s channel
```
# oc mirror list operators --catalog=registry.redhat.io/redhat/redhat-operator-index:v4.11 --package=local-storage-operator
NAME                    DISPLAY NAME   DEFAULT CHANNEL
local-storage-operator  Local Storage  stable

PACKAGE                 CHANNEL  HEAD
local-storage-operator  stable   local-storage-operator.4.11.0-202301062015

# oc mirror list operators --catalog=registry.redhat.io/redhat/redhat-operator-index:v4.11 --package=local-storage-operator --channel=stable
VERSIONS
4.11.0-202301062015
```




