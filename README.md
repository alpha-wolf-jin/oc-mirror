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




