---
title: "kubernetes container device interface (CDI)"
date: "2023-11-19T00:00:00+08:00"
tags: 
- cdi
- kubernetes
showToc: true
---


## CDI 是什么？

Container Device Interface (CDI) 是一个提议的标准，它定义了如何在容器运行时环境中向容器提供设备。这个提议的目的是使得设备供应商能够更容易地将其设备集成到 Kubernetes 集群中，而不必修改 Kubernetes 核心代码。

CDI 插件通常负责：

1. 配置设备以供容器使用（例如，分配设备文件或设置必要的环境变量）。
2. 在容器启动时将设备资源注入到容器中。

**官网**

- https://github.com/cncf-tags/container-device-interface

## 为什么需要CDI？

如果我们想在容器内使用 nvidia 的 gpu，在没有 CDI 之前，我们需要修改 containerd 的 low-level container runtime(runc) 到 nvidia runtime。这么做的原因就是使用 gpu 不单单要绑定 gpu device 文件到容器内，还需要绑定一些驱动文件和可执行命令（比如 nvidia-smi）等到容器内，还有就执行一些 hooks。 nvidia runtime 的作用就是绑定一些文件和执行一些 hooks 然后调用 runc。

现在我们可以使用 CDI 做这些事情，除了无需修改 runtime 外，还有抽象和插件化等优点。

## 版本及准备工作

- kubelet version >= 1.28.0
- containerd version >= 1.7.0

而且这在 k8s 1.28 (1.29版本是 beta 了 默认就打开了) 版本中是一个 alpha 版本的功能，所以我们需要在 kubelet 的启动参数中加入开启特性门：`--feature-gates=DevicePluginCDIDevices =true`。

```bash
sudo vim /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS --feature-gates=DevicePluginCDIDevices=true
```

containerd 也需要开启 CDI cdi_spec_dirs 为 cdi 配置文件的目录，enable_cdi 为开启 CDI 功能。

```bash
sudo vim /etc/containerd/config.toml

cdi_spec_dirs = ["/etc/cdi", "/var/run/cdi"]
enable_cdi = true
```

重启 containerd 和 kubelet

```bash
sudo systemctl restart kubelet.service containerd.service
```


## mock

因为我的集群里没有 gpu , 所以我就随便 mock 几个文件作为 device 了。

```bash
sudo mkdir /dev/mock
cd /dev/mock

sudo mknod /dev/mock/device_0 c 89 1
sudo mknod /dev/mock/device_1 c 89 1
sudo mknod /dev/mock/device_2 c 89 1
sudo mknod /dev/mock/device_3 c 89 1
sudo mknod /dev/mock/device_4 c 89 1
sudo mknod /dev/mock/device_5 c 89 1
sudo mknod /dev/mock/device_6 c 89 1
sudo mknod /dev/mock/device_7 c 89 1
```

```bash
sudo vim /mock/bin/list_device.sh
#!/bin/bash

# 定义目录数组
directories=(/dev /dev/mock)

# 遍历目录数组
for dir in "${directories[@]}"; do
  # 检查目录是否存在
  if [ -d "$dir" ]; then
    # 目录存在，打印目录下的所有文件
    ls  "$dir"
  fi
done

sudo chmod a+x /mock/bin/list_device.sh
```

```bash
sudo mkdir /mock/so
cd /mock/so
sudo touch device_0.so device_1.so device_2.so device_3.so device_5.so device_6.so device_7.so device_4.so
```

## 开启 kubelet 的 device plugin

下面是简单写的一个 device plugin，及其 dockerfile 还有部署到 k8s 的 yaml 文件。

