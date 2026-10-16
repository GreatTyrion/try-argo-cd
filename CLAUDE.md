# CLAUDE.md

本文件为 Claude Code 提供本仓库的工作指引，同时记录**项目结构**和**当前进度**。
**每次项目有变化（新增文件或目录、完成一个步骤、调整方案）都要同步更新本文件。**

## 项目简介
学习 Argo CD 的实验仓库：先在本地 k3d 上动手，再部署到 GCP 的 GKE（预算 ≤ $20）。
- 主方案：[docs/learning-plan.md](docs/learning-plan.md)（第 0–16 步，每步都有「做什么 / 为什么 / 成功标准」）
- 费用与清理：[docs/cost-and-cleanup.md](docs/cost-and-cleanup.md)
- 参考项目审计：[docs/reference-audit.md](docs/reference-audit.md)（参考项目是 `../c-core-cd`，它的 CLAUDE.md 已部分过期，以它的代码为准）

## 规则
- 不要替用户 commit 或 push，除非用户明确要求。
- **绝不提交明文密钥**：明文文件必须以 `-unencrypted.yaml` 结尾（已被 gitignore），加密后的文件叫 `*-sops.yaml`；age 私钥只放在 `~/.config/sops/age/keys.txt`。
- 版本一律固定（Argo CD 的 install.yaml URL、Helm chart 版本、镜像 tag），不用 `stable` 或 `latest`。
- 环境差异通过 overlay 补丁实现，不复制 base。
- 涉及云资源的操作先看 `docs/cost-and-cleanup.md`；访问服务一律用 port-forward，不创建付费的负载均衡器或静态 IP（除非是方案中写明的临时演示）。
- 新的 prompt 放在 `prompts/`，按 `prompt-NN.md` 编号；方案和设计文档放在 `docs/`。

## 目录结构
```
try-argo-cd/
├── CLAUDE.md                 # 本文件：结构、规则、进度
├── README.md
├── .gitignore
├── prompts/                  # 驱动 Claude 的 prompt（prompt-01.md = 制定学习方案）
├── docs/                     # 方案、审计、费用文档
├── local/                    # k3d 集群配置（k3d-config.yaml，第 2 步）
├── bootstrap/
│   ├── argocd/base/          # Argo CD 安装：远程 install.yaml（v3.5.5）+ KSOPS 补丁（第 7、9 步）
│   ├── argocd/overlays/{local,gke}/   # 各环境的 Argo CD 配置差异
│   └── root-{local,gke}.yaml # 根 Application，每个集群只手动 apply 一次（第 7、13 步）
├── clusters/{local,gke}/     # App of Apps 的子项：Application / ApplicationSet / AppProject
├── apps/
│   ├── hello/{base,overlays/{local,gke}}/   # 自己的 Kustomize 示例应用（第 5 步）
│   └── podinfo/values-{local,gke}.yaml      # Helm chart 的 values（第 6 步）
├── infra/terraform/gke/      # GKE zonal 集群 + Spot 节点池（第 12 步）
└── scripts/                  # bootstrap-local.sh / teardown-local.sh / bootstrap-gke.sh
```
目前大部分目录只有 `.gitkeep`。具体文件由用户按方案逐步编写。

## 关键约定
| 项目 | 值 |
|---|---|
| Argo CD 版本 | v3.5.5 |
| 本地集群 / context | `argocd-lab` / `k3d-argocd-lab` |
| GKE 集群 / context | `argocd-lab`（us-central1-a，zonal）/ `gke-argocd-lab` |
| Argo CD UI | 本地 `https://localhost:8443`，GKE `https://localhost:9443`（都通过 port-forward） |
| 示例应用入口 | 本地 `http://hello.localhost:8080`（k3d 的 Traefik） |
| Git 仓库 | GitHub 公开仓库 `GreatTyrion/try-argo-cd` |

## 常用命令
```bash
kustomize build apps/hello/overlays/local                  # 本地预览渲染结果
kustomize build bootstrap/argocd/overlays/local
kubectl apply --server-side --force-conflicts -k bootstrap/argocd/overlays/local   # Argo CD CRD 太大，必须用 server-side apply
kubectl port-forward svc/argocd-server -n argocd 8443:443
argocd app list
```

## 当前进度
最后更新：2026-10-15

- [x] 制定学习方案、审计参考项目、建立目录骨架
- [x] 第 0 步 初始化项目仓库（2026-10-15：commit `1dce157` 已推送到 https://github.com/GreatTyrion/try-argo-cd ，公开仓库）
- [ ] 第 1 步 安装本地工具
- [ ] 第 2 步 创建本地 k3d 集群
- [ ] 第 3 步 手动安装 Argo CD
- [ ] 第 4 步 第一个 Application（guestbook）
- [ ] 第 5 步 自己的应用，跑通 GitOps 闭环
- [ ] 第 6 步 Helm 多源应用（podinfo）
- [ ] 第 7 步 App of Apps，Argo CD 管理自己
- [ ] 第 8 步 AppProject、ApplicationSet、同步顺序
- [ ] 第 9 步 SOPS + age + KSOPS
- [ ] 第 10 步 从零一键重建本地环境
- [ ] 第 11 步 GCP 项目和预算提醒
- [ ] 第 12 步 Terraform 创建 GKE
- [ ] 第 13 步 在 GKE 上引导 Argo CD
- [ ] 第 14 步 应用部署到 GKE
- [ ] 第 15 步（可选）多集群
- [ ] 第 16 步 清理云端资源

当前云端资源：**无**（还没有创建任何 GCP 资源，费用 $0）。
