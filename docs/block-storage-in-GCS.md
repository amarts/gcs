## Block Storage in GCS

### References:

* [Document from gluster-k8s](https://github.com/gluster/gluster-kubernetes/blob/master/docs/design/gluster-block-provisioning.md)
* [Document PR on glusterd2](https://github.com/gluster/glusterd2/blob/4780a01fa48127b5100e46974f9f32d1217a8682/doc/design/block-volumes-integration.md)

## Components Involved

The components involved in the overall flow from kubernetes through
to gluster for this new functionality are:

* A new CSI driver for `glusterblock`, to be exposed via a `StorageClass`.
* Gluster tool `gluster-block`, consisting of daemon component and a
  command line tool. This tool is responsible for creating the
  loopback files in a gluster volume and exporting them via iSCSI.
* A stand-alone process to provide REST APIs for using gluster-block CLI.


## Overall Design and Flow

The high-level flow for requesting and using a gluster-block based volume
is as follows.

* Administrator creates a `StorageClass` referring to the `glusterblock` CSI.
* User requests a new `RWO` volume with a `PersistentVolumeClaim` (PVC).
* The `glusterblock` CSI is invoked.
* The CSI calls out to `g-b-rest` REST process, which creates block volume
  with the requested characteristics like size, etc.
* `g-b-rest` process looks for an appropriate gluster volume in one of the
  suitable gluster clusters and calls out to `gluster-blockd` with the request to
  create a block volume on the specified gluster file volume.
* `gluster-block` creates a file on the gluster file volume and exports this
  as an iSCSI target.
* Upon success the resulting volume info (how to reach ...) is handed back
  all the way to the CSI driver.
* The CSI driver puts this volume information into a `PersistentVolume`
  (PV) and sets it up to use the iSCSI mount plugin for mounting.
* The PV is bound to the original PVC.
* The user references the PVC in an application description.
* When the application pod is brought up on a node, the iSCSI mount plugin
  takes care of initiating the block device, formatting (if required) and
  mounting it.


## Details About The `glusterblock` CSI driver

The glusterblock provisioner is an external provisioner that is running in a
container. It's code is located in the `gluster/block` subdirectory of
<https://github.com/gluster/gluster-csi-drivers>.

The CSI is mainly a simple translation engine that turns PVC requests
into requests for `g-b-rest` via RESTful API, and wraps the resulting
volume information into a PV.

The CSI is configured via a StorageClass which can look like this:

```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: glusterblock
provisioner: gluster.org/glusterblock
parameters:
    resturl: "http://127.0.0.1:8061"
    restuser: "admin"
    secretnamespace: "gcs"
    secretname: "gb-secret"
    hacount: "3"
    clusterid: "630372ccdc720a92c681fb928f27b53f"
```

The available parameters are:

* `resturl`: how to reach `g-b-rest` process
* `hacount` (optional): How many paths to the target server to configure.
* `clusterid` (optional): List of one or more clusters to consider for
  finding space for the requested volume.

See  <https://github.com/kubernetes-incubator/external-storage/blob/master/gluster/block/README.md> for
additional and up-to-date details.


## Details About ```gluster-block```

```gluster-block``` is the gluster-level tool to make creation
and consumption of block volumes very easy. It consists of
a server component ```gluster-blockd``` that runs on the gluster
storage nodes and a command line client utility ```gluster-block```
that talks to the ```gluster-blockd``` with local RPC mechanism
and can be invoked on any of the gluster storage nodes.

gluster-block takes care of creating loopback files on the
specified gluster volume. These volumes are then exported
as iSCSI targets with the help of the tcmu-runner mechanism,
using the gluster backend with libgfapi. This has the big
advantage that it is talking to the gluster volume directly
in user space without the need of a glusterfs fuse mount,
skipping the kernel/userspace context switches and the
user-visible mount altogether.

The supported operations are:

* create
* list
* info
* delete
* modify

Details about the gluster-block architecture can be found
in the gluster-block git repository <https://github.com/gluster/gluster-block>
and the original design notes
<https://docs.google.com/document/d/1psjLlCxdllq1IcJa3FN3MFLcQfXYf0L5amVkZMXd-D8/edit?usp=sharing>.


