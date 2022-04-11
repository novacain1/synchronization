# Silicom TimeSync Operator on OpenShift
<!--
We want to publish a blog that contains a guided example of using the STS Operator on OpenShift. This should contain:

-   Operator Overview
-   Pictorial Overview of T-GM, T-BC, T-SC concepts
-   Operator Installation
-   Using T-GM
-->

<!-- Red Hat® OpenShift® blah blah blah.-->

Below are the steps to install the Silicom TimeSync Operator on Red Hat OpenShift. Towards the end of the installation, we will monitor the time synchronization functionalities on T-GM node.

## Table of Contents

1. [Pre-requisites](#intro)
2. [Telecom Grandmaster Background](#background)
3. [Installing Silicom Timesync Operator](#installation)
4. [Telecom Grandmaster Provisioning](#stsconfig)
6. [Telecom Grandmaster Operation](#stsops)
7. [Uninstalling Silicom Timesync Operator](#uninstalling)
8. [Wrapup](#conclusion)
9. [References](#refs)

The term *Project* and *namespace* maybe used interchangeably in this guide.

## Pre-requisites <a name="pre-requisites"></a>
- Terminal environment
  - Your terminal has the following commands
    - [oc](https://access.redhat.com/documentation/en-us/openshift_container_platform/4.9/html/cli_tools/openshift-cli-oc) binary
    - [git](https://git-scm.com/downloads) binary
- Physical card itself
  - STS2, STS4
  - Connections including GNSS
- [Authenticate as Cluster Admin inside your environment](https://access.redhat.com/documentation/en-us/openshift_container_platform/4.9/html/cli_tools/openshift-cli-oc#cli-logging-in_cli-developer-commands) of an OpenShift 4.9 Cluster.
- Your cluster meets the minimum requirements for the Operator:
  - 1 worker nodes, with at least:
    - 8 logical CPU
    - 24 GiB memory
    - 1+ storage devices

## Telecom Grandmaster Background <a name="background"></a>

<!-- Background on T-GM with pictures? -->
Precise timing is of paramount importance for 5G RAN. 5G RAN leverage of sophisticated technologies to maximize achieved data rates. These techniques rely on tight synchronization between varius elements of the 5G Radio Access Network (RAN). Not getting timing right means mobile subscribers are likely to suffer a poor user experience. Typically, this requires receivers of a Global Navidation Satellite Systems (GNSS) such as GPS, and protocols to transport efficiently the timing information to where it is needed. 
<!-- a high-level picture with long-run goal with the operator -->
![Sync Background](imgs/back.png)
<!-- T-GM -->
As you can see in the picture above, in a packet-based network PTP is the protocol of choice to carry time information. The synchronization solution consists of the following elements:

- The recipient of the GNSS information in a PTP network is referred to as the grandmaster (GM) clock. The GM clock is conencted to the GNSS receiver and provides the source of time for the connected network elements lower in the synchronization hierarcy. 

- The slave functionality terminates the PTP protocol and tries to estimate the correct time from the master. An O-RRU contain the slave functionality and takes the time information from the slave for its usage.

- The boundary clock consists of both GM and SC functionalities. At the slave side, it receives PTP packet from the GM or another boundary clock, terminates the PTP, and estimates timing from the GM. At master side, a new PTP packet is created based on the timing information of the boundary clock and pass it to the next boundary or slave clock in the chain. 

<!-- Silicom -->
We need cards and timings stack to feed timing sync to other O-RAN network elements.

<!-- operator needed-->
....In what follows, we present a guide to use the Silicom operator to automate the installation, monitoring, and operation of Silicom cards as well as the time sync stack (i.e., the operands). 

## Installing Silicom Timesync Operator <a name="installation"></a>

There two Operands to manage: one is the Silicom TimeSync cards and the other is the TimeSync sw stack. The operator dramatically simplifies the configuration and management 

#### Detect Silicom Card

* After installing the cards conforming a representative topology 

#### Time Sync Stack 

Before you begin. Check you have an OpenShift 4.8/4.9 fresh cluster with NFD and SRO operators installed.
<!-- topology -->

### Install from the embedded OperatorHub

<!-- Install steps clearly defined on a FRESH CLUSTER with output-->

* Alpha and beta: select alpha.

* The only installation mode supported from the operatorhub is for all namespaces in the cluster: operator will be available in all Namespaces. This means that the namespaces this operator can watch are ALL.

* Aim: Supported InstallModes come predefine by CSV. We support ownNamespace, singelNamespace, AllNamespace. 

* We select the namespace from the operatorhub console. However, note that to allow the installation in the selected namespace it is required that:
      - The operator must be a member of an operatorgroup that selects one namespace (ownNamespace or singlenamespace).
      - Operator is considered to be a member of an operatorgroup if 1) CSV of the operator is installed in the same namespace as the operator group, 2) install mode in CSV support the namespaces targetted by the [operator group][1]     

* Pre-requisites: NodeFeature Discovery operator and Special Resource Operator must be installed. In which namespace?
      - Supported: NFD + SRO MUST be installed in the same namespace as Silicom Operator. The namespace can be selected and by default `openshift-silicom` will be created.
      - Unsupported: However we need the flexibility to select the namespace where the three operators will be located. Next version: 
          * NFD, SRO and Silicom can live in their own namespaces
          * Silicom operand should live in a different namespace than silicom operator.

* The operator gets installed in namespace `openshift-silicom` by default or in the namespace specified by the OCP administrator.


2. Provision StsOperatorConfig CR object to provision the desired timing stack configuration

```yaml
cat <<EOF | oc apply -f -
apiVersion: sts.silicom.com/v1alpha1
kind: StsOperatorConfig
metadata:
  name: sts-operator-config
  namespace: openshift-operators
spec:
  images:
    #artifacts composing silicom timing sync stack
    tsyncd: quay.io/silicom/tsyncd:2.1.1.0
    tsyncExtts: quay.io/silicom/tsync_extts:1.0.0 
    phcs2Sys: quay.io/silicom/phcs2sys:3.1.1
    grpcTsyncd: quay.io/silicom/grpc-tsyncd:2.1.1.0 
    stsPlugin: quay.io/silicom/sts-plugin:0.0.3
    gpsd: quay.io/silicom/gpsd:3.23.1
  grpcSvcPort: 50051
  gpsSvcPort: 2947
  sro:
    build: false
EOF
```  

### Install from the CLI

<!-- Omit this step Install steps clearly defined on a FRESH CLUSTER with output-->


## Telecom Grandmaster Provisioning <a name="stsconfig"></a>

<!-- Show the user how to place the card in T-GM mode -->
* Supported: Currently, the administrator must manually set labels in the nodes to identify what worker nodes inside the OCP cluster can get the T-TGM.8275.1 role.
* Not supported: automated discovery of nodes that are directly connected to the GPS signal to add the proper label.

1. Add a node label `gm-1` in the worker node that has GPS cable connected to the (i.e., du3-ldc1).

```console
oc label node du3-ldc1 sts.silicom.com/config="gm-1" 
```

2. Create a StsConfig CR object to provision the desired Telecom PTP profile (i.e., T-GM.8275.1)

3. For a full listing of the possible configuration parameters and their possible values.

``` console
oc explain StsConfig.spec
```

* Gnss configuration explainations

``` console
oc explain StsConfig.spec.GnssSpec
```

```yaml
cat <<EOF | oc apply -f -
apiVersion: sts.silicom.com/v1alpha1
kind: StsConfig
metadata:
  name: gm-1
  namespace: openshift-operators
spec:
  namespace: openshift-operators
  imageRegistry: quay.io/silicom
  nodeSelector:
    sts.silicom.com/config: "gm-1"
  mode: T-GM.8275.1
  twoStep: 0
  esmcMode: 2
  ssmMode: 1
  forwardable: 1
  synceRecClkPort: 3
  syncOption: 1
  gnssSpec:
    gnssSigGpsEn: 1
  interfaces:
    - ethName: enp81s0f2
      holdoff: 500
      synce: 1
      mode: Master
      ethPort: 3
      qlEnable: 1
      ql: 2
EOF                 
```

3. Provision the timing stack in the node labelled as `sts.silicom./config: "gm-1"` 

* Unsupported: Creation of timing sync stack in a different namespace than the operator.

```console
gm-1-du3-ldc1-gpsd-964ch                  2/2     Running   0          49m
gm-1-du3-ldc1-phc2sys-6fv9k               1/1     Running   0          49m
gm-1-du3-ldc1-tsync-pkxwv                 2/2     Running   0          44m
```

* Pods above represent the timing solution for T-GM. Zooming into the pods show picture below with the resulting deployment in node du3-ldc1. 

![Timing Stack](imgs/tgm.png)



<!--
mode: Telecom PTP profiles as defined by the ITU-T. T-GM.8275.1 PTP profile corresponds to the profile for the RAN fronthaul network. 
twoStep and forwardable are PTP parameters.
gnssSigGpsEn:
esmcMode, ssmMode, and synceRecClkPort
syncOption:
interfaces
-->

## Telecom Grandmaster Operation <a name="stsops"></a>

The timing stack is deployed but, how do we know it is synchronizing the clock in the Silicom network card?
The time sync stack exposes an API based on gRPC to query timing status information.

1. Let's execute a grpc client in the container exposing the gRPC API.

```console
oc exec -it gm-1-du3-ldc1-tsync-pkxwv -c du3-ldc1-grpc-tsyncd -- tsynctl_grpc 
Tsynctl gRPC Client v1.0.9
$
```

2. Check the status of the clock in the Silicom network card. Locked status is a good sympthon. We need to know to what primary reference time it has been locked to.

```console
$ get_clk_class 
Clock Class: 6, LOCKED
```

3. Check more privileged timing information status related to the clock, PTP, and SyncE.

```console
$ register 1 2 3 4 5
$ get_timing_status 1 2 3 4 5

Timing Status:
==============
Clock Mode:   GM Clock

Clock Status:
=============
Sync Status:    Locked
PTP Lock Status:  Locked
Synce Lock Status:  Locked
Sync Failure Cause: N/A

PTP Data:
=========
Profile:    G_8275_1
GM Clock ID:    00:E0:ED:FF:FE:F0:28:EC
Parent Clock ID:  00:E0:ED:FF:FE:F0:28:EC
Configured Clock Class: 248
Received Clock Class: 6
PTP Interface:    according to T-GM series Port Bit Mask value in tsyncd.conf file

SyncE Data:
===========
SyncE Interface:  according to T-GM series SyncE Port Bit Mask value in tsyncd.conf file
Clock Quality:    4

GNSS Data:
==========
Number of satellites: 31
GNSS Fix Type:    5
GNSS Fix Validity:  true
GNSS Latitude:    32.943067
GNSS Longitude:   -96.994507
GNSS Height:    143.924000
```


## Uninstalling Silicom Timesync Operator <a name="uninstalling"></a>

Show the user helpful output from the pods running on the node, log output from successful assocation with GPS coordinates, etc

### Uninstall from the embedded OperatorHub

<!-- Uninstall steps clearly defined on a FRESH CLUSTER with output-->


### Uninstall from the CLI
<!-- Omit this part for the blogpost -->
* Deleting the subscription and the csv does not delete nfd daemonset or the specialresource daemonsets or the silicom sts-plugin daemonset will not delete the CRs associated to the operator

* If we want to fully delete the set of elements created by the operator we need to delete the stsoperatorconfig CR. The action below will delete the stsoperatorconfig daemonset (i.e., the sts-plugin) and the nfd and sro deploment (if used). 
      

## Wrap-up <a name="stsconfig"></a>

## References <a name="refs"></a>

[1]: https://docs.openshift.com/container-platform/4.8/operators/understanding/olm/olm-understanding-operatorgroups.html#olm-operatorgroups-target-namespace_olm-understanding-operatorgroups