```go
package main

import (
	"context"
	"fmt"
	"time"

	"github.com/kubevirt/device-plugin-manager/pkg/dpm"
	pluginapi "k8s.io/kubelet/pkg/apis/deviceplugin/v1beta1"
)

type PluginLister struct {
	ResUpdateChan chan dpm.PluginNameList
}

var ResourceNamespace = "mock.com"
var PluginName = "gpu"

func (p *PluginLister) GetResourceNamespace() string {
	return ResourceNamespace
}

func (p *PluginLister) Discover(pluginListCh chan dpm.PluginNameList) {
	pluginListCh <- dpm.PluginNameList{PluginName}
}

func (p *PluginLister) NewPlugin(name string) dpm.PluginInterface {
	return &Plugin{}
}

type Plugin struct {
}

func (p *Plugin) GetDevicePluginOptions(ctx context.Context, e *pluginapi.Empty) (*pluginapi.DevicePluginOptions, error) {
	options := &pluginapi.DevicePluginOptions{
		PreStartRequired: true,
	}
	return options, nil
}

func (p *Plugin) PreStartContainer(ctx context.Context, r *pluginapi.PreStartContainerRequest) (*pluginapi.PreStartContainerResponse, error) {
	return &pluginapi.PreStartContainerResponse{}, nil
}

func (p *Plugin) GetPreferredAllocation(ctx context.Context, r *pluginapi.PreferredAllocationRequest) (*pluginapi.PreferredAllocationResponse, error) {
	return &pluginapi.PreferredAllocationResponse{}, nil
}

func (p *Plugin) ListAndWatch(e *pluginapi.Empty, r pluginapi.DevicePlugin_ListAndWatchServer) error {
	devices := []*pluginapi.Device{}
	for i := 0; i < 8; i++ {
		devices = append(devices, &pluginapi.Device{
			// 和 device 名称保持一致
			ID:     fmt.Sprintf("device_%d", i),
			Health: pluginapi.Healthy,
		})
	}
	for {
		fmt.Println("register devices")
		// 每分钟注册一次
		r.Send(&pluginapi.ListAndWatchResponse{
			Devices: devices,
		})
		time.Sleep(time.Second * 60)
	}
}

func (p *Plugin) Allocate(ctx context.Context, r *pluginapi.AllocateRequest) (*pluginapi.AllocateResponse, error) {
	// 使用cdi插件
	responses := &pluginapi.AllocateResponse{}
	for _, req := range r.ContainerRequests {
		cdidevices := []*pluginapi.CDIDevice{}
		for _, id := range req.DevicesIDs {
			cdidevices = append(cdidevices, &pluginapi.CDIDevice{
				Name: fmt.Sprintf("%s/%s=%s", ResourceNamespace, PluginName, id),
			})
		}
		responses.ContainerResponses = append(responses.ContainerResponses, &pluginapi.ContainerAllocateResponse{
			CDIDevices: cdidevices,
		})
	}
	return responses, nil
}

func main() {
	m := dpm.NewManager(&PluginLister{})
	m.Run()
}
```

```Dockerfile
FROM golang:1.21.3 as builder
  
COPY . /src
WORKDIR /src
RUN go env -w GO111MODULE=on && go env -w GOPROXY=https://goproxy.io,direct
RUN go build
  
FROM debian:bookworm-slim
  
RUN sed -i 's/deb.debian.org/mirrors.ustc.edu.cn/g' /etc/apt/sources.list.d/debian.sources
  
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates   \
      netbase \
      pciutils \
      curl \
      && rm -rf /var/lib/apt/lists/ \
      && apt-get autoremove -y && apt-get autoclean -y
  
RUN update-pciids
  
COPY --from=builder /src /app
  
WORKDIR /app
```

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: mock-plugin
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: mock-plugin-daemonset
  namespace: mock-plugin
spec:
  selector:
    matchLabels:
      name: mock-plugin
  template:
    metadata:
      labels:
        name: mock-plugin
        app.kubernetes.io/component: mock-plugin
        app.kubernetes.io/name: mock-plugin
    spec:
      containers:
      - image: zhaohaiyu/mock:v1
        name: mock-plugin
        command: ['/app/mock']
        imagePullPolicy: Always
        securityContext:
          privileged: true
        tty: true
        volumeMounts:
        - name: kubelet
          mountPath: /var/lib/kubelet
      volumes:
      - name: kubelet
        hostPath:
          path: /var/lib/kubelet
