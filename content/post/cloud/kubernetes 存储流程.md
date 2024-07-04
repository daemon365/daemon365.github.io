---
title: "kubernetes 存储流程"
date: "2024-05-03T00:00:00+08:00"
tags: 
- csi
- kubernetes
showToc: true
---

## PV 与 PVC

PVC (PersistentVolumeClaim)，命名空间（namespace）级别的资源，由 用户 or StatefulSet 控制器（根据VolumeClaimTemplate） 创建。PVC 类似于 Pod，Pod 消耗 Node 资源，PVC 消耗 PV 资源。Pod 可以请求特定级别的资源（CPU 和内存），而 PVC 可以请求特定存储卷的大小及访问模式（Access Mode
PV（PersistentVolume）是集群中的一块存储资源，可以是 NFS、iSCSI、Ceph、GlusterFS 等存储卷，PV 由集群管理员创建，然后由开发者使用 PVC 来申请 PV，PVC 是对 PV 的申请，类似于 Pod 对 Node 的申请。

### 静态创建存储卷

![](/images/e1e2ee82-b5ea-4edb-a587-b1d5860f81d9.png)

也就是我们手动创建一个pv和pvc，然后将pv和pvc绑定，然后pod使用pvc，这样就可以使用pv了。

创建一个 nfs 的 pv 以及 对应的 pvc

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  nfs:
    server: 192.168.203.110
    path: /data/nfs
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

查看 pvc

```BASH
$ kubectl get pvc
NAME      STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
nfs-pvc   Bound    nfs-pv   10Gi       RWO                           101s
```

创建一个 pod 使用 pvc

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-nfs
spec:
  containers:
  - image: ubuntu:22.04
    name: ubuntu
    command:
    - /bin/sh
    - -c
    - sleep 10000
    volumeMounts:
    - mountPath: /data
      name: nfs-volume
  volumes:
  - name: nfs-volume
    persistentVolumeClaim:
      claimName: nfs-pvc
```

```bash
❯ kubectl exec -it test-nfs -- cat /data/nfs
192.168.203.110:/data/nfs
```

### pvc pv 绑定流程

```GO
func (ctrl *PersistentVolumeController) syncUnboundClaim(ctx context.Context, claim *v1.PersistentVolumeClaim) error {
	logger := klog.FromContext(ctx)
	if claim.Spec.VolumeName == "" {
		// 是不是延迟绑定 也就是 VolumeBindingMode 为 WaitForFirstConsumer
		delayBinding, err := storagehelpers.IsDelayBindingMode(claim, ctrl.classLister)
		if err != nil {
			return err
		}
    // 通过 pvc 找到最合适的 pv
		volume, err := ctrl.volumes.findBestMatchForClaim(claim, delayBinding)
		if err != nil {
			logger.V(2).Info("Synchronizing unbound PersistentVolumeClaim, Error finding PV for claim", "PVC", klog.KObj(claim), "err", err)
			return fmt.Errorf("error finding PV for claim %q: %w", claimToClaimKey(claim), err)
		}
		if volume == nil {
			//// No PV found for this claim
		} else /* pv != nil */ {
			
			claimKey := claimToClaimKey(claim)
			logger.V(4).Info("Synchronizing unbound PersistentVolumeClaim, volume found", "PVC", klog.KObj(claim), "volumeName", volume.Name, "volumeStatus", getVolumeStatusForLogging(volume))
      // 绑定 pv 和 pvc
      // 这里会处理 pvc 的 spec.volumeName status 和 pv 的 status
			if err = ctrl.bind(ctx, volume, claim); err != nil {
				return err
			}
			return nil
		}
	} else /* pvc.Spec.VolumeName != nil */ {
		/*
      ......
    */
	}
}

