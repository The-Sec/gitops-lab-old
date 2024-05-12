<div align="center">
  <img src="./docs/assets/talos.svg" alt="Talos Linux logo" width="150" height="150">
  <img src="./docs/assets/kubernetes.svg" alt="Kubernetes logo" width="150" height="150">
</div>

<div align=center>

### GitOps-Lab, the Home-Lab Repository.

_... powered by Talos, Kubernetes and my imagination_

</div>

<div align="center">
  <img src="https://img.shields.io/badge/v1.7.1-a?style=for-the-badge&logo=talos&logoColor=fff&label=Talos&labelColor=302d41&color=cba6f7" alt="Talos version">
  <img src="https://img.shields.io/badge/v1.30.0-a?url=https%3A%2F%2Fkromgo.cjsolsen.com%2Fquery%3Fformat%3Dendpoint%26metric%3Dkubernetes_version&style=for-the-badge&logo=kubernetes&logoColor=fff&label=Kubernetes&labelColor=302d41&color=cba6f7" alt="Kubernetes version">
  <img src="https://img.shields.io/badge/ArgoCD-v2.10.6-cba6f7?logo=argo&logoColor=fff&style=for-the-badge&labelColor=302D41" alt="ArgoCD version">
  <img src="https://img.shields.io/github/issues-pr/the-sec/gitops-lab?logo=github&color=f2cdcd&logoColor=fff&style=for-the-badge&labelColor=302d41" alt="Open Pull Requests">
</div>

---

## Intro
Due to my hunger in search of more insight, more knowledge and more power, there are mutiple clusters in this repo. The repo will be split up into sections. the base branch will only contain the most basic applications needed to start a cluster.