```


执行完毕使用 kubectl 查看

```bash
❯ kubectl -n mock-plugin get pod
NAME                          READY   STATUS    RESTARTS   AGE
mock-plugin-daemonset-8w2r8   1/1     Running   0          3m27s
```

查看 node 是否已经注册了 device plugin

```bash
kubectl describe node node1

Capacity:
  cpu:                8
  ephemeral-storage:  102626232Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             24570324Ki
  mock.com/gpu:       8
  pods:               110
Allocatable:
  cpu:                8
  ephemeral-storage:  94580335255
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             24467924Ki
  mock.com/gpu:       8
  pods:               110
```

可以看到已经注册了 8 个 gpu 设备 名字叫 mock.com/gpu 也就是我们代码中定义的。

## CDI配置文件

CDI Spec: https://github.com/cncf-tags/container-device-interface/blob/main/SPEC.md

我们也生成了一个 CDI 的配置文件，这个配置文件会被 containerd 读取，然后根据配置文件中的信息去调用 device plugin。


```yaml
# vim /etc/cdi/mock.yaml
cdiVersion: 0.5.0
kind: mock.com/gpu
devices:

- name: device_0
  containerEdits:
    deviceNodes:
    - hostPath: "/dev/mock/device_0"
      path: "/dev/mock/device_0"
      type: c
      permissions: rw
    mounts:
    - hostPath: "/mock/so/device_0.so"
      containerPath: "/mock/so/device_0.so"
      options:
      - ro
      - nosuid
      - nodev
      - bind

- name: device_1
  containerEdits:
    deviceNodes:
    - hostPath: "/dev/mock/device_1"
      path: "/dev/mock/device_1"
      type: c
      permissions: rw
    mounts:
    - hostPath: "/mock/so/device_1.so"
      containerPath: "/mock/so/device_1.so"
      options:
      - ro
      - nosuid
      - nodev
      - bind

- name: device_2
  containerEdits:
    deviceNodes:
    - hostPath: "/dev/mock/device_2"
      path: "/dev/mock/device_2"
      type: c
      permissions: rw
    mounts:
    - hostPath: "/mock/so/device_2.so"
      containerPath: "/mock/so/device_2.so"
      options:
      - ro
      - nosuid
      - nodev
      - bind

- name: device_3
  containerEdits:
    deviceNodes:
    - hostPath: "/dev/mock/device_3"
      path: "/dev/mock/device_3"
      type: c
      permissions: rw
    mounts:
    - hostPath: "/mock/so/device_3.so"
      containerPath: "/mock/so/device_3.so"
      options:
      - ro
      - nosuid
      - nodev
      - bind

- name: device_4
  containerEdits:
    deviceNodes:
    - hostPath: "/dev/mock/device_4"
      path: "/dev/mock/device_4"
      type: c
      permissions: rw
    mounts:
    - hostPath: "/mock/so/device_4.so"
      containerPath: "/mock/so/device_4.so"
      options:
      - ro
      - nosuid
      - nodev
      - bind

- name: device_5
  containerEdits:
    deviceNodes:
    - hostPath: "/dev/mock/device_5"
      path: "/dev/mock/device_5"
      type: c
      permissions: rw
    mounts:
    - hostPath: "/mock/so/device_5.so"
      containerPath: "/mock/so/device_5.so"
      options:
      - ro
      - nosuid
      - nodev
      - bind

- name: device_6
  containerEdits:
    deviceNodes:
    - hostPath: "/dev/mock/device_6"
      path: "/dev/mock/device_6"
      type: c
      permissions: rw
    mounts:
    - hostPath: "/mock/so/device_6.so"
      containerPath: "/mock/so/device_6.so"
      options:
      - ro
      - nosuid
      - nodev
      - bind

- name: device_7
  containerEdits:
    deviceNodes:
    - hostPath: "/dev/mock/device_7"
      path: "/dev/mock/device_7"
      type: c
      permissions: rw
    mounts:
    - hostPath: "/mock/so/device_7.so"
      containerPath: "/mock/so/device_7.so"
      options:
      - ro
      - nosuid
      - nodev
      - bind


containerEdits:
  mounts:
  - hostPath: "/mock/bin/list_device.sh"
    containerPath: "/usr/local/bin/list_device.sh"
    options:
    - ro
    - nosuid
    - nodev
    - bind
```

这里我只是简单示例，还有 hooks 和 env 等用法查看官方文档。


## 部署 pod

我部署一个 pod 使用 mock gpu 这个资源 4个。

```YAML
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu1
spec:
  containers:
  - name: ubuntu-container
    image: ubuntu:latest
    command: ["sleep"]
    args: ["3600"]
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
        mock.com/gpu: "4"
      limits:
        memory: "128Mi"
        cpu: "500m"
        mock.com/gpu: "4"
```

```bash
ubuntu1   1/1     Running   0          49s
```

现在我们 使用 `kubectl exec -it ubuntu1 bash` 进入容器看一看。

```bash
ls /dev/mock/
device_0  device_1  device_6  device_7

ls /mock/so/
device_0.so  device_1.so  device_6.so  device_7.so

list_device.sh
so
device_0  device_1  device_6  device_7
```

可以看到我们 cdi 配置文件配置的 device 和 so 文件和还有我们的 list_device.sh 都已经挂载到容器内了。

我现在再启动一个 pod 使用 mock gpu 这个资源 3 个。

```YAML
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu2
spec:
  containers:
  - name: ubuntu-container
    image: ubuntu:latest
    command: ["sleep"]
    args: ["3600"]
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
        mock.com/gpu: "3"
      limits:
        memory: "128Mi"
        cpu: "500m"
        mock.com/gpu: "3"
```

```bash
ls /dev/mock/
device_2  device_3  device_5
```

查看node使用了多少资源
```
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests     Limits
  --------           --------     ------
  cpu                1550m (19%)  1 (12%)
  memory             668Mi (2%)   596Mi (2%)
  ephemeral-storage  0 (0%)       0 (0%)
  hugepages-1Gi      0 (0%)       0 (0%)
  hugepages-2Mi      0 (0%)       0 (0%)
  mock.com/gpu       7            7
