# 费用估算与清理清单

> 价格查询于 2026-10-15。Spot 价格每天都可能变，正式开始云端步骤前，请到 [Spot VMs pricing](https://cloud.google.com/spot-vms/pricing) 和 [GKE pricing](https://cloud.google.com/kubernetes-engine/pricing) 再核对一次。

## 1. 为什么选「GKE Standard zonal 集群 + 1 个 Spot 节点」

| 方案 | 集群管理费 | 计算费用 | 跑 Argo CD 一个月的大致费用 | 结论 |
|---|---|---|---|---|
| **GKE Standard zonal + 1×Spot e2-standard-2** | $0.10/h，**免费额度全额抵扣** | 约 $0.02–0.034/h | **约 $15–25（全月不关）** | ✅ 最便宜，按需开关的话只要几美元 |
| GKE Autopilot | $0.10/h，免费额度可抵扣 | 按 Pod 申请的资源计费（约 $0.0445/vCPU·h + $0.0049/GiB·h），Argo CD 的 7 个 Pod 按最小规格算也要约 1.75 vCPU / 3.5 GiB | 约 $70 | ❌ 超预算 |
| GKE regional 集群 | $0.10/h，**免费额度不能抵扣** | — | 仅管理费就要 $73 | ❌ |
| 普通（非 Spot）e2-standard-2 | 免费额度抵扣 | 约 $0.067/h | 约 $49 | ❌ |

GKE 的免费额度是每个结算账号每月 $74.40，只能抵扣**一个** zonal 或 Autopilot 集群的管理费。所以整个学习期间**同时只保留一个 GKE 集群**。

## 2. 预计费用

| 项目 | 单价 | 推荐用法（只在学习时开，约 40 小时） | 如果连续开 2 周（336 小时） |
|---|---|---|---|
| GKE 管理费 | $0.10/h | $0（免费额度抵扣） | $0（免费额度抵扣） |
| Spot e2-standard-2 ×1 | 约 $0.02–0.034/h | 约 $1.4 | 约 $11 |
| 节点磁盘 pd-standard 30 GB | 约 $0.04/GB·月 | 约 $0.1 | 约 $0.6 |
| 集群刚创建时的默认节点池（几分钟后就删除） | — | < $0.05 | < $0.05 |
| 网络出站流量（拉镜像是入站，不收费） | — | < $0.5 | < $1 |
| Cloud Logging（只收集系统组件日志，在免费额度内） | — | $0 | $0 |
| 临时 LoadBalancer（第 14 步可选，开 2 小时） | 约 $0.025/h | 约 $0.05 | 约 $0.05 |
| **合计** | | **约 $2** | **约 $13** |

两种情况都远低于 **$20** 的预算。再加上第 11 步设置的预算提醒（$5 / $10 / $15 / $20），超支前就会收到邮件。

> 注意：预算提醒**只发通知，不会自动停止**资源。真正避免超支要靠第 16 步的清理。
> 新注册的 GCP 账号通常有 $300 试用额度，可以先用试用额度来学。

### 省钱习惯
- **学完一次就 `terraform destroy`**，下次需要时再 `terraform apply`（大约 8–10 分钟）。所有配置都在 Git 里，Argo CD 能一键恢复，这本身就是 GitOps 的意义。
- 不创建静态 IP、Cloud NAT、Gateway 或 Ingress。访问服务一律用 `kubectl port-forward`。
- 不创建 PVC 磁盘（示例应用都是无状态的）。

## 3. 清理清单

### 云端（每次学习结束、全部学完时都要做）
```bash
# 0) 先删除会产生「集群之外」云资源的 Kubernetes 对象（LoadBalancer Service、PVC）
kubectl --context gke-argocd-lab get svc -A | grep LoadBalancer   # 应该没有输出
kubectl --context gke-argocd-lab get pvc -A                       # 应该没有输出
#    如果有：在 Git 里删掉对应的配置，让 Argo CD 自动清理（prune），或者手动 kubectl delete

# 1) 销毁集群和节点池（⚠️ 会删除整个 GKE 集群）
cd infra/terraform/gke && terraform destroy

# 2) 逐项确认（全部应该返回空列表，或者 "Listed 0 items."）
gcloud container clusters list
gcloud compute instances list
gcloud compute disks list
gcloud compute addresses list
gcloud compute forwarding-rules list
gcloud compute target-pools list
```

### 全部学完后（最彻底的做法，推荐）
```bash
# ⚠️ 删除整个项目：项目中所有资源都会停止计费，30 天后彻底删除
gcloud projects delete <PROJECT_ID>
```
- 第二天打开 Cloud Console → Billing → Reports，按项目筛选，确认当天费用是 $0。
- 预算提醒可以保留，也可以用 `gcloud billing budgets delete` 删除（预算本身不收费）。

### 本地
```bash
k3d cluster delete argocd-lab      # 删除本地集群
docker system prune                # （可选）清理 Docker 镜像缓存
```

### 清理完成的标准
- 上面所有 `gcloud ... list` 命令都返回空。
- `k3d cluster list` 中没有 `argocd-lab`。
- 第二天的账单报表显示这个项目的当日费用为 $0。