// 选择
func FindMatchingVolume(
	claim *v1.PersistentVolumeClaim,
	volumes []*v1.PersistentVolume,
	node *v1.Node,
	excludedVolumes map[string]*v1.PersistentVolume,
	delayBinding bool) (*v1.PersistentVolume, error) {

	var smallestVolume *v1.PersistentVolume
	var smallestVolumeQty resource.Quantity
	requestedQty := claim.Spec.Resources.Requests[v1.ResourceName(v1.ResourceStorage)]
	requestedClass := GetPersistentVolumeClaimClass(claim)

	var selector labels.Selector
	if claim.Spec.Selector != nil {
		internalSelector, err := metav1.LabelSelectorAsSelector(claim.Spec.Selector)
		if err != nil {
			return nil, fmt.Errorf("error creating internal label selector for claim: %v: %v", claimToClaimKey(claim), err)
		}
		selector = internalSelector
	}

	// Go through all available volumes with two goals:
	// - find a volume that is either pre-bound by user or dynamically
	//   provisioned for this claim. Because of this we need to loop through
	//   all volumes.
	// - find the smallest matching one if there is no volume pre-bound to
	//   the claim.
	for _, volume := range volumes {
		if _, ok := excludedVolumes[volume.Name]; ok {
			// Skip volumes in the excluded list
			continue
		}
		if volume.Spec.ClaimRef != nil && !IsVolumeBoundToClaim(volume, claim) {
			continue
		}

		volumeQty := volume.Spec.Capacity[v1.ResourceStorage]
		if volumeQty.Cmp(requestedQty) < 0 {
			continue
		}
		// filter out mismatching volumeModes
		if CheckVolumeModeMismatches(&claim.Spec, &volume.Spec) {
			continue
		}

		// check if PV's DeletionTimeStamp is set, if so, skip this volume.
		if volume.ObjectMeta.DeletionTimestamp != nil {
			continue
		}

		nodeAffinityValid := true
		if node != nil {
			// Scheduler path, check that the PV NodeAffinity
			// is satisfied by the node
			// CheckNodeAffinity is the most expensive call in this loop.
			// We should check cheaper conditions first or consider optimizing this function.
			err := CheckNodeAffinity(volume, node.Labels)
			if err != nil {
				nodeAffinityValid = false
			}
		}

		if IsVolumeBoundToClaim(volume, claim) {
			// If PV node affinity is invalid, return no match.
			// This means the prebound PV (and therefore PVC)
			// is not suitable for this node.
			if !nodeAffinityValid {
				return nil, nil
			}

			return volume, nil
		}

		if node == nil && delayBinding {
			// PV controller does not bind this claim.
			// Scheduler will handle binding unbound volumes
			// Scheduler path will have node != nil
			continue
		}

		// filter out:
		// - volumes in non-available phase
		// - volumes whose labels don't match the claim's selector, if specified
		// - volumes in Class that is not requested
		// - volumes whose NodeAffinity does not match the node
		if volume.Status.Phase != v1.VolumeAvailable {
			// We ignore volumes in non-available phase, because volumes that
			// satisfies matching criteria will be updated to available, binding
			// them now has high chance of encountering unnecessary failures
			// due to API conflicts.
			continue
		} else if selector != nil && !selector.Matches(labels.Set(volume.Labels)) {
			continue
		}
		if GetPersistentVolumeClass(volume) != requestedClass {
			continue
		}
		if !nodeAffinityValid {
			continue
		}

		if node != nil {
			// Scheduler path
			// Check that the access modes match
			if !CheckAccessModes(claim, volume) {
				continue
			}
		}

		if smallestVolume == nil || smallestVolumeQty.Cmp(volumeQty) > 0 {
			smallestVolume = volume
			smallestVolumeQty = volumeQty
		}
	}

	if smallestVolume != nil {
		// Found a matching volume
		return smallestVolume, nil
	}

	return nil, nil
}