### XS4(all)
The xs4 tag is one of the cluster running in an dataceter. There is a montly cost assosiated with it, but a lot more stable in terms of internet access, power, etc. and i there for feel the price is justifed. While this is a VMware vSphere cluster there is only the ability to access vSphere using a RDP to web gui avalible for management of the resources. This means amazing tools like [GOVC](https://github.com/vmware/govmomi/tree/main) are out of the question 🙁.


### SWH
The swh tag is my arm (pi) cluster running at home, unfortunatly power and internet acccess are not very stable. 
that is located in a datacenter where i rent resources on a monthly bases. 


# XS4 Walkthrough
## setup encryption key's
We use [Mozilla sops](https://github.com/mozilla/sops) and [KSOPS](https://github.com/viaduct-ai/kustomize-sops) with [age](https://github.com/FiloSottile/age).

### age
We use age for encryption secrets en data that shouldn't be other than your eye's only and ArgoCD's eye's. together with Sops and KSops in we enable ArgoCD to decrypt the encrypted elementes within our yaml files.

We start off with generating a new set of key's.
```sh
  % % age-keygen -o key.txt
Public key: age1clr58c9k6xl3pq23zy25vv26rglv7s3xv9r7x3297m8vlc7ane4sxnr4vm
```
Both the Public key shown on the command line as the private key located inside the ./key.txt file should be stored in a safe place (KeePassXC, 1Password, etc.). The file ./key.txt will contain a string for easy of use. Its not very good security practice to leave private encryption key's on disk un-protected so delete the key.txt file for good measure.

```sh
  % unlink ./key.txt
```

Now with encryption keys, we can start working on the Talos cluster and encrypt any sensitive information with these keys.

### Talos
While we setup the cluster one could leave the secrets in the default talosconfig file, but for managability its easier to have them seperated. So below we first generate the secrets for Talos.
```sh
 % talosctl gen secrets --output-file ./clusters/xs4/talos/secrets.yaml
```
We need to store the following thow valus in envourment variables in order to pass them to the following commands we will use.
```bash
#Name your cluster Don't forget to update ./clusters/xs4/talos/patches/cluster-name.yaml
export CLUSTER_NAME="xs4"
#Define a FQDN or IP address that will point to your control node(s).
#Mutiple A records are alllowed in case of multiple control nodes.
export API_ENDPOINT="https://<CONTROLNODE_IP>:6443"
```
Lets generate the config file so we can start with Talos.
```shell
talosctl gen config \
$CLUSTER_NAME $API_ENDPOINT \
--with-secrets ./clusters/xs4/talos/secrets.yaml \
--output-types talosconfig \
--output ./clusters/xs4/talos/talosconfig
```
Now that we have a talosconfig file and we want to keep it seperated the easiest way to have the talosctl us it, is to store a referance to the file in an envrourment var. 
```shell
export TALOSCONFIG=./clusters/xs4/talos/talosconfig
talosctl gen config \
$CLUSTER_NAME $API_ENDPOINT \
--with-secrets ./clusters/xs4/talos/secrets.yaml \
--config-patch @./clusters/xs4/talos/patches/cluster-name.yaml \
--config-patch @./clusters/xs4/talos/patches/install-disks.yaml \
--config-patch @./clusters/xs4/talos/patches/multihomed.yaml \
--config-patch @./clusters/xs4/talos/patches/network.yaml \
--config-patch @./clusters/xs4/talos/patches/vmware.yaml \
--config-patch @./clusters/xs4/talos/nodes/tc01.yaml \
--output-types controlplane \
--output ./clusters/xs4/talos/tc01.yaml
talosctl gen config \
$CLUSTER_NAME $API_ENDPOINT \
--with-secrets ./clusters/xs4/talos/secrets.yaml \
--config-patch @./clusters/xs4/talos/patches/cluster-name.yaml \
--config-patch @./clusters/xs4/talos/patches/install-disks.yaml \
--config-patch @./clusters/xs4/talos/patches/network.yaml \
--config-patch @./clusters/xs4/talos/patches/vmware.yaml \
--config-patch @./clusters/xs4/talos/nodes/tw02.yaml \
--output-types worker \
--output ./clusters/xs4/talos/tw02.yaml
talosctl gen config \
$CLUSTER_NAME $API_ENDPOINT \
--with-secrets ./clusters/xs4/talos/secrets.yaml \
--config-patch @./clusters/xs4/talos/patches/cluster-name.yaml \
--config-patch @./clusters/xs4/talos/patches/install-disks.yaml \
--config-patch @./clusters/xs4/talos/patches/mayastor.yaml \
--config-patch @./clusters/xs4/talos/patches/network.yaml \
--config-patch @./clusters/xs4/talos/patches/vmware.yaml \
--config-patch @./clusters/xs4/talos/nodes/tw03.yaml \
--output-types worker \
--output ./clusters/xs4/talos/tw03.yaml
```

```sh
talosctl apply-config --nodes 192.168.80.127 --insecure --file ./clusters/xs4/talos/tc01.yaml
talosctl bootstrap -n 192.168.80.91
```

```sh
 % sops -age <Public key> -e -i ./secrets.yaml
```

## Prepare nodes
Talos was at time of starting this journey the best choice as its an OS with only a few binaries in order to get kubernetes up and running. It doesn't have a shell or even a SSH server to login into. This means there are far less components and binaries involved that need mainenance or could pose an potential security risk as any point in time.
[Node Requierments](https://www.talos.dev/v1.7/introduction/system-requirements/)

#### RESET NODE
reset wipe the whole drive but that doesn't work if you don't deploy nodes using PXE. So lets only wipe the partisions that contain state data(META, STATE and EPHEMERAL).
//https://www.talos.dev/v1.7/learn-more/architecture/#file-system-partitions
```bash
talosctl reset --system-labels-to-wipe META --system-labels-to-wipe STATE  --system-labels-to-wipe EPHEMERAL --reboot
```

#### upscale cluster
just run apply-config to machines not joined to the Talos cluster. To view nodes run
```bash
talosctl get members
```
```sh
talosctl apply-config --insecure --nodes $NODE_IP --file <CONTROLPLANE-FILE|WORKER-FILE>
```

### upgrade nodes
talosctl upgrade --preserve=true --nodes 192.168.80.91,192.168.80.92,192.168.80.93

## VMWare vSphere (OPTIONAL)
The addisional container needs to be able to authenticate against the talos API.
```zah
#Create authentication credentials
talosctl config new vmtoolsd-secret.yaml --roles os:admin
#Store credentials as Secret within our kube clsuter for the vmtoolsd pod to access them.
kubectl -n kube-system create secret generic talos-vmtoolsd-config --from-file=talosconfig=./vmtoolsd-secret.yaml

```