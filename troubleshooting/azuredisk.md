# AzureDisk 排错

[AzureDisk](https://docs.microsoft.com/zh-cn/azure/virtual-machines/windows/about-disks-and-vhds) 为 Azure 上面运行的虚拟机提供了弹性块存储服务，它以 VHD 的形式挂载到虚拟机中，并可以在 Kubernetes 容器中使用。AzureDisk 有点是性能高，特别是 Premium Storage 提供了非常好的[性能](https://docs.microsoft.com/en-us/azure/virtual-machines/windows/premium-storage)；其缺点是不支持共享，只可以用在单个 Pod 内。

根据配置的不同，Kubernetes 支持的 AzureDisk 可以分为以下几类

- Managed Disks: 由 Azure 自动管理磁盘和存储账户
- Blob Disks:
  - Dedicated (默认)：为每个 AzureDisk 创建单独的存储账户，当删除 PVC 的时候删除该存储账户
  - Shared：AzureDisk 共享 ResourceGroup 内的同一个存储账户，这时删除 PVC 不会删除该存储账户

> 注意：
> - AzureDisk 的类型必须跟 VM OS Disk 类型一致，即要么都是 Manged Disks，要么都是 Blob Disks。当两者不一致时，AzureDisk PV 会报无法挂载的错误。
> - 由于 Managed Disks 需要创建和管理存储账户，其创建过程会比 Blob Disks 慢（3 分钟 vs 1-2 分钟）。
> - 但节点最大支持同时挂载 16 个 AzureDisk。

使用 AzureDisk 推荐的版本：

| Kubernetes version | Recommended version |
| ------------------ | :-----------------: |
| 1.12               |  1.12.9 或更高版本  |
| 1.13               |  1.13.6 或更高版本  |
| 1.14               |  1.14.2 或更高版本  |
| >=1.15             |       >=1.15        |

使用 [aks-engine](https://github.com/Azure/aks-engine) 部署的 Kubernetes 集群，会自动创建两个 StorageClass，默认为managed-standard（即HDD）：

```sh
kubectl get storageclass
NAME                PROVISIONER                AGE
default (default)   kubernetes.io/azure-disk   45d
managed-premium     kubernetes.io/azure-disk   53d
managed-standard    kubernetes.io/azure-disk   53d
```

## AzureDisk 挂载失败

在 AzureDisk 从一个 Pod 迁移到另一 Node 上面的 Pod 时或者同一台 Node 上面使用了多块 AzureDisk 时有可能会碰到这个问题。这是由于 kube-controller-manager 未对 AttachDisk 和 DetachDisk 操作加锁从而引发了竞争问题（[kubernetes#60101](https://github.com/kubernetes/kubernetes/issues/60101) [acs-engine#2002](https://github.com/Azure/acs-engine/issues/2002) [ACS#12](https://github.com/Azure/ACS/issues/12)）。

通过 kube-controller-manager 的日志，可以查看具体的错误原因。常见的错误日志为

```sh
Cannot attach data disk 'cdb-dynamic-pvc-92972088-11b9-11e8-888f-000d3a018174' to VM 'kn-edge-0' because the disk is currently being detached or the last detach operation failed. Please wait until the disk is completely detached and then try again or delete/detach the disk explicitly again.
```

临时性解决方法为

（1）更新所有受影响的虚拟机状态

使用powershell：

```powershell
$vm = Get-AzureRMVM -ResourceGroupName $rg -Name $vmname
Update-AzureRmVM -ResourceGroupName $rg -VM $vm -verbose -debug
```

使用 Azure CLI：

```sh
# For VM:
az vm update -n <VM_NAME> -g <RESOURCE_GROUP_NAME>

# For VMSS:
az vmss update-instances -g <RESOURCE_GROUP_NAME> --name <VMSS_NAME> --instance-id <ID>
```

（2）重启虚拟机

- `kubectl cordon NODE`
- 如果 Node 上运行有 StatefulSet，需要手动删除相应的 Pod
- `kubectl drain NODE`
- `Get-AzureRMVM -ResourceGroupName $rg -Name $vmname | Restart-AzureVM`
- `kubectl uncordon NODE`

该问题的修复 [#60183](https://github.com/kubernetes/kubernetes/pull/60183) 已包含在 v1.10 中。

## 挂载新的 AzureDisk 后，该 Node 中其他 Pod 已挂载的 AzureDisk 不可用

在 Kubernetes v1.7 中，AzureDisk 默认的缓存策略修改为 `ReadWrite`，这会导致在同一个 Node 中挂载超过 5 块 AzureDisk 时，已有 AzureDisk 的盘符会随机改变（[kubernetes#60344](https://github.com/kubernetes/kubernetes/issues/60344) [kubernetes#57444](https://github.com/kubernetes/kubernetes/issues/57444) [AKS#201](https://github.com/Azure/AKS/issues/201) [acs-engine#1918](https://github.com/Azure/acs-engine/issues/1918)）。比如，当挂载第六块 AzureDisk 后，原来 lun0 磁盘的挂载盘符有可能从 `sdc` 变成 `sdk`：

```sh
$ tree /dev/disk/azure
...
â””â”€â”€ scsi1
    â”œâ”€â”€ lun0 -> ../../../sdk
    â”œâ”€â”€ lun1 -> ../../../sdj
    â”œâ”€â”€ lun2 -> ../../../sde
    â”œâ”€â”€ lun3 -> ../../../sdf
    â”œâ”€â”€ lun4 -> ../../../sdg
    â”œâ”€â”€ lun5 -> ../../../sdh
    â””â”€â”€ lun6 -> ../../../sdi
```

这样，原来使用 lun0 磁盘的 Pod 就无法访问 AzureDisk 了

```sh
[root@admin-0 /]# ls /datadisk
ls: reading directory .: Input/output error
```

临时性解决方法是设置 AzureDisk StorageClass 的 `cachingmode: None`，如

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: managed-standard
provisioner: kubernetes.io/azure-disk
parameters:
  skuname: Standard_LRS
  kind: Managed
  cachingmode: None
```

该问题的修复 [#60346](https://github.com/kubernetes/kubernetes/pull/60346) 将包含在 v1.10 中。

## AzureDisk 挂载慢

AzureDisk PVC 的挂载过程一般需要 1 分钟的时间，这些时间主要消耗在 Azure ARM API 的调用上（查询 VM 以及挂载 Disk）。[#57432](https://github.com/kubernetes/kubernetes/pull/57432) 为 Azure VM 增加了一个缓存，消除了 VM 的查询时间，将整个挂载过程缩短到大约 30 秒。该修复包含在v1.9.2+ 和 v1.10 中。

另外，如果 Node 使用了 `Standard_B1s` 类型的虚拟机，那么 AzureDisk 的第一次挂载一般会超时，等再次重复时才会挂载成功。这是因为在 `Standard_B1s`  虚拟机中格式化 AzureDisk 就需要很长时间（如超过 70 秒）。

```sh
$ kubectl describe pod <pod-name>
...
Events:
  FirstSeen     LastSeen        Count   From                                    SubObjectPath                           Type            Reason                  Message
  ---------     --------        -----   ----                                    -------------                           --------        ------                  -------
  8m            8m              1       default-scheduler                                                               Normal          Scheduled               Successfully assigned nginx-azuredisk to aks-nodepool1-15012548-0
  7m            7m              1       kubelet, aks-nodepool1-15012548-0                                               Normal          SuccessfulMountVolume   MountVolume.SetUp succeeded for volume "default-token-mrw8h"
  5m            5m              1       kubelet, aks-nodepool1-15012548-0                                               Warning         FailedMount             Unable to mount volumes for pod "nginx-azuredisk_default(4eb22bb2-0bb5-11e8-8
d9e-0a58ac1f0a2e)": timeout expired waiting for volumes to attach/mount for pod "default"/"nginx-azuredisk". list of unattached/unmounted volumes=[disk01]
  5m            5m              1       kubelet, aks-nodepool1-15012548-0                                               Warning         FailedSync              Error syncing pod
  4m            4m              1       kubelet, aks-nodepool1-15012548-0                                               Normal          SuccessfulMountVolume   MountVolume.SetUp succeeded for volume "pvc-20240841-0bb5-11e8-8d9e-0a58ac1f0
a2e"
  4m            4m              1       kubelet, aks-nodepool1-15012548-0       spec.containers{nginx-azuredisk}        Normal          Pulling                 pulling image "nginx"
  3m            3m              1       kubelet, aks-nodepool1-15012548-0       spec.containers{nginx-azuredisk}        Normal          Pulled                  Successfully pulled image "nginx"
  3m            3m              1       kubelet, aks-nodepool1-15012548-0       spec.containers{nginx-azuredisk}        Normal          Created                 Created container
  2m            2m              1       kubelet, aks-nodepool1-15012548-0       spec.containers{nginx-azuredisk}        Normal          Started                 Started container
```

## Azure German Cloud 无法使用 AzureDisk

Azure German Cloud 仅在 v1.7.9+、v1.8.3+ 以及更新版本中支持（[#50673](https://github.com/kubernetes/kubernetes/pull/50673)），升级 Kubernetes 版本即可解决。

## MountVolume.WaitForAttach failed

```sh
MountVolume.WaitForAttach failed for volume "pvc-f1562ecb-3e5f-11e8-ab6b-000d3af9f967" : azureDisk - Wait for attach expect device path as a lun number, instead got: /dev/disk/azure/scsi1/lun1 (strconv.Atoi: parsing "/dev/disk/azure/scsi1/lun1": invalid syntax)
```

[该问题](https://github.com/kubernetes/kubernetes/issues/62540) 仅在 Kubernetes v1.10.0 和 v1.10.1 中存在，将在 v1.10.2 中修复。

## 在 mountOptions 中设置 uid 和 gid 时失败

默认情况下，Azure 磁盘使用 ext4、xfs filesystem 和 mountOptions，如 uid = x，gid = x 无法在装入时设置。 例如，如果你尝试设置 mountOptions uid = 999，gid = 999，将看到类似于以下内容的错误:

```console
Warning  FailedMount             63s                  kubelet, aks-nodepool1-29460110-0  MountVolume.MountDevice failed for volume "pvc-d783d0e4-85a1-11e9-8a90-369885447933" : azureDisk - mountDevice:FormatAndMount failed with mount failed: exit status 32
Mounting command: systemd-run
Mounting arguments: --description=Kubernetes transient mount for /var/lib/kubelet/plugins/kubernetes.io/azure-disk/mounts/m436970985 --scope -- mount -t xfs -o dir_mode=0777,file_mode=0777,uid=1000,gid=1000,defaults /dev/disk/azure/scsi1/lun2 /var/lib/kubelet/plugins/kubernetes.io/azure-disk/mounts/m436970985
Output: Running scope as unit run-rb21966413ab449b3a242ae9b0fbc9398.scope.
mount: wrong fs type, bad option, bad superblock on /dev/sde,
       missing codepage or helper program, or other error
```

可以通过执行以下操作之一来缓解此问题

* 通过在 fsGroup 中的 runAsUser 和 gid 中设置 uid 来[配置 pod 的安全上下文](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)。例如，以下设置会将 pod 设置为 root，使其可供任何文件访问：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo
spec:
  securityContext:
    runAsUser: 0
    fsGroup: 0
```

> 备注: 因为 gid 和 uid 默认装载为 root 或0。如果 gid 或 uid 设置为非根（例如1000），则 Kubernetes 将使用 `chown` 更改该磁盘下的所有目录和文件。此操作可能非常耗时，并且可能会导致装载磁盘的速度非常慢。

* 使用 initContainers 中的 `chown` 设置 gid 和 uid。例如：

```yaml
initContainers:
- name: volume-mount
  image: busybox
  command: ["sh", "-c", "chown -R 100:100 /data"]
  volumeMounts:
  - name: <your data volume>
    mountPath: /data
```

## 删除 pod 使用的 Azure 磁盘 PersistentVolumeClaim 时出错

如果尝试删除 pod 使用的 Azure 磁盘 PersistentVolumeClaim，可能会看到错误。例如：

```console
$ kubectl describe pv pvc-d8eebc1d-74d3-11e8-902b-e22b71bb1c06
...
Message:         disk.DisksClient#Delete: Failure responding to request: StatusCode=409 -- Original Error: autorest/azure: Service returned an error. Status=409 Code="OperationNotAllowed" Message="Disk kubernetes-dynamic-pvc-d8eebc1d-74d3-11e8-902b-e22b71bb1c06 is attached to VM /subscriptions/{subs-id}/resourceGroups/MC_markito-aks-pvc_markito-aks-pvc_westus/providers/Microsoft.Compute/virtualMachines/aks-agentpool-25259074-0."
```

在 Kubernetes 版本1.10 及更高版本中，默认情况下已启用 PersistentVolumeClaim protection 功能以防止此错误。如果你使用的 Kubernetes 版本不能解决此问题，则可以通过在删除 PersistentVolumeClaim 前使用 PersistentVolumeClaim 删除 pod 来缓解此问题。

## 参考文档

- [Known kubernetes issues on Azure](https://github.com/andyzhangx/demo/tree/master/issues)
- [Introduction of AzureDisk](https://docs.microsoft.com/zh-cn/azure/virtual-machines/windows/about-disks-and-vhds)
- [AzureDisk volume examples](https://github.com/kubernetes/examples/tree/master/staging/volumes/azure_disk)
- [High-performance Premium Storage and managed disks for VMs](https://docs.microsoft.com/en-us/azure/virtual-machines/windows/premium-storage)
