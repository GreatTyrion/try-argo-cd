# Argo CD 学习与部署方案

> 编写日期：2026-10-15。版本号来自各项目 GitHub releases 的最新稳定版，用法来自 Argo CD 官方文档（Context7）。
> 每一步都分三段：**做什么**、**为什么**、**成功标准**。完成一步后，在根目录 `CLAUDE.md` 的进度清单里打勾。

## 总览

```
阶段 0 准备          阶段 1 本地 k3d（$0）                         阶段 2 云端 GKE（约 $2–13）
──────────           ─────────────────────────────────────         ─────────────────────────────
0 项目初始化  ──▶    2 建集群 ─▶ 3 装 Argo CD ─▶ 4 第一个 App       11 GCP 准备
1 装工具             5 自己的 App（GitOps 闭环）                    12 Terraform 建 GKE
                     6 Helm 多源 App                                13 在 GKE 上引导 Argo CD
                     7 App of Apps + 管理自己                       14 应用部署到 GKE
                     8 AppProject + ApplicationSet + 同步顺序        15（可选）多集群
                     9 SOPS 密钥                                    16 清理
                     10 从零一键重建
```

**最终架构**（每个集群上的结构相同，只是 overlay 不同）：

```
GitHub 仓库 try-argo-cd ──(拉取)──▶ Argo CD（运行在集群内）
                                     │
     bootstrap/root-<env>.yaml ──▶  root（App of Apps，指向 clusters/<env>/）
                                     ├─ argocd       → bootstrap/argocd/overlays/<env>（Argo CD 管理自己）
                                     ├─ AppProject learning（权限边界）
                                     ├─ ApplicationSet apps → apps/*/overlays/<env>（每个目录生成一个 App）
                                     └─ podinfo      → Helm chart + Git 中的 values（多源）
```

**版本表**（2026-10-15 核实）

| 组件 | 版本 | 用途 |
|---|---|---|
| Argo CD | **v3.5.5** | GitOps 控制器（3.6 还在 RC 阶段，不用） |
| k3d | v5.9.0 | 在本地 Docker 里跑 k3s 集群 |
| kubectl | 本机已有 v1.34.1 | 操作集群 |
| kustomize | v5.8.3 | 渲染 base/overlays |
| Helm | v4.3.1 | 本地预览 chart（Argo CD 内部自带 Helm） |
| argocd CLI | 跟随 brew（3.5.x） | 命令行操作 Argo CD |
| SOPS / age / KSOPS | v3.13.3 / v1.3.2 / v4.5.1 | 加密密钥后再提交到 Git |
| podinfo chart | 6.15.0 | Helm 示例应用 |
| gcloud CLI | 最新 | GCP 操作 |
| Terraform / google provider | v1.16.6 / ~> 8.7 | 基础设施即代码 |