```


### kubelet 绑定

```go
if err := os.MkdirAll(dir, 0750); err != nil {
		return err
}
source := fmt.Sprintf("%s:%s", nfsMounter.server, nfsMounter.exportPath)
options := []string{}
if nfsMounter.readOnly {
  options = append(options, "ro")
}
mountOptions := util.JoinMountOptions(nfsMounter.mountOptions, options)
err = nfsMounter.mounter.MountSensitiveWithoutSystemd(source, dir, "nfs", mountOptions, nil)
```

kubelet 就会在调用 `sudo mount -t nfs ...` 命令把 nfs 绑定到主机上 绑定的目录大概为 `/var/lib/kubelet/pods/[POD-ID]/volumes/` 

## StorageClass

StorageClass 是 Kubernetes 中用来定义存储卷的类型的资源对象，StorageClass 用来定义存储卷的类型，比如 NFS、iSCSI、Ceph、GlusterFS 等存储卷。StorageClass 是集群级别的资源，由集群管理员创建，用户可以使用 StorageClass 来动态创建 PV。

### 动态创建存储卷

动态创建存储卷相比静态创建存储卷，少了集群管理员的干预，流程如下图所示：

![](/images/7cc8ddd7-1c7d-430d-9332-d7da9d4a2752.png)

创建一个 StorageClass pvc pod

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: rancher.io/local-path
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-local-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 128Mi
  storageClassName: local-storage
---
apiVersion: v1
kind: Pod
metadata:
  name: test
spec:
  containers:
  - image: ubuntu:22.04
    name: ubuntu
    command:
    - /bin/sh
    - -c
    - sleep 10000
    volumeMounts:
    - mountPath: /data
      name: my-local-pvc
  volumes:
  - name: my-local-pvc
    persistentVolumeClaim:
      claimName: my-local-pvc
```

查看 pv

```bash
❯ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                  STORAGECLASS    REASON   AGE
pvc-9d257d8a-29a8-4abf-a1e2-c7e4953fc0ca   128Mi      RWO            Delete           Bound    default/my-local-pvc   local-storage            85s
```

### StorageClass 创建 pv 流程

```go
// 还是 syncUnboundClaim 中
// volume 为空，说明没有找到合适的 pv 那么去检查 如果 pvc 的 storageClassName 不为空，那么就会去找到对应的 storageClass
if volume == nil {

			switch {
			case delayBinding && !storagehelpers.IsDelayBindingProvisioning(claim):
        // ......
			case storagehelpers.GetPersistentVolumeClaimClass(claim) != "":
				// 如果 pvc 的 storageClassName 不为空，那么就会去找到对应的 storageClass
				if err = ctrl.provisionClaim(ctx, claim); err != nil {
					return err
				}
				return nil
			default:
			}
			return nil
}

func (ctrl *PersistentVolumeController) provisionClaim(ctx context.Context, claim *v1.PersistentVolumeClaim) error {
	plugin, storageClass, err := ctrl.findProvisionablePlugin(claim)
	ctrl.scheduleOperation(logger, opName, func() error {
		var err error
		if plugin == nil {
      // 如果是外部的 provisioner 这里我们就安装了 rancher.io/local-path 这个插件
      // 所以这里会调用 provisionClaimOperationExternal
			_, err = ctrl.provisionClaimOperationExternal(ctx, claim, storageClass)
		} else {
      // 内部的 provisioner 直接处理
			_, err = ctrl.provisionClaimOperation(ctx, claim, plugin, storageClass)
		}
		return err
	})
	return nil
}

// 如果是外部的 provisioner 会在 pvc 的 annotations 加入 volume.beta.kubernetes.io/storage-provisioner: rancher.io/local-path 和 volume.kubernetes.io/storage-provisioner: rancher.io/local-path
func (ctrl *PersistentVolumeController) setClaimProvisioner(ctx context.Context, claim *v1.PersistentVolumeClaim, provisionerName string) (*v1.PersistentVolumeClaim, error) {
	if val, ok := claim.Annotations[storagehelpers.AnnStorageProvisioner]; ok && val == provisionerName {
		// annotation is already set, nothing to do
		return claim, nil
	}

	// The volume from method args can be pointing to watcher cache. We must not
	// modify these, therefore create a copy.
	claimClone := claim.DeepCopy()
	// TODO: remove the beta storage provisioner anno after the deprecation period
	logger := klog.FromContext(ctx)
	metav1.SetMetaDataAnnotation(&claimClone.ObjectMeta, storagehelpers.AnnBetaStorageProvisioner, provisionerName)
	metav1.SetMetaDataAnnotation(&claimClone.ObjectMeta, storagehelpers.AnnStorageProvisioner, provisionerName)
	updateMigrationAnnotations(logger, ctrl.csiMigratedPluginManager, ctrl.translator, claimClone.Annotations, true)
	newClaim, err := ctrl.kubeClient.CoreV1().PersistentVolumeClaims(claim.Namespace).Update(ctx, claimClone, metav1.UpdateOptions{})
	if err != nil {
		return newClaim, err
	}
	_, err = ctrl.storeClaimUpdate(logger, newClaim)
	if err != nil {
		return newClaim, err
	}
	return newClaim, nil
}
```

