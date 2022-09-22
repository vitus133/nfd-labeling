# Labeling nodes with NFD #
This demo shows how to assign Openshift nodes with custom labels using node feature discovery operator (NFD).
NFD discovers node features related to hardware and OS and adding them to the kubernetes nodes as labels.
NFD is highly extensible, and can label nodes with custom labels defined as text files on the node file system, or even derived from stdout of user scripts. Why not use this flexibility to label a large amount of nodes?

## Prerequisites ##
1. A cluster (tested on OCP 4.12.0-ec.3) with kubeconfig set for cli access
2. The `butane` binary (get it from https://mirror.openshift.com/pub/openshift-v4/clients/butane/latest/).

## Workflow ##
### Deploying the NFD operator ###
In the root directory of this repository, run:

```bash
$ oc apply -f deploy
```
### Configuring the operator for custom labels ###
Custom label namespaces must be explicitly [enabled](deploy/nfd.yaml#L8)
### Preparing nodes and labels relations ###
As a user, you want to apply a set of labels to each worker node in the cluster. Start from planning and create a list of nodes and labels relations:
```text
vgrinberg-virt-worker-0.karmalabs.com:custom-label/parameter1=true,custom-label/parameter2=false
vgrinberg-virt-worker-1.karmalabs.com:custom-label/parameter1=false,custom-label/parameter2=true
```
When ready, copy and paste this list in the `butane` [template](labeling.bu.yaml):
```yaml
variant: openshift
version: 4.11.0
metadata:
  name: 99-worker-labeling
  labels:
    machineconfiguration.openshift.io/role: worker 
storage:
  files:
  - path: /etc/kubernetes/node-feature-discovery/source.d/conf/rules.txt
    mode: 420 
    overwrite: true
    contents:
      inline: |
        # Format: "hostname:key1=value1,key2=value2"
        vgrinberg-virt-worker-0.karmalabs.com:custom-label/parameter1=true,custom-label/parameter2=false
        vgrinberg-virt-worker-1.karmalabs.com:custom-label/parameter1=false,custom-label/parameter2=true
```
Deploy to your cluster by
```bash
$ butane labeling.bu.yaml | oc apply -f -
```
After the `machineconfig` operator will create the files, NFD will run the script that extracts a set of labels for each particular node from the list, and then apply them to the nodes:
```bash
$ oc describe no/vgrinberg-virt-worker-1.karmalabs.com
Name:               vgrinberg-virt-worker-1.karmalabs.com
Roles:              worker,worker-cnf
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    custom-label/parameter1=false
                    custom-label/parameter2=true

$ oc describe no/vgrinberg-virt-worker-0.karmalabs.com
Name:               vgrinberg-virt-worker-0.karmalabs.com
Roles:              worker,worker-cnf
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    custom-label/parameter1=true
                    custom-label/parameter2=false

```
Done!

## Caveats ## 
Applying the machineconfig to your cluster will restart the nodes.
However, there is a way around this. If the machineconfig is incorporated into ignition (as for example in [ztp workflow](https://github.com/openshift-kni/cnf-features-deploy/tree/master/ztp/gitops-subscriptions/argocd)) it will be installed at Day0 without additional rebooting.
Then, at Day 2, the NFD operator will be installed and configured, and the labels will be applied.
Any labelling changes after the installation will cost a reboot.



