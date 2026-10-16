# 参考项目审计：c-core-cd 的 CLAUDE.md 哪里过期了

- 审计对象：`../c-core-cd/CLAUDE.md`
- 审计日期：2026-10-15（对照当时的代码和 `git log`，最新提交 `582e137`）
- 结论：文档描述的是项目**早期**只管 Prefect 一个应用的样子。现在项目已经发展成一个 Argo CD 统一管理 4 个集群的多集群 GitOps 仓库，文档大部分架构描述都已过期。

## 过期或不准确的地方

| # | 文档原话（大意） | 实际情况 | 证据 |
|---|---|---|---|
| 1 | 单个 GKE Autopilot 集群（`c-core-autopilot`） | c-core-autopilot 上的 Argo CD 是**控制中心**，还管理另外 3 个集群：`c-core-iceland`（GKE，europe-north1）、`dalo` 和 `c-core-dalo`（两个 AWS EKS，eu-north-1） | `terraform/variables.tf` 中的 `iceland_*` 变量；`holmes-applicationset.yaml` 的 destination 指向 EKS 地址；`*-cluster-secret-sops.yaml` |
| 2 | 只有 `overlays/c-core-autopilot/` 一个 overlay | 一共有 5 个：`c-core-autopilot`、`c-core-iceland`、`c-core-dalo`、`dalo`、`k3d` | `ls overlays/` |
| 3 | `base/argocd` 引用 `stable/manifests/install.yaml` | 版本固定为 **`v3.4.3`**，比最新的 v3.5.5 落后。另外加了 KSOPS 补丁（`argocd-repo-server-ksops-patch.yaml`） | `base/argocd/kustomization.yaml` |
| 4 | `argocd-cm` 只设置了 `--enable-helm` | 实际是 `--enable-helm --enable-alpha-plugins --enable-exec`（KSOPS 需要后两个参数） | `base/argocd/argocd-cm-patch.yaml` |
| 5 | `base/prefect` 用 Kustomize `helmCharts:` 渲染 prefect-server | 这里只剩一个 `configmap-base-job-template.yaml`。Prefect 改由 **prefect-operator** 管理（`PrefectServer` CR） | `base/prefect/kustomization.yaml`、`overlays/c-core-autopilot/prefect/prefect-server.yaml` |
| 6 | overlay 打补丁设置 `PREFECT_SERVER_API_BASE_PATH=/prefect/api`，并修改 readiness probe | 文件注释明确写着 **不要** 设置这个变量（operator 会覆盖 probe，设了会导致服务反复崩溃）。外部的 `/prefect/api` 路径改为在 HTTPRoute 里重写成 `/api` | `prefect-server.yaml` 顶部注释 |
| 7 | Gateway 资源放在 prefect overlay 里，cert map 叫 `prefect-cert-map` | Gateway 已经独立到 `overlays/c-core-autopilot/gateway/`（名字 `c-core-gateway`，namespace `gateway`）。cert map 叫 **`c-core-cert-map`**，并使用静态 IP `c-core-gateway-ip` | `overlays/c-core-autopilot/gateway/gateway.yaml`、`terraform/main.tf` |
| 8 | DNS 记录手工管理，不在 Terraform 里 | `dev.c-core.app` 的 A 记录**已经由 Terraform 管理**（`google_dns_record_set.gateway`），只有 zone `c-core` 本身还是手工创建的 | `terraform/main.tf` |
| 9 | Argo CD overlay 只创建 Prefect 一个 Application | 实际有约 35 个 Application 和 2 个 ApplicationSet，包括 image-updater（Helm 多源写法）、config-connector、atlas-operator、cloudnative-pg、hasura 等 | `overlays/c-core-autopilot/argocd/kustomization.yaml` |
| 10 | Terraform 只管集群、VPC 和证书 | 还有：`argocd-iceland.tf`（Argo CD 使用的 GSA、Workload Identity、KMS 解密权限）、`kcc-*.tf`（Config Connector）、IAP 用户授权，以及 provider 别名 `ccore_holmes` | `terraform/*.tf` |
| 11 | 密钥只在本地用 `sops --encrypt --gcp-kms ...` 加密 | 仓库里已经有 `.sops.yaml` 创建规则，加密时**不需要**再传 `--gcp-kms`。集群里由 **KSOPS** 解密：repo-server 通过 Workload Identity 获得 KMS 解密权限 | `.sops.yaml`、`terraform/argocd-iceland.tf` |
| 12 | 文档没提到 | Argo CD 通过 `https://dev.c-core.app/argocd` 对外暴露（`server.rootpath: /argocd`，`server.insecure: true`，前面有 IAP 保护） | `argocd-cmd-params-cm-patch.yaml`、`httproute.yaml` |
| 13 | 文档没提到 | 已经有本地 k3d 环境（`scripts/k3d.sh` + `overlays/k3d`）。它**不使用 Argo CD**，直接用 kustomize 部署，原因见 `overlays/k3d/README.md` | `overlays/k3d/README.md` |
| 14 | `kustomize build overlays/c-core-autopilot/argocd` 可以直接预览 | argocd overlay 中的 `*-sops.yaml`（admin 密码、GitHub App 凭证、各集群凭证）**没有写进** `kustomization.yaml`，看起来是手动解密后 apply 的。文档应该写明这个手动步骤 | 对比 `ls overlays/c-core-autopilot/argocd` 和 `kustomization.yaml` |

## 文档中仍然准确的部分
- 引导方式是一次性的 `kubectl apply --server-side --force-conflicts -k overlays/c-core-autopilot/argocd`，Argo CD 的 overlay 本身**不自动同步**，也就是 Argo CD 没有管理自己（仓库里找不到指向 `overlays/c-core-autopilot/argocd` 的 Application）。
- 对外流量用 GKE Gateway API（`gke-l7-global-external-managed`），TLS 用 Certificate Manager，没有用 Ingress。
- 「通过 overlay 打补丁实现集群差异，不复制 base」这一原则依然成立。
- `plans/` 和 `prompts/` 这两个目录的用途描述正确。

## 对本学习项目的启发
| 参考项目的做法 | 本项目的做法 | 原因 |
|---|---|---|
| Argo CD 用 Kustomize 远程 base 安装，版本固定 | 沿用，固定 v3.5.5 | 版本可复现 |
| Argo CD 不管理自己 | **改进**：加一个 Application 让 Argo CD 管理自己 | 升级和配置修改也走 Git |
| SOPS + GCP KMS + KSOPS | SOPS + **age** + KSOPS | 机制相同，但不花钱、不依赖云 |
| Autopilot 集群 | **Standard zonal + Spot 节点** | Autopilot 按 Pod 计费，跑 Argo CD 每月约 $70，超出预算 |
| 用 Gateway + 证书 + 静态 IP 暴露服务 | 用 port-forward | 负载均衡器和静态 IP 按小时计费 |
| 根目录堆放了 notebook、geojson 等杂项文件 | 目录分工明确，杂项不进仓库 | 保持结构清晰 |