## kubernetes external provisioner

kubernetes external provisioner 是一个独立的进程，用来动态创建 PV，它通过监听 StorageClass 的事件，当 StorageClass 的 ReclaimPolicy 为 Retain 时，会创建 PV。


在这里我新建一个 关于 nfs 的 external provisioner

```go
package main

import (
	"context"
	"fmt"
	"path/filepath"

	"github.com/golang/glog"
	v1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/client-go/util/homedir"
	"sigs.k8s.io/controller-runtime/pkg/log"
	"sigs.k8s.io/sig-storage-lib-external-provisioner/v10/controller"
)

const provisionerName = "provisioner.test.com/nfs"

var _ controller.Provisioner = &nfsProvisioner{}

type nfsProvisioner struct {
	client kubernetes.Interface
}

func (p *nfsProvisioner) Provision(ctx context.Context, options controller.ProvisionOptions) (*v1.PersistentVolume, controller.ProvisioningState, error) {
	if options.PVC.Spec.Selector != nil {
		return nil, controller.ProvisioningFinished, fmt.Errorf("claim Selector is not supported")
	}
	glog.V(4).Infof("nfs provisioner: VolumeOptions %v", options)

	pv := &v1.PersistentVolume{
		ObjectMeta: metav1.ObjectMeta{
			Name: options.PVName,
		},
		Spec: v1.PersistentVolumeSpec{
			PersistentVolumeReclaimPolicy: *options.StorageClass.ReclaimPolicy,
			AccessModes:                   options.PVC.Spec.AccessModes,
			MountOptions:                  options.StorageClass.MountOptions,
			Capacity: v1.ResourceList{
				v1.ResourceName(v1.ResourceStorage): options.PVC.Spec.Resources.Requests[v1.ResourceName(v1.ResourceStorage)],
			},
			PersistentVolumeSource: v1.PersistentVolumeSource{
				NFS: &v1.NFSVolumeSource{
					Server:   options.StorageClass.Parameters["server"],
					Path:     options.StorageClass.Parameters["path"],
					ReadOnly: options.StorageClass.Parameters["readOnly"] == "true",
				},
			},
		},
	}

	return pv, controller.ProvisioningFinished, nil
}

func (p *nfsProvisioner) Delete(ctx context.Context, volume *v1.PersistentVolume) error {
	// 因为是 nfs 没有产生实际的资源，所以这里不需要删除
	// 如果在 provisioner 中创建了资源，那么这里需要删除
	// 一般是调用 csi 创建/删除资源
	return nil
}

func main() {
	l := log.FromContext(context.Background())
	config, err := clientcmd.BuildConfigFromFlags("", filepath.Join(homedir.HomeDir(), ".kube", "config"))
	if err != nil {
		glog.Fatalf("Failed to create kubeconfig: %v", err)
	}
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		glog.Fatalf("Failed to create client: %v", err)
	}

	clientNFSProvisioner := &nfsProvisioner{
		client: clientset,
	}

	pc := controller.NewProvisionController(l,
		clientset,
		provisionerName,
		clientNFSProvisioner,
		controller.LeaderElection(true),
	)
	glog.Info("Starting provision controller")
	pc.Run(context.Background())
}

```

创建一个 nfs 的 storageClass 

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: my-nfs
provisioner: provisioner.test.com/nfs
reclaimPolicy: Delete
volumeBindingMode: Immediate
parameters:
  server: "192.168.203.110"
  path: /data/nfs
  readOnly: "false"
```

```YAML
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-nfs-pvc
spec:
  storageClassName: my-nfs
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-nfs
spec:
  containers:
  - image: ubuntu:22.04
    name: ubuntu
    command:
    - /bin/sh
    - -c
    - sleep 10000
    volumeMounts:
    - mountPath: /data
      name: my-nfs-pvc
  volumes:
  - name: my-nfs-pvc
    persistentVolumeClaim:
      claimName: my-nfs-pvc
