---
title:  OCP/OKD Support
weight: 4
---

To deploy Longhorn on a cluster provisioned with OpenShift 4.x, some additional configurations are required.

> **Note**: OKD currently does not support the ARM platform. For more information, see the [OKD website](https://www.okd.io) and [GitHub issue #1165](https://github.com/okd-project/okd/issues/1165) (*OKD in ARM platform*).

- [Install Longhorn](#install-longhorn)
  - [Install With Helm](#install-with-helm)
  - [Install With `oc` Command](#install-with-oc-command)
- [Prepare A Customized Default Longhorn Disk (Optional)](#prepare-a-customized-default-longhorn-disk-optional)
  - [Add An Extra Disk to Longhorn Storage](#add-an-extra-disk-to-longhorn-storage)
    - [Create Filesystem For The Device](#create-filesystem-for-the-device)
    - [Mounting The Device On Boot with MachineConfig CRD](#mounting-the-device-on-boot-with-machineconfig-crd)
    - [Label and Annotate The Node](#label-and-annotate-the-node)
- [Reference](#reference)
- [Main Contributor](#main-contributor)

## Install Longhorn

### Install With Helm

Please refer to this section [Install with Helm](../../../deploy/install/install-with-helm/) first.

Install Longhorn with the following settings:

| Setting | Value | Example | 
| --- | --- | --- |
| `openshift.enabled` | `true` | N/A |
| `image.openshift.oauthProxy.repository` | Upstream image | `quay.io/openshift/origin-oauth-proxy` |
| `image.openshift.oauthProxy.tag` | Version 4.1 or later | `4.18` |

```bash
  helm install longhorn longhorn/longhorn \
    --namespace longhorn-system \
    --create-namespace \
    --set openshift.enabled=true \
    --set image.openshift.oauthProxy.repository=quay.io/openshift/origin-oauth-proxy \
    --set image.openshift.oauthProxy.tag=4.18
```

### Install With `oc` Command

Perform the following steps to install Longhorn on [OKD](https://www.okd.io/) clusters.

1. Download the `longhorn-okd.yaml` file.
  ```
  wget https://raw.githubusercontent.com/longhorn/longhorn/v{{< current-version >}}/deploy/longhorn-okd.yaml
  ```
1. Specify the target `oauth-proxy` container image in the `longhorn-okd.yaml` file (for example, `quay.io/openshift/origin-oauth-proxy:4.18`).

1. Run the following command:
  ```shell
  oc apply -f longhorn-okd.yaml
  ```

One way to monitor the progress of the installation is to watch pods being created in the `longhorn-system` namespace:

  ```shell
    oc get pods \
    --namespace longhorn-system \
    --watch
  ```

For more information, see [Install with Kubectl](../../../deploy/install/install-with-kubectl).

## Prepare A Customized Default Longhorn Disk (Optional)

To understand more about configuring the disks for Longhorn, please refer to the section [Configuring Defaults for Nodes and Disks](../../../nodes-and-volumes/nodes/default-disk-and-node-config/#launch-longhorn-with-multiple-disks)

Longhorn will use the directory `/var/lib/longhorn` as default storage mount point and that means Longhorn use the root device as the default storage. If you don't want to use the root device as the Longhorn storage, set ***defaultSettings.createDefaultDiskLabeledNodes*** true when installing Longhorn by helm:

```txt
--set defaultSettings.createDefaultDiskLabeledNodes=true
```

And then add another device formatted to Longhorn storage

### Add An Extra Disk to Longhorn Storage

#### Create Filesystem For The Device

Create the filesystem on the device with the label `longhorn` on the storage node. Get into the node by oc command:

```bash
oc get nodes --no-headers | awk '{print $1}'
oc debug node/${NODE_NAME} -t -- chroot /host bash
```

Check if the device is present and format it with Longhorn label:

```bash
lsblk
sudo mkfs.ext4 -L longhorn /dev/${DEVICE_NAME}
```

#### Mounting The Device On Boot with MachineConfig CRD

The secondary drive needs to be mounted automatically when node boots up by the `MachineConfig` that can be created and deployed by:

```bash
cat <<EOF >>auto-mount-machineconfig.yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: worker
  name: 71-mount-storage-worker
spec:
  config:
    ignition:
      version: 3.2.0
    systemd:
      units:
        - name: var-mnt-longhorn.mount
          enabled: true
          contents: |
            [Unit]
            Before=local-fs.target
            [Mount]
            # Example mount point, you can change it to where you like for each device.
            Where=/var/mnt/longhorn
            What=/dev/disk/by-label/longhorn
            Options=rw,relatime,discard
            [Install]
            WantedBy=local-fs.target
EOF

oc apply -f auto-mount-machineconfig.yaml
```

#### Label and Annotate The Node

Please refer to the section [Customizing Default Disks for New Nodes](../../../nodes-and-volumes/nodes/default-disk-and-node-config/#customizing-default-disks-for-new-nodes) to label and annotate storage node on where your device is by oc commands:

```bash
oc get nodes --no-headers | awk '{print $1}'

oc annotate node ${NODE_NAME} --overwrite node.longhorn.io/default-disks-config='[{"path":"/var/mnt/longhorn","allowScheduling":true}]'
oc label node ${NODE_NAME} --overwrite node.longhorn.io/create-default-disk=config
```

**Note**: You might need to reboot the node to validate the modified configuration.

## Reference

- [OCP/OKD Documentation and Helm Support](https://github.com/longhorn/longhorn/pull/5004)
- [OKD Official Website](https://www.okd.io/)
- [OKD Official Documentation Website](https://docs.okd.io/latest/welcome/index.html)
- [oauth-proxy](https://github.com/openshift/oauth-proxy/blob/master/contrib/sidecar.yaml)

## Main Contributor

- [@ArthurVardevanyan](https://github.com/ArthurVardevanyan)