```

可以看到我们使用了 7 个 mock.com/gpu 资源，还剩下 1 个。


## nvdia gpu

我找了一台带有 nvidia gpu 的机器，然后安装了 nvidia-container-toolkit-base。

使用 `nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml` 生产 cdi 配置文件。

```yaml
---
cdiVersion: 0.5.0
containerEdits:
  deviceNodes:
  - path: /dev/nvidia-modeset
  - path: /dev/nvidia-uvm
  - path: /dev/nvidia-uvm-tools
  - path: /dev/nvidiactl
  hooks:
  - args:
    - nvidia-ctk
    - hook
    - create-symlinks
    - --link
    - libglxserver_nvidia.so.525.147.05::/usr/lib/x86_64-linux-gnu/nvidia/xorg/libglxserver_nvidia.so
    hookName: createContainer
    path: /usr/bin/nvidia-ctk
  - args:
    - nvidia-ctk
    - hook
    - update-ldcache
    - --folder
    - /usr/lib/x86_64-linux-gnu
    hookName: createContainer
    path: /usr/bin/nvidia-ctk
  mounts:
  - containerPath: /run/nvidia-persistenced/socket
    hostPath: /run/nvidia-persistenced/socket
    options:
    - ro
    - nosuid
    - nodev
    - bind
    - noexec
  - containerPath: /usr/bin/nvidia-cuda-mps-control
    hostPath: /usr/bin/nvidia-cuda-mps-control
    options:
    - ro
    - nosuid
    - nodev
    - bind
  - containerPath: /usr/bin/nvidia-cuda-mps-server
    hostPath: /usr/bin/nvidia-cuda-mps-server
    options:
    - ro
    - nosuid
    - nodev
    - bind
  - containerPath: /usr/bin/nvidia-debugdump
    hostPath: /usr/bin/nvidia-debugdump
    options:
    - ro
    - nosuid
    - nodev
    - bind
  - containerPath: /usr/bin/nvidia-persistenced
    hostPath: /usr/bin/nvidia-persistenced
    options:
    - ro
    - nosuid
    - nodev
    - bind
  - containerPath: /usr/bin/nvidia-smi
    hostPath: /usr/bin/nvidia-smi
    options:
    - ro
    - nosuid
    - nodev
    - bind
  - containerPath: /usr/lib/x86_64-linux-gnu/libEGL_nvidia.so.525.147.05
    hostPath: /usr/lib/x86_64-linux-gnu/libEGL_nvidia.so.525.147.05
    options:
    - ro
    - nosuid
    - nodev
    - bind
  - containerPath: /usr/lib/x86_64-linux-gnu/libGLESv1_CM_nvidia.so.525.147.05
    hostPath: /usr/lib/x86_64-linux-gnu/libGLESv1_CM_nvidia.so.525.147.05
    options:
    - ro
    - nosuid
    - nodev
    - bind
  - containerPath: /usr/lib/x86_64-linux-gnu/libGLESv2_nvidia.so.525.147.05
    hostPath: /usr/lib/x86_64-linux-gnu/libGLESv2_nvidia.so.525.147.05
    options:
    - ro
    - nosuid
    - nodev
    - bind
  - containerPath: /usr/lib/x86_64-linux-gnu/libGLX_nvidia.so.525.147.05
    hostPath: /usr/lib/x86_64-linux-gnu/libGLX_nvidia.so.525.147.05
    options:
    - ro
    - nosuid
    - nodev
    - bind
  - containerPath: /usr/lib/x86_64-linux-gnu/libcuda.so.525.147.05
    hostPath: /usr/lib/x86_64-linux-gnu/libcuda.so.525.147.05
    options:
    - ro
    - nosuid
    - nodev
    - bind
  - containerPath: /usr/lib/x86_64-linux-gnu/libcudadebugger.so.525.147.05
    hostPath: /usr/lib/x86_64-linux-gnu/libcudadebugger.so.525.147.05
    options:
    - ro
    - nosuid
    - nodev
    - bind
  - containerPath: /usr/lib/x86_64-linux-gnu/libnvcuvid.so.525.147.05
    hostPath: /usr/lib/x86_64-linux-gnu/libnvcuvid.so.525.147.05
    options:
    - ro
    - nosuid
    - nodev
    - bind
  - containerPath: /usr/lib/x86_64-linux-gnu/libnvidia-allocator.so.525.147.05
    hostPath: /usr/lib/x86_64-linux-gnu/libnvidia-allocator.so.525.147.05
    options:
    - ro
    - nosuid
    - nodev
    - bind
  - containerPath: /usr/lib/x86_64-linux-gnu/libnvidia-cfg.so.525.147.05
    hostPath: /usr/lib/x86_64-linux-gnu/libnvidia-cfg.so.525.147.05
    options:
    - ro
    - nosuid
    - nodev
    - bind
  - containerPath: /usr/lib/x86_64-linux-gnu/libnvidia-compiler.so.525.147.05
    hostPath: /usr/lib/x86_64-linux-gnu/libnvidia-compiler.so.525.147.05
    options:
    - ro
    - nosuid
    - nodev
    - bind
  - containerPath: /usr/lib/x86_64-linux-gnu/libnvidia-egl-gbm.so.1.1.0
    hostPath: /usr/lib/x86_64-linux-gnu/libnvidia-egl-gbm.so.1.1.0
    options:
    - ro
    - nosuid
    - nodev
    - bind
  - containerPath: /usr/lib/x86_64-linux-gnu/libnvidia-eglcore.so.525.147.05
    hostPath: /usr/lib/x86_64-linux-gnu/libnvidia-eglcore.so.525.147.05
    options:
    - ro
    - nosuid
    - nodev
    - bind
  - containerPath: /usr/lib/x86_64-linux-gnu/libnvidia-encode.so.525.147.05
    hostPath: /usr/lib/x86_64-linux-gnu/libnvidia-encode.so.525.147.05
    options:
    - ro
    - nosuid
    - nodev
    - bind
  - containerPath: /usr/lib/x86_64-linux-gnu/libnvidia-fbc.so.525.147.05
    hostPath: /usr/lib/x86_64-linux-gnu/libnvidia-fbc.so.525.147.05
    options:
    - ro
    - nosuid
    - nodev
    - bind
  - containerPath: /usr/lib/x86_64-linux-gnu/libnvidia-glcore.so.525.147.05
    hostPath: /usr/lib/x86_64-linux-gnu/libnvidia-glcore.so.525.147.05
    options:
    - ro
    - nosuid
    - nodev
    - bind
  - containerPath: /usr/lib/x86_64-linux-gnu/libnvidia-glsi.so.525.147.05
    hostPath: /usr/lib/x86_64-linux-gnu/libnvidia-glsi.so.525.147.05
    options:
    - ro
    - nosuid
    - nodev
    - bind
  - containerPath: /usr/lib/x86_64-linux-gnu/libnvidia-glvkspirv.so.525.147.05
    hostPath: /usr/lib/x86_64-linux-gnu/libnvidia-glvkspirv.so.525.147.05
    options:
    - ro
    - nosuid
    - nodev
    - bind
  - containerPath: /usr/lib/x86_64-linux-gnu/libnvidia-ml.so.525.147.05
    hostPath: /usr/lib/x86_64-linux-gnu/libnvidia-ml.so.525.147.05
    options:
    - ro
    - nosuid
    - nodev
    - bind
  - containerPath: /usr/lib/x86_64-linux-gnu/libnvidia-ngx.so.525.147.05
    hostPath: /usr/lib/x86_64-linux-gnu/libnvidia-ngx.so.525.147.05
    options:
    - ro
    - nosuid
    - nodev
    - bind
  - containerPath: /usr/lib/x86_64-linux-gnu/libnvidia-nvvm.so.525.147.05
    hostPath: /usr/lib/x86_64-linux-gnu/libnvidia-nvvm.so.525.147.05
    options:
    - ro
    - nosuid
    - nodev
    - bind
  - containerPath: /usr/lib/x86_64-linux-gnu/libnvidia-opencl.so.525.147.05
    hostPath: /usr/lib/x86_64-linux-gnu/libnvidia-opencl.so.525.147.05
    options:
    - ro
    - nosuid
    - nodev
    - bind
  - containerPath: /usr/lib/x86_64-linux-gnu/libnvidia-opticalflow.so.525.147.05
    hostPath: /usr/lib/x86_64-linux-gnu/libnvidia-opticalflow.so.525.147.05
    options:
    - ro
    - nosuid
    - nodev
    - bind
  - containerPath: /usr/lib/x86_64-linux-gnu/libnvidia-ptxjitcompiler.so.525.147.05
    hostPath: /usr/lib/x86_64-linux-gnu/libnvidia-ptxjitcompiler.so.525.147.05
    options:
    - ro
    - nosuid
    - nodev
    - bind
  - containerPath: /usr/lib/x86_64-linux-gnu/libnvidia-rtcore.so.525.147.05
    hostPath: /usr/lib/x86_64-linux-gnu/libnvidia-rtcore.so.525.147.05
    options:
    - ro
    - nosuid
    - nodev
    - bind
  - containerPath: /usr/lib/x86_64-linux-gnu/libnvidia-tls.so.525.147.05
    hostPath: /usr/lib/x86_64-linux-gnu/libnvidia-tls.so.525.147.05
    options:
    - ro
    - nosuid
    - nodev
    - bind
  - containerPath: /usr/lib/x86_64-linux-gnu/libnvoptix.so.525.147.05
    hostPath: /usr/lib/x86_64-linux-gnu/libnvoptix.so.525.147.05
    options:
    - ro
    - nosuid
    - nodev
    - bind
  - containerPath: /lib/firmware/nvidia/525.147.05/gsp_ad10x.bin
    hostPath: /lib/firmware/nvidia/525.147.05/gsp_ad10x.bin
    options:
    - ro
    - nosuid
    - nodev
    - bind
  - containerPath: /lib/firmware/nvidia/525.147.05/gsp_tu10x.bin
    hostPath: /lib/firmware/nvidia/525.147.05/gsp_tu10x.bin
    options:
    - ro
    - nosuid
    - nodev
    - bind
  - containerPath: /usr/share/X11/xorg.conf.d/10-nvidia.conf
    hostPath: /usr/share/X11/xorg.conf.d/10-nvidia.conf
    options:
    - ro
    - nosuid
    - nodev
    - bind
  - containerPath: /usr/share/egl/egl_external_platform.d/15_nvidia_gbm.json
    hostPath: /usr/share/egl/egl_external_platform.d/15_nvidia_gbm.json
    options:
    - ro
    - nosuid
    - nodev
    - bind
  - containerPath: /usr/share/glvnd/egl_vendor.d/10_nvidia.json
    hostPath: /usr/share/glvnd/egl_vendor.d/10_nvidia.json
    options:
    - ro
    - nosuid
    - nodev
    - bind
  - containerPath: /usr/share/vulkan/icd.d/nvidia_icd.json
    hostPath: /usr/share/vulkan/icd.d/nvidia_icd.json
    options:
    - ro
    - nosuid
    - nodev
    - bind
  - containerPath: /usr/lib/x86_64-linux-gnu/nvidia/xorg/libglxserver_nvidia.so.525.147.05
    hostPath: /usr/lib/x86_64-linux-gnu/nvidia/xorg/libglxserver_nvidia.so.525.147.05
    options:
    - ro
    - nosuid
    - nodev
    - bind
  - containerPath: /usr/lib/x86_64-linux-gnu/nvidia/xorg/nvidia_drv.so
    hostPath: /usr/lib/x86_64-linux-gnu/nvidia/xorg/nvidia_drv.so
    options:
    - ro
    - nosuid
    - nodev
    - bind