```

```bash
❯ kubectl exec -it test-nfs -- cat /data/nfs
192.168.203.110:/data/nfs
```

## CSI 流程

持久化存储流程图如下：

![](/images/d0dd40b9-6ce3-40bc-a048-0b3f1653d952.jpg)

### Provisioner

当部署 csi-controller 时 ，会启动一个伴生容器，项目地址为 `https://github.com/kubernetes-csi/external-provisioner`  这个项目是一个 csi 的 provisioner
它会监控属于自己的pvc，当有新的pvc创建时，会调用 csi 的 createVolume 方法，创建一个 volume，然后创建一个 pv。当 pvc 删除时，会调用 csi 的 deleteVolume 方法，然后删除 volume 和 pv。

![](/images/8556b88f-a2a3-447a-a2ac-e0ed179e3b24.png)

![](/images/06af3f24-bdbc-44be-a08a-36f1da8ab8c5.png)

### Attacher

external-attacher 也是 csi-controller 的伴生容器，项目地址为 `https://github.com/kubernetes-csi/external-attacher` 这个项目是一个 csi 的 attacher, 它会监控 AttachDetachController 资源，当有新的资源创建时，会调用 csi 的 controllerPublishVolume 方法，挂载 volume 到 node 上。当资源删除时，会调用 csi 的 controllerUnpublishVolume 方法，卸载 volume。

![](/images/d4f2c430-8b28-43e6-b150-960db81b6bf7.png)

![](/images/2486b91a-6a5a-476a-8795-a69305dd6011.png)

### Snapshotter

external-snapshotter 也是 csi-controller 的伴生容器，项目地址为 `https://github.com/kubernetes-csi/external-snapshotter` 这个项目是一个 csi 的 snapshotter, 它会监控 VolumeSnapshot 资源，当有新的资源创建时，会调用 csi 的 createSnapshot 方法，创建一个快照。当资源删除时，会调用 csi 的 deleteSnapshot 方法，删除快照。

### csi-node

csi-node 是一个 kubelet 的插件，所以它需要每个节点上都运行，当 pod 创建时，并且 VolumeAttachment 的 .spec.Attached 时，kubelet 会调用 csi 的 NodeStageVolume 函数，之后插件（csiAttacher）调用内部 in-tree CSI 插件（csiMountMgr）的 SetUp 函数，该函数内部会调用 csi 的 NodePublishVolume 函数，挂载 volume 到 pod 上。当 pod 删除时，kubelet 观察到包含 CSI 存储卷的 Pod 被删除，于是调用内部 in-tree CSI 插件（csiMountMgr）的 TearDown 函数，该函数内部会通过 unix domain socket 调用外部 CSI 插件的 NodeUnpublishVolume 函数。kubelet 调用内部 in-tree CSI 插件（csiAttacher）的 UnmountDevice 函数，该函数内部会通过 unix domain socket 调用外部 CSI 插件的 NodeUnstageVolume 函数。

![](/images/d6bd8e82-8227-4e1e-9764-c66a34572398.png)

![](/images/b8a8b74d-4c97-4dcd-a097-20b59561b107.png)

### csi-node-driver-registrar

这个是 csi-node 的伴生容器，项目地址为 `https://github.com/kubernetes-csi/node-driver-registrar`,
它的主要作用是向 kubelet 注册 csi 插件，kubelet 会调用 csi 插件的 Probe 方法，如果返回成功，kubelet 会调用 csi 插件的 NodeGetInfo 方法，获取节点信息。

### csi-livenessprobe

这个是 csi-node 的伴生容器，项目地址为 `https://github.com/kubernetes-csi/livenessprobe`, 它的主要作用是给 kubernetes 的 livenessprobe 提供一个接口，用来检查 csi 插件是否正常运行。它在 `/healthz` 时，会调用 csi 的 Probe 方法，如果返回成功，返回 200，否则返回 500。

## Reference
- https://developer.aliyun.com/article/754434
- https://developer.aliyun.com/article/783464