**命名约定**：本地集群叫 `argocd-lab`（kube context：`k3d-argocd-lab`），GKE 集群也叫 `argocd-lab`（context 重命名为 `gke-argocd-lab`），GitHub 仓库为 [`GreatTyrion/try-argo-cd`](https://github.com/GreatTyrion/try-argo-cd)（公开）。

---

# 阶段 0：准备

## 第 0 步：初始化项目仓库

**做什么**
```bash
cd ~/Documents/projects/learn-argo-cd/try-argo-cd
git status                                  # 本地已初始化 git，目录骨架已建好
git add -A && git commit -m "Initiate project"
git remote add origin https://github.com/GreatTyrion/try-argo-cd.git   # 仓库已在 GitHub 网页上创建
git push -u origin main
```

**为什么**
GitOps 的核心思想是「**Git 是唯一的事实来源**」：Argo CD 从 Git 仓库读取期望状态，再让集群和它保持一致。所以必须先有一个 Argo CD 能访问到的远程仓库。这里用公开仓库，Argo CD 不需要任何凭证就能拉取（私有仓库的配置见第 8 步的可选练习）。**任何明文密钥都不能提交**，`.gitignore` 已经排除了 `*-unencrypted.yaml` 和 `*.agekey`。

**成功标准**
- `git status` 输出 `nothing to commit, working tree clean`。
- 打开 https://github.com/GreatTyrion/try-argo-cd 能看到 `docs/`、`prompts/` 等目录。
- `git ls-remote origin main` 输出的 commit 与 `git rev-parse HEAD` 相同。
- `git remote -v` 显示 `origin  https://github.com/GreatTyrion/try-argo-cd.git`。

## 第 1 步：安装本地工具

**做什么**
```bash
brew install k3d kustomize helm argocd sops age
# 本机已有：docker、kubectl、gh、git
# 确认 Docker 有足够内存：Docker Desktop → Settings → Resources，建议 ≥ 6 GB
```

**为什么**
- `k3d`：在 Docker 容器里运行轻量的 k3s，几秒钟就能创建或删除一个集群，适合反复试错。参考项目也用它。
- `kustomize` / `helm`：在本地**预览** Argo CD 将要渲染出的 YAML，在提交前就发现错误。
- `argocd`：Argo CD 的命令行工具，方便写脚本和验证。
- `sops` / `age`：第 9 步加密密钥要用。
- 云端工具（gcloud、terraform）到第 11 步再装，避免一开始装太多东西。

**成功标准**
```bash
k3d version            # k3d version v5.9.x
kustomize version      # v5.8.x
helm version --short   # v4.3.x
argocd version --client --short   # argocd: v3.5.x
sops --version         # sops 3.13.x
age --version          # v1.3.x
docker info | grep "Total Memory"   # ≥ 6GiB
```

---

# 阶段 1：本地实验（k3d，花费 $0）

## 第 2 步：创建本地 k3d 集群

**做什么**
新建 `local/k3d-config.yaml`：
```yaml
apiVersion: k3d.io/v1alpha5
kind: Simple
metadata:
  name: argocd-lab
servers: 1
agents: 0
ports:
  - port: 8080:80          # k3s 自带的 Traefik → http://localhost:8080
    nodeFilters:
      - loadbalancer
options:
  k3d:
    wait: true
  k3s:
    extraArgs:
      - arg: --disable=metrics-server
        nodeFilters:
          - server:*
```
```bash
k3d cluster create --config local/k3d-config.yaml
kubectl config current-context
```

**为什么**
集群是 Argo CD 运行的地方，也是它部署应用的目标。用配置文件而不是一长串命令行参数，是为了让集群本身**可以复现**（同样是「一切皆代码」的思路）。端口 8080 映射到 k3s 自带的 Traefik，后面可以用 Ingress 在浏览器里访问示例应用。关掉 metrics-server 是为了省内存。

**成功标准**
- `kubectl config current-context` 输出 `k3d-argocd-lab`。
- `kubectl get nodes` 显示 1 个节点，STATUS 为 `Ready`。
- `kubectl get pods -n kube-system` 中 coredns、traefik 都是 `Running`（或 helm-install 类 Job 显示 `Completed`）。

## 第 3 步：手动安装 Argo CD，认识各个组件

**做什么**
```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/v3.5.5/manifests/install.yaml
kubectl wait --for=condition=available --timeout=300s deployment --all -n argocd

# 打开 UI（单独开一个终端，保持运行）
kubectl port-forward svc/argocd-server -n argocd 8443:443
# 浏览器访问 https://localhost:8443（自签证书，点击「继续访问」）

# 获取初始密码并登录 CLI
argocd admin initial-password -n argocd
argocd login localhost:8443 --username admin --insecure
argocd account update-password
kubectl -n argocd delete secret argocd-initial-admin-secret   # 改完密码后删除
```
看一下有哪些组件：
```bash
kubectl get deploy,statefulset -n argocd
```
| 组件 | 作用 |
|---|---|
| `argocd-server` | API 和 Web UI，CLI 也连它 |
| `argocd-repo-server` | 克隆 Git 仓库，把 Kustomize/Helm/YAML **渲染**成最终的清单 |
| `argocd-application-controller`（StatefulSet） | 核心：不断比较「Git 期望状态」和「集群实际状态」，必要时执行同步 |
| `argocd-applicationset-controller` | 根据模板批量生成 Application（第 8 步） |
| `argocd-redis` | 缓存 |
| `argocd-dex-server` | SSO 登录（本项目只用 admin，用不到） |
| `argocd-notifications-controller` | 发通知（Slack、邮件等） |

**为什么**
先用官方的一条命令装起来，最直观。版本固定为 `v3.5.5` 而不是 `stable`，保证明天重装得到的是同一个版本。必须加 `--server-side`，因为 Argo CD 的 CRD 太大，超过了客户端 apply 的注解长度限制（参考项目文档里也提到了这一点）。初始密码以明文存在 Secret 中，改完密码就应该删掉。

**成功标准**
- `kubectl get pods -n argocd` 中全部 Pod 都是 `Running`，READY 列为 `1/1`。
- 能在浏览器 `https://localhost:8443` 用 admin 和新密码登录。
- `argocd version` 同时显示 client 和 server 的版本，server 为 `v3.5.5+...`。
- `kubectl -n argocd get secret argocd-initial-admin-secret` 返回 `NotFound`。

## 第 4 步：第一个 Application（官方 guestbook 示例）

**做什么**
```bash
argocd app create guestbook \
  --repo https://github.com/argoproj/argocd-example-apps.git \
  --path guestbook \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace guestbook \
  --sync-option CreateNamespace=true

argocd app get guestbook        # 此时状态是 OutOfSync（还没部署）
argocd app sync guestbook       # 手动同步
argocd app get guestbook        # Synced + Healthy
argocd app history guestbook
```
在 UI 里点开 guestbook，看资源树（Application → Service / Deployment → ReplicaSet → Pod）。然后删除它：
```bash
argocd app delete guestbook --yes
```

**为什么**
理解 Argo CD 最核心的对象 **Application**：它把「来源（repo + path + 版本）」和「目标（集群 + namespace）」绑在一起。理解两种状态：
- **Sync 状态**：集群和 Git 一致吗？（`Synced` / `OutOfSync`）
- **Health 状态**：资源在正常工作吗？（`Healthy` / `Progressing` / `Degraded` / `Missing`）

这里先用**手动同步**，感受「Argo CD 只发现差异，什么时候同步由你决定」。

**成功标准**
- 同步前，`argocd app get guestbook` 显示 `Sync Status: OutOfSync`。
- 同步后显示 `Sync Status: Synced` 和 `Health Status: Healthy`。
- `kubectl get deploy -n guestbook` 显示 `guestbook-ui   1/1`。
- 删除后，`kubectl get all -n guestbook` 输出 `No resources found`（CLI 默认级联删除资源；namespace 本身会保留）。

## 第 5 步：自己的应用，跑通完整的 GitOps 闭环

**做什么**
1. 在 `apps/hello/` 下用 Kustomize 写一个 nginx 示例（base 放共同部分，overlay 放环境差异，**overlay 中不写 namespace**，由 Application 的 destination 决定）：
   ```
   apps/hello/base/        kustomization.yaml  deployment.yaml(nginx:1.31-alpine)  service.yaml  index.html
   apps/hello/overlays/local/  kustomization.yaml（replicas: 1、页面内容 "Hello from local"、ingress.yaml 主机名 hello.localhost）
   ```
   页面内容用 `configMapGenerator` 生成。生成的 ConfigMap 名字带内容哈希，内容一变，Deployment 就会自动滚动更新。
2. 先在本地预览：`kustomize build apps/hello/overlays/local`。
3. 写声明式 Application `clusters/local/hello.yaml`：
   ```yaml
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: hello
     namespace: argocd
   spec:
     project: default
     source:
       repoURL: https://github.com/GreatTyrion/try-argo-cd.git
       targetRevision: HEAD
       path: apps/hello/overlays/local
     destination:
       server: https://kubernetes.default.svc
       namespace: hello
     syncPolicy:
       automated:
         prune: true      # Git 里删掉的资源，集群里也删掉
         selfHeal: true   # 有人手动改了集群，自动改回 Git 中的状态
       syncOptions:
         - CreateNamespace=true
   ```
4. `git add -A && git commit && git push`，然后 `kubectl apply -f clusters/local/hello.yaml`。
5. 做 4 个实验：
   | 实验 | 操作 | 预期 |
   |---|---|---|
   | A. Git 驱动变更 | 把 local overlay 的 replicas 改成 2，commit 并 push | 最多约 3 分钟（或在 UI 点 Refresh）后变成 2 个 Pod |
   | B. 自愈 | `kubectl scale deploy/hello -n hello --replicas=5` | 几秒内被改回 2 |
   | C. 清理（prune） | 删除 ingress.yaml 并从 kustomization 里移除，然后 push | 集群中的 Ingress 被删除 |
   | D. 回滚 | `git revert HEAD && git push` | Ingress 恢复 |

**为什么**
这是 GitOps 的**完整闭环**：所有变更都通过 Git 提交进入集群，集群被手动改动后会自动恢复（防止配置漂移），历史可以审计，回滚就是一次 `git revert`。Kustomize 的 base/overlays 结构是业界常用做法，参考项目也是这样组织的：环境差异放在 overlay，不复制 base。

**成功标准**
- `curl -s http://hello.localhost:8080` 返回包含 `Hello from local` 的页面（如果解析不了 `hello.localhost`，改用 `curl -H 'Host: hello.localhost' http://localhost:8080`）。
- `argocd app get hello` 显示 `Synced` 和 `Healthy`，`Sync Policy: Automated (Prune)`。
- 实验 A–D 的结果与表中预期一致；`argocd app history hello` 中能看到每次同步对应的 Git commit。

## 第 6 步：Helm 类型的应用（多源写法）

**做什么**
1. 写 `apps/podinfo/values-local.yaml`（例如 `replicaCount: 1`、`ui.message: "podinfo on k3d"`）。
2. 写 `clusters/local/podinfo.yaml`：
   ```yaml
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: podinfo
     namespace: argocd
   spec:
     project: default
     sources:
       - repoURL: https://stefanprodan.github.io/podinfo
         chart: podinfo
         targetRevision: 6.15.0
         helm:
           valueFiles:
             - $values/apps/podinfo/values-local.yaml
       - repoURL: https://github.com/GreatTyrion/try-argo-cd.git
         targetRevision: HEAD
         ref: values          # 让上面的 $values 指向本仓库根目录
     destination:
       server: https://kubernetes.default.svc
       namespace: podinfo
     syncPolicy:
       automated: { prune: true, selfHeal: true }
       syncOptions: [CreateNamespace=true]
   ```
3. 在本地预览：`helm template podinfo podinfo --repo https://stefanprodan.github.io/podinfo --version 6.15.0 -f apps/podinfo/values-local.yaml | head`。
4. push 后执行 `kubectl apply -f clusters/local/podinfo.yaml`。

**为什么**
很多第三方软件（监控、数据库、operator）都以 Helm chart 的形式发布。**多源 Application** 可以直接引用上游 chart，而 values 文件放在自己的 Git 仓库里：既不用复制 chart，又能让配置留在 Git 中审计。参考项目的 image-updater 就是这种写法。注意：Argo CD 用 `helm template` 渲染 chart 再自己 apply，所以 `helm list` 里**看不到**这个应用，这是正常的。

**成功标准**
- `argocd app get podinfo` 显示 `Synced` 和 `Healthy`，Sources 中有两条。
- `kubectl port-forward -n podinfo svc/podinfo 9898:9898`，然后 `curl -s localhost:9898 | grep message`，输出 `"message": "podinfo on k3d"`。
- 修改 values 中的 `ui.message` 并 push，几分钟后 curl 的结果跟着变化。

## 第 7 步：App of Apps，并让 Argo CD 管理自己

**做什么**
1. 把 Argo CD 的安装改写成 Kustomize（与参考项目做法一致）：
   ```yaml
   # bootstrap/argocd/base/kustomization.yaml
   apiVersion: kustomize.config.k8s.io/v1beta1
   kind: Kustomization
   namespace: argocd
   resources:
     - namespace.yaml
     - https://raw.githubusercontent.com/argoproj/argo-cd/v3.5.5/manifests/install.yaml
   ```
   `bootstrap/argocd/overlays/local/kustomization.yaml` 引用 `../../base`，并打一个 `argocd-cm` 补丁：`timeout.reconciliation: 60s`（本地缩短轮询间隔，反馈更快。这是一个环境差异的例子）。
2. 新增 `clusters/local/argocd.yaml`，让 Argo CD 管理自己：
   ```yaml
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: argocd
     namespace: argocd
   spec:
     project: default
     source:
       repoURL: https://github.com/GreatTyrion/try-argo-cd.git
       targetRevision: HEAD
       path: bootstrap/argocd/overlays/local
     destination:
       server: https://kubernetes.default.svc
       namespace: argocd
     syncPolicy:
       automated:
         prune: false      # 对 Argo CD 自身保守一些：不自动删除资源
         selfHeal: true
       syncOptions:
         - ServerSideApply=true   # CRD 太大，必须用 server-side apply
   ```
3. 新增根应用 `bootstrap/root-local.yaml`（只需手动 apply 一次）：
   ```yaml
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: root
     namespace: argocd
   spec:
     project: default
     source:
       repoURL: https://github.com/GreatTyrion/try-argo-cd.git
       targetRevision: HEAD
       path: clusters/local
       directory:
         recurse: true
     destination:
       server: https://kubernetes.default.svc
       namespace: argocd
     syncPolicy:
       automated: { prune: true, selfHeal: true }
   ```
4. push 后执行：`kubectl apply --server-side -k bootstrap/argocd/overlays/local`，然后 `kubectl apply -f bootstrap/root-local.yaml`。

**为什么**
- **App of Apps**：只需手动创建一个根 Application，它管理的是「其他 Application 的 YAML」。以后新增应用，只要往 `clusters/local/` 放一个文件再 push，不用再碰 kubectl。这是官方文档推荐的集群引导（cluster bootstrapping）方式。
- **Argo CD 管理自己**：升级 Argo CD 只要把 `v3.5.5` 改成新版本再 push；改配置也走 Git 审核。参考项目**没有做**这一步（它需要手动重新 apply），这里是一个改进。
- 前面手动 apply 的 hello、podinfo Application 会被 root **接管**（名字相同），不会重建。

**成功标准**
- `argocd app list` 中有 `root`、`argocd`、`hello`、`podinfo`，全部 `Synced` 和 `Healthy`。
- UI 中 root 的资源树下挂着另外三个 Application。
- `kubectl get cm argocd-cm -n argocd -o jsonpath='{.data.timeout\.reconciliation}'` 输出 `60s`（说明 Argo CD 已经在用 Git 中的配置管理自己）。
- 删除测试：`kubectl delete application hello -n argocd`，几秒后 root 会把它重新创建出来。

## 第 8 步：AppProject、ApplicationSet 和同步顺序

**做什么**
1. **AppProject**（权限边界），文件 `clusters/local/projects/learning.yaml`：
   ```yaml
   apiVersion: argoproj.io/v1alpha1
   kind: AppProject
   metadata:
     name: learning
     namespace: argocd
   spec:
     sourceRepos:
       - https://github.com/GreatTyrion/try-argo-cd.git
       - https://stefanprodan.github.io/podinfo
     destinations:
       - server: https://kubernetes.default.svc
         namespace: hello*
       - server: https://kubernetes.default.svc
         namespace: podinfo
     clusterResourceWhitelist:
       - group: ""
         kind: Namespace
   ```
   把 podinfo 的 `project` 改成 `learning`。`argocd` 和 `root` 这两个 Application 仍然用 `default` 项目，因为它们需要管理集群级资源。
2. **ApplicationSet**：删除 `clusters/local/hello.yaml`，同一个 commit 里新增 `clusters/local/apps-appset.yaml`：
   ```yaml
   apiVersion: argoproj.io/v1alpha1
   kind: ApplicationSet
   metadata:
     name: apps
     namespace: argocd
   spec:
     goTemplate: true
     goTemplateOptions: ["missingkey=error"]
     generators:
       - git:
           repoURL: https://github.com/GreatTyrion/try-argo-cd.git
           revision: HEAD
           directories:
             - path: apps/*/overlays/local
     template:
       metadata:
         name: '{{index .path.segments 1}}'        # apps/<名字>/overlays/local → <名字>
       spec:
         project: learning
         source:
           repoURL: https://github.com/GreatTyrion/try-argo-cd.git
           targetRevision: HEAD
           path: '{{.path.path}}'
         destination:
           server: https://kubernetes.default.svc
           namespace: '{{index .path.segments 1}}'
         syncPolicy:
           automated: { prune: true, selfHeal: true }
           syncOptions: [CreateNamespace=true]
   ```
   然后复制一份应用试试：`cp -r apps/hello apps/hello2`，把 `apps/hello2` 中 Ingress 的主机名改成 `hello2.localhost`，再 push。
3. **同步顺序**：在 `apps/hello/base/` 中加一个 PreSync hook（模拟数据库迁移），并给资源设置 wave：
   ```yaml
   # apps/hello/base/presync-job.yaml
   apiVersion: batch/v1
   kind: Job
   metadata:
     generateName: hello-migrate-
     annotations:
       argocd.argoproj.io/hook: PreSync
       argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
   spec:
     template:
       spec:
         restartPolicy: Never
         containers:
           - name: migrate
             image: busybox:1.38
             command: ["sh", "-c", "echo running migration; sleep 5"]
   ```
   再给 Service 加 `argocd.argoproj.io/sync-wave: "0"`，给 Deployment 加 `argocd.argoproj.io/sync-wave: "1"`。
4. **验证项目限制**：临时把 podinfo 的 destination namespace 改成 `kube-system` 再 push，观察报错，然后 `git revert`。
5. （可选）私有仓库：把仓库改成 private，用 `argocd repo add https://github.com/GreatTyrion/try-argo-cd.git --username GreatTyrion --password <fine-grained PAT>` 添加凭证，观察应用恢复正常。练习完改回 public，或者保留凭证。

**为什么**
- **AppProject** 就像团队的权限边界：限制「能从哪些仓库部署、能部署到哪些 namespace、能创建哪些集群级资源」。生产环境里，`default` 项目不应该给业务应用用。
- **ApplicationSet** 用一个模板加生成器批量产生 Application。新增应用只要建一个目录，不用再写 Application YAML。参考项目用它把 holmes 系列服务部署到 EKS 集群。
- 删掉 `hello.yaml` 时，旧的 hello Application 没有设置级联删除的 finalizer，所以工作负载不会被删除。新生成的同名 Application 会直接接管这些资源。
- **Sync phase、wave 和 hook** 解决部署顺序问题，例如「先迁移数据库，再启动新版本」。参考项目里 Hasura 和 Alembic 的 PreSync hook 就是这个用途。

**成功标准**
- `argocd proj get learning` 能看到上面配置的仓库和目标 namespace。
- `argocd appset get apps` 显示生成了 `hello`、`hello2`；`kubectl get pods -n hello2` 有 Running 的 Pod；`curl -H 'Host: hello2.localhost' http://localhost:8080` 返回页面。
- 每次同步 hello 时，UI 的同步结果里能看到 `hello-migrate-xxxx` Job 先执行（PreSync 阶段，Succeeded），然后是 Service（wave 0），最后是 Deployment（wave 1）。
- 改成 `kube-system` 后，`argocd app get podinfo` 的 Conditions 中出现 `InvalidSpecError ... is not permitted in project 'learning'`；revert 后恢复 Healthy。

## 第 9 步：用 SOPS + age + KSOPS 管理密钥

**做什么**
1. 生成 age 密钥（**私钥只放在本机，绝不能提交**）：
   ```bash
   mkdir -p ~/.config/sops/age
   age-keygen -o ~/.config/sops/age/keys.txt     # 输出中有 Public key: age1...
   export SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt   # 写进 ~/.zshrc
   ```
2. 在仓库根目录新建 `.sops.yaml`：
   ```yaml
   creation_rules:
     - path_regex: .*-unencrypted\.yaml$
       encrypted_regex: ^(data|stringData)$
       age: age1xxxxxxxx...    # 换成你的公钥
   ```
3. 把私钥放进集群（**必须先于下面的 repo-server 补丁**，否则 Pod 会因为找不到 Secret 而起不来）：
   ```bash
   kubectl create secret generic sops-age -n argocd \
     --from-file=keys.txt=$HOME/.config/sops/age/keys.txt
   ```
4. 在 `bootstrap/argocd/base/` 增加两个补丁（写法参照参考项目的 `argocd-repo-server-ksops-patch.yaml`，再加上 age 相关配置）：
   - `argocd-cm-patch.yaml`：`kustomize.buildOptions: --enable-alpha-plugins --enable-exec`
   - `argocd-repo-server-ksops-patch.yaml`：用 initContainer `viaductoss/ksops:v4.5.1` 执行 `ksops install --with-kustomize /custom-tools`，把 kustomize 和 ksops 挂载到 `/usr/local/bin/`；设置环境变量 `XDG_CONFIG_HOME=/.config`、`SOPS_AGE_KEY_FILE=/.config/sops/age/keys.txt`；再把 Secret `sops-age` 挂载到 `/.config/sops/age`。
5. 加密一个 Secret 并接入 hello：
   ```bash
   # apps/hello/overlays/local/hello-secret-unencrypted.yaml（已被 .gitignore 排除）
   #   kind: Secret, metadata.name: hello-secret, stringData: { token: my-local-token }
   sops --encrypt apps/hello/overlays/local/hello-secret-unencrypted.yaml \
     > apps/hello/overlays/local/hello-secret-sops.yaml
   ```
   新建 `secret-generator.yaml`（`apiVersion: viaduct.ai/v1`、`kind: ksops`、`files: [./hello-secret-sops.yaml]`，并加上 `config.kubernetes.io/function` 的 exec 注解），然后在 overlay 的 kustomization 中加入 `generators: [./secret-generator.yaml]`。
6. push。Argo CD 会先同步自己（repo-server 重启并装上 KSOPS），然后同步 hello。

**为什么**
Git 里不能放明文密钥，但 GitOps 又要求「一切都在 Git 中」。SOPS 只加密 `data/stringData` 的**值**，键名和结构仍然可读，方便 code review。KSOPS 让 repo-server 在渲染时解密，明文只存在于集群内部。参考项目用的是同一套机制，只是把 age 换成了 GCP KMS（密钥由云端托管，通过 Workload Identity 授权）。这里用 age，免费且不依赖云。

**成功标准**
- `git show HEAD:apps/hello/overlays/local/hello-secret-sops.yaml | grep token` 显示的是 `ENC[AES256_GCM,...]`，而不是明文。
- `git check-ignore apps/hello/overlays/local/hello-secret-unencrypted.yaml` 有输出（说明已被忽略）。
- `sops -d apps/hello/overlays/local/hello-secret-sops.yaml` 能在本机解密。
- `kubectl get secret hello-secret -n hello -o jsonpath='{.data.token}' | base64 -d` 输出 `my-local-token`。
- `kubectl get pod -n argocd -l app.kubernetes.io/name=argocd-repo-server -o jsonpath='{.items[0].spec.initContainers[0].image}'` 输出 `viaductoss/ksops:v4.5.1`。

## 第 10 步：从零一键重建本地环境

**做什么**
编写 `scripts/bootstrap-local.sh`（需要 `set -euo pipefail`，每一步打印日志）：
1. `k3d cluster create --config local/k3d-config.yaml`
2. `kubectl create namespace argocd`
3. `kubectl create secret generic sops-age ...`（从 `$SOPS_AGE_KEY_FILE` 读取）
4. `kubectl apply --server-side --force-conflicts -k bootstrap/argocd/overlays/local`
5. `kubectl wait --for=condition=available deployment --all -n argocd --timeout=300s`
6. `kubectl apply -f bootstrap/root-local.yaml`
7. 循环等待，直到 `kubectl get applications -n argocd` 中所有应用都是 Synced/Healthy，最多等 10 分钟

再编写 `scripts/teardown-local.sh`（`k3d cluster delete argocd-lab`）。然后实际测试一次：
```bash
scripts/teardown-local.sh && time scripts/bootstrap-local.sh
```

**为什么**
这一步检验前面的成果：集群是「牲口」不是「宠物」，可以随时丢掉。只要 Git 仓库加上**一个私钥**，就能完整重建整个环境。这也是灾难恢复和创建新环境的基础。第 13 步在 GKE 上会用完全相同的流程。

**成功标准**
- 脚本退出码为 0，`time` 显示的耗时 ≤ 10 分钟。
- `argocd app list`（重新 port-forward 并登录后）中所有应用都是 `Synced` 和 `Healthy`。
- `curl -s http://hello.localhost:8080` 和 hello-secret 的检查结果与第 5、9 步一致。
- 整个过程中**除了脚本之外没有任何手动 kubectl 操作**。

---

# 阶段 2：云端部署（GCP / GKE，预算 ≤ $20）

> 先读一遍 [cost-and-cleanup.md](cost-and-cleanup.md)。原则：**学完一次就销毁集群，需要时再创建**。

## 第 11 步：准备 GCP 项目和预算提醒

**做什么**
```bash
brew install --cask google-cloud-sdk
brew tap hashicorp/tap && brew install hashicorp/tap/terraform
gcloud components install gke-gcloud-auth-plugin

gcloud auth login
gcloud auth application-default login          # 给 Terraform 用

export PROJECT_ID=argocd-lab-$RANDOM            # 项目 ID 必须全局唯一
gcloud projects create $PROJECT_ID
gcloud billing accounts list                    # 记下 ACCOUNT_ID
gcloud billing projects link $PROJECT_ID --billing-account=<ACCOUNT_ID>
gcloud config set project $PROJECT_ID
gcloud services enable container.googleapis.com compute.googleapis.com billingbudgets.googleapis.com

gcloud billing budgets create \
  --billing-account=<ACCOUNT_ID> \
  --display-name="argocd-lab budget" \
  --budget-amount=20USD \
  --filter-projects=projects/$PROJECT_ID \
  --threshold-rule=percent=0.25 \
  --threshold-rule=percent=0.5 \
  --threshold-rule=percent=0.75 \
  --threshold-rule=percent=1.0
```

**为什么**
用一个**独立的项目**来学习：费用清楚，清理时一条 `gcloud projects delete` 就能删干净。预算提醒在 $5/$10/$15/$20 时发邮件，是防止超支的安全网（注意它只发通知，不会自动关停资源）。`gke-gcloud-auth-plugin` 是 kubectl 连接 GKE 的认证插件。application-default 凭证给 Terraform 使用。

**成功标准**
- `gcloud config get project` 输出你的 `$PROJECT_ID`。
- `gcloud billing projects describe $PROJECT_ID` 显示 `billingEnabled: true`。
- `gcloud billing budgets list --billing-account=<ACCOUNT_ID>` 能看到 `argocd-lab budget`，金额为 20 USD。
- `terraform version` 显示 `v1.16.x`；`gke-gcloud-auth-plugin --version` 有输出。

## 第 12 步：用 Terraform 创建 GKE（zonal Standard 集群 + Spot 节点）

**做什么**
在 `infra/terraform/gke/` 下编写：
- `versions.tf`：`required_version = ">= 1.16"`；`hashicorp/google` 版本 `~> 8.7`。
- `variables.tf`：`project_id`、`region = "us-central1"`、`zone = "us-central1-a"`、`cluster_name = "argocd-lab"`。
- `main.tf`：
  ```hcl
  provider "google" {
    project = var.project_id
    region  = var.region
  }

  # 最小权限的节点服务账号（不用 Compute Engine 默认账号）
  resource "google_service_account" "nodes" {
    account_id   = "gke-nodes"
    display_name = "GKE nodes (least privilege)"
  }

  resource "google_project_iam_member" "nodes" {
    project = var.project_id
    role    = "roles/container.defaultNodeServiceAccount"
    member  = "serviceAccount:${google_service_account.nodes.email}"
  }

  resource "google_container_cluster" "lab" {
    name     = var.cluster_name
    location = var.zone              # 单个 zone → 管理费可以用免费额度抵扣

    remove_default_node_pool = true  # 默认节点池创建后立刻删除，改用下面单独管理的节点池
    initial_node_count       = 1
    deletion_protection      = false # 学习用，方便销毁

    release_channel { channel = "REGULAR" }
    workload_identity_config { workload_pool = "${var.project_id}.svc.id.goog" }
    logging_config    { enable_components = ["SYSTEM_COMPONENTS"] }   # 只收系统日志，控制费用
    monitoring_config { enable_components = ["SYSTEM_COMPONENTS"] }
  }

  resource "google_container_node_pool" "spot" {
    name       = "spot-pool"
    cluster    = google_container_cluster.lab.id
    node_count = 1

    node_config {
      spot            = true
      machine_type    = "e2-standard-2"   # 2 vCPU / 8 GB
      disk_type       = "pd-standard"
      disk_size_gb    = 30
      service_account = google_service_account.nodes.email
      oauth_scopes    = ["https://www.googleapis.com/auth/cloud-platform"]
    }

    management {
      auto_repair  = true
      auto_upgrade = true
    }
  }
  ```
- `outputs.tf`：输出 `cluster_name`、`zone`，以及 `get-credentials` 命令。
- `terraform.tfvars`（已被 gitignore）：`project_id = "<你的 PROJECT_ID>"`。

```bash
cd infra/terraform/gke
terraform init
terraform fmt -check && terraform validate
terraform plan -out=tfplan        # 仔细检查：应该只新建 4 个资源
terraform apply tfplan            # 大约 8–10 分钟
gcloud container clusters get-credentials argocd-lab --zone us-central1-a
kubectl config rename-context gke_${PROJECT_ID}_us-central1-a_argocd-lab gke-argocd-lab
```

**为什么**
- **Terraform** 把基础设施也写成代码，与参考项目一致。`terraform destroy` 能一次删干净，方便「按需创建、用完就删」。
- **zonal Standard 集群**：管理费可以由免费额度全额抵扣。Autopilot 按 Pod 计费，跑 Argo CD 每月约 $70，超预算（详见 cost-and-cleanup.md）。
- **Spot 节点**：价格大约是普通节点的 30–50%，代价是 Google 随时可能回收。学习环境完全可以接受，还能顺便观察 Argo CD 在节点被回收后如何自动恢复。
- 用**单独管理的节点池**，加上**最小权限的服务账号**，这是 Terraform 官方文档和 GKE 安全加固指南推荐的做法。
- state 文件存在本地，并已被 gitignore（单人学习足够）。团队协作时应该改用 GCS 后端存储 state。

**成功标准**
- `terraform plan` 显示 `Plan: 4 to add, 0 to change, 0 to destroy`。
- `gcloud container clusters list` 中 LOCATION 列为 `us-central1-a`（是一个 zone，不是 region），STATUS 为 `RUNNING`。
- `kubectl --context gke-argocd-lab get nodes` 显示 1 个节点 `Ready`；`kubectl get nodes -L cloud.google.com/gke-spot` 中该列为 `true`。
- `gcloud container node-pools list --cluster argocd-lab --zone us-central1-a` 中只有 `spot-pool`（默认节点池已删除）。

## 第 13 步：在 GKE 上引导 Argo CD

**做什么**
1. 新建 `bootstrap/argocd/overlays/gke/kustomization.yaml`，引用 `../../base`（已包含 KSOPS），并加上 GKE 环境的差异：把 `argocd-dex-server` 的 replicas 打补丁改成 0（只用 admin 账号，省一些内存）。
2. 新建 `bootstrap/root-gke.yaml`（与 root-local.yaml 相同，只是 path 改为 `clusters/gke`）。
3. 新建 `clusters/gke/`：
   - `argocd.yaml`：path 为 `bootstrap/argocd/overlays/gke`
   - `projects/learning.yaml`：与 local 相同
   - `apps-appset.yaml`：directories 改为 `apps/*/overlays/gke`
   - `podinfo.yaml`：valueFiles 改为 `$values/apps/podinfo/values-gke.yaml`
4. push 后执行（与第 10 步的流程完全相同，只是换了 context）：
   ```bash
   kubectl --context gke-argocd-lab create namespace argocd
   kubectl --context gke-argocd-lab create secret generic sops-age -n argocd \
     --from-file=keys.txt=$SOPS_AGE_KEY_FILE
   kubectl --context gke-argocd-lab apply --server-side --force-conflicts -k bootstrap/argocd/overlays/gke
   kubectl --context gke-argocd-lab wait --for=condition=available deployment --all -n argocd --timeout=600s
   kubectl --context gke-argocd-lab apply -f bootstrap/root-gke.yaml
   kubectl --context gke-argocd-lab port-forward svc/argocd-server -n argocd 9443:443
   ```
   （可以把这些命令提炼成 `scripts/bootstrap-gke.sh`。）

**为什么**
本地和云端用**同一个仓库、同一个 base**，只有 overlay 不同，这就是「多环境靠 overlay、不复制配置」的实践。引导流程和本地完全一样，证明第 10 步的自动化可以直接用于新环境。UI 继续用 port-forward 访问（端口改成 9443，避免和本地的 8443 冲突），不创建任何付费的负载均衡器。

**成功标准**
- `kubectl --context gke-argocd-lab get pods -n argocd` 全部 Running；`argocd-dex-server` 的副本数为 0。
- 浏览器访问 `https://localhost:9443` 能登录（用 `argocd admin initial-password -n argocd` 获取初始密码，改密码后删除初始 Secret）。
- `argocd app list` 中 root、argocd、podinfo 都是 Synced/Healthy（hello 在第 14 步加上 gke overlay 后才会出现）。

## 第 14 步：把应用部署到 GKE

**做什么**
1. 新建 `apps/hello/overlays/gke/`：replicas 改为 2，页面内容 `Hello from GKE`，**不加 Ingress**。如果要用 SOPS Secret，就用同样的方法生成 `hello-secret-sops.yaml`，内容写 GKE 专用的值。
2. 新建 `apps/podinfo/values-gke.yaml`（例如 `replicaCount: 2`、`ui.message: "podinfo on GKE"`）。
3. push，然后观察 ApplicationSet 自动生成 `hello` 应用。
4. （可选，演示完马上删除）临时给 gke overlay 加一个补丁，把 Service 改成 `type: LoadBalancer`，push 后拿到 EXTERNAL-IP 并用 curl 访问。然后 `git revert` 并 push，确认负载均衡器已经被删除。
5. 观察 Spot 节点被回收的情况（遇到了就看看，没遇到不用强求）：`kubectl get events -A --field-selector reason=Preempted`。

**为什么**
同一个应用、两个环境，差异只在 overlay 中，这是多环境发布的标准做法。可选的 LoadBalancer 实验演示「云资源也由 Git 控制，revert 后自动释放」，同时说明为什么平时不开它（按小时计费）。参考项目用 GCP KMS + Workload Identity 来解密，不需要手动放私钥。这里为了省钱和简单，手动注入 age 私钥，两种方式各有取舍，记下来作为对比学习。

**成功标准**
- `kubectl --context gke-argocd-lab get deploy hello -n hello` 显示 `2/2`。
- `kubectl --context gke-argocd-lab port-forward -n hello svc/hello 8081:80`，然后 `curl -s localhost:8081` 返回 `Hello from GKE`。
- 本地集群的 `curl http://hello.localhost:8080` 仍然返回 `Hello from local`（两个环境互不影响）。
- （可选）revert 之后，`gcloud compute forwarding-rules list` 返回空。

## 第 15 步（可选）：多集群，一个 Argo CD 管理两个集群

**做什么**
```bash
# CLI 登录本地的 Argo CD（localhost:8443），然后：
argocd cluster add gke-argocd-lab --name gke
argocd cluster list
```
在 `clusters/local/` 中加一个 Application，`project: default`（learning 项目只允许 in-cluster），destination 写 `name: gke`、`namespace: hello-remote`，path 为 `apps/hello/overlays/gke`，并开启 `CreateNamespace=true`。观察本地 Argo CD 把应用部署到 GKE。实验结束后，删除这个 Application 并执行 `argocd cluster rm gke`。

**为什么**
这正是参考项目的架构：c-core-autopilot 上的 Argo CD 作为**控制中心**，管理 iceland（GKE）和 dalo（EKS）等远程集群。`argocd cluster add` 会在远程集群中创建一个 ServiceAccount，并把它的 token 保存为 Argo CD 的集群凭证。参考项目改用 Workload Identity（`argocd-k8s-auth gcp`）来认证，避免长期有效的 token，安全性更高。

**成功标准**
- `argocd cluster list` 中 `gke` 的 STATUS 为 `Successful`。
- `kubectl --context gke-argocd-lab get pods -n hello-remote` 中有 Running 的 Pod，而且它是由**本地** Argo CD 创建的（本地 UI 中能看到这个应用）。
- 清理后，`argocd cluster list` 中只剩 `in-cluster`。

## 第 16 步：清理，避免产生额外费用

**做什么**
按照 [cost-and-cleanup.md 第 3 节](cost-and-cleanup.md#3-清理清单)执行：
1. 确认没有 LoadBalancer Service 和 PVC。
2. `cd infra/terraform/gke && terraform destroy`（⚠️ 会删除整个集群）。
3. 运行全部 `gcloud ... list` 检查命令。
4. 全部学完后：`gcloud projects delete $PROJECT_ID`（⚠️ 删除整个项目）。
5. 本地：`scripts/teardown-local.sh`。
6. 第二天查看账单报表。

**为什么**
云上的费用只要资源还在就会一直产生。只删 Kubernetes 对象不够，可能会留下磁盘、负载均衡器、静态 IP 等「孤儿资源」。删除整个项目是最彻底、最不容易遗漏的方法。用 Terraform 创建的资源，也应该用 Terraform 删除，避免 state 和实际资源不一致。

**成功标准**
- `terraform destroy` 结束时显示 `Destroy complete! Resources: 4 destroyed.`。
- `gcloud container clusters list`、`compute instances list`、`compute disks list`、`compute addresses list`、`compute forwarding-rules list` 全部返回空。
- `gcloud projects describe $PROJECT_ID` 显示 `lifecycleState: DELETE_REQUESTED`（如果删除了项目）。
- `k3d cluster list` 中没有 `argocd-lab`。
- 第二天的账单报表中，该项目当日费用为 $0，累计费用 ≤ $20。

---

## 学完之后可以继续探索
- **Argo CD Image Updater**：镜像有新版本时自动更新 Git（参考项目在用）。
- **Notifications**：同步失败时发消息到 Slack。
- **SSO（Dex/OIDC）+ RBAC**：给团队成员分配不同权限。
- **Argo Rollouts**：金丝雀发布、蓝绿发布。
- **Workload Identity + GCP KMS**：用云端托管的密钥代替手动注入的 age 私钥（参考项目的做法）。