devices:
- containerEdits:
    deviceNodes:
    - path: /dev/nvidia0
    - path: /dev/dri/card0
    - path: /dev/dri/renderD128
    hooks:
    - args:
      - nvidia-ctk
      - hook
      - create-symlinks
      - --link
      - ../card0::/dev/dri/by-path/pci-0000:01:00.0-card
      - --link
      - ../renderD128::/dev/dri/by-path/pci-0000:01:00.0-render
      hookName: createContainer
      path: /usr/bin/nvidia-ctk
    - args:
      - nvidia-ctk
      - hook
      - chmod
      - --mode
      - "755"
      - --path
      - /dev/dri
      hookName: createContainer
      path: /usr/bin/nvidia-ctk
  name: "0"
- containerEdits:
    deviceNodes:
    - path: /dev/nvidia0
    - path: /dev/dri/card0
    - path: /dev/dri/renderD128
    hooks:
    - args:
      - nvidia-ctk
      - hook
      - create-symlinks
      - --link
      - ../card0::/dev/dri/by-path/pci-0000:01:00.0-card
      - --link
      - ../renderD128::/dev/dri/by-path/pci-0000:01:00.0-render
      hookName: createContainer
      path: /usr/bin/nvidia-ctk
    - args:
      - nvidia-ctk
      - hook
      - chmod
      - --mode
      - "755"
      - --path
      - /dev/dri
      hookName: createContainer
      path: /usr/bin/nvidia-ctk
  name: all
kind: nvidia.com/gpu
```

可以看到 nvidia 的 cdi 配置文件比 mock 的要复杂很多，因为 nvidia 的 gpu 需要绑定很多文件到容器内，还有 hooks 等。这些工作之前都是在 runtime 中做的，现在都可以通过 cdi 插件来做了。