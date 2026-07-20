# Phase 0：從 0 認識 Argo CD（GitOps 基礎）

目標：能口述並實際驗證這條链路：

```text
改 Git → push → Argo CD 偵測 → Helm 渲染 → Sync → 叢集 Live 狀態對齊
```

預計時間：半天～1 天

---

## 0.1 環境檢查

確認本機有這些工具：

```bash
kubectl version --client
helm version
argocd version --client
git --version
```

確認目前 kube context（之後所有指令都打在這個叢集）：

```bash
kubectl config get-contexts
kubectl config current-context
kubectl get nodes
```

本 lab 使用 **k3d** 叢集 `argocd-lab`。若 context 不是它，先切換：

```bash
# 若叢集是停的
k3d cluster start argocd-lab

# 切到正確 context（很重要：不要打到公司/其他叢集）
kubectl config use-context k3d-argocd-lab
kubectl get nodes
```

> 本 lab 假設：單叢集（k3d）、本機可 `kubectl` 存取。

### Checklist

- [ ] kubectl / helm / argocd CLI 可用
- [ ] `kubectl config current-context` 顯示 `k3d-argocd-lab`
- [ ] 能成功 `kubectl get nodes`

---

## 0.2 安裝／確認 Argo CD

### 若尚未安裝

```bash
kubectl create namespace argocd

kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 若先前已裝好（你的情況）

跳過 apply，直接確認 Pod：

```bash
kubectl get pods -n argocd
kubectl get applications -n argocd
```

等 Pod Ready（若剛 start 叢集，可能要等一會）：

```bash
kubectl get pods -n argocd -w
```

取得初始 admin 密碼：

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
echo
```

開 UI（另開一個終端機，保持執行）：

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

瀏覽器開啟：`https://localhost:8080`

- Username: `admin`
- Password: 上一步拿到的字串

用 CLI 登入（略過憑證驗證，僅限 lab）：

```bash
argocd login localhost:8080 --username admin --insecure
```

### Checklist

- [ ] context 是 `k3d-argocd-lab`
- [ ] `argocd` namespace 存在，核心 Pod Running
- [ ] UI 可登入
- [ ] CLI `argocd login` 成功
---

## 0.3 必懂概念（先讀再動手）

| 名詞 | 意思 |
|------|------|
| Desired State | Git 裡宣告「應該長怎樣」 |
| Live State | 叢集目前真實狀態 |
| Application | Argo CD 的部署單位：某個 Git path → 某個 namespace |
| Sync | 把 Live 對齊 Desired |
| OutOfSync | Git 與叢集不一致 |
| Synced | 已對齊 |
| Healthy / Degraded | 資源健康狀態（和 Synced 不同） |
| prune | Sync 時刪掉 Git 已移除、但叢集還在的資源 |
| selfHeal | 有人手動改叢集時，自動改回 Git 定義 |

你現有 repo 的對應關係：

```text
Git repo (jerryhf0603/gitops-lab)
├── charts/demo-web/              ← Helm 模板（Desired 的長相）
├── environments/dev/values.yaml  ← 環境參數
└── applications/demo-web-dev.yaml ← 告訴 Argo CD：用哪個 chart + values，部署到哪
```

---

## 0.4 註冊 Git 倉庫

### 公開 repo（目前你的 lab）

若 repo 是 public，通常可不建 credential，直接在 Application 寫 `repoURL`。

### 若之後改 private

UI：Settings → Repositories → Connect Repo  
或 CLI：

```bash
argocd repo add https://github.com/jerryhf0603/gitops-lab.git \
  --username <user> \
  --password <token>
```

### Checklist

- [ ] 知道 Application 的 `repoURL` 指向哪個 Git
- [ ] （若 private）repo 已連上且狀態正常

---

## 0.5 建立第一個 Application（dev）

先只做 **dev**，不要一次上三個環境。

```bash
kubectl apply -f applications/demo-web-dev.yaml
```

觀察：

```bash
kubectl get applications -n argocd
argocd app get demo-web-dev
argocd app diff demo-web-dev
```

若尚未自動 sync，手動：

```bash
argocd app sync demo-web-dev
```

驗證資源：

```bash
kubectl get all -n demo-dev
kubectl get deploy,svc -n demo-dev
```

用 Helm 角度看「Argo CD 實際渲染結果」（本機模擬）：

```bash
helm template demo-web-dev ./charts/demo-web \
  -f environments/dev/values.yaml
```

### Checklist

- [ ] Application `demo-web-dev` 狀態為 Synced / Healthy
- [ ] `demo-dev` namespace 有 Deployment + Service
- [ ] 理解 `applications/*.yaml` 與 `charts/`、`environments/` 的分工

---

## 0.6 實驗 A：改 Git → 看 Auto Sync

1. 編輯 `environments/dev/values.yaml`，把 `replicaCount` 從 `1` 改成 `2`
2. commit + push 到 `main`
3. 在 Argo CD UI 看 `demo-web-dev`：出現 Diff → 自動 Sync
4. 確認：

```bash
kubectl get deploy -n demo-dev
# READY 應變成 2/2
```

### 觀察重點

- UI 的 APP DIFF 顯示什麼變了？
- Sync 是自動還是要按按鈕？（你的 YAML 有 `automated`）

### Checklist

- [ ] push 後不需手動 kubectl，replicas 已變

---

## 0.7 實驗 B：selfHeal（有人手動改叢集）

```bash
kubectl scale deploy -n demo-dev --replicas=5 \
  -l app.kubernetes.io/name=demo-web
```

若 label 不確定，先：

```bash
kubectl get deploy -n demo-dev --show-labels
```

然後 scale 該 deployment 名稱。

等待約 1～3 分鐘（或 UI 按 Refresh），觀察是否被拉回 Git 定義的 replicas。

```bash
kubectl get deploy -n demo-dev
argocd app get demo-web-dev
```

### Checklist

- [ ] 手動 scale 後，最終又回到 Git 的值
- [ ] 能解釋 selfHeal 與「禁止手動改正式環境」的關係

---

## 0.8 實驗 C：prune（Git 刪了，叢集也刪）

> 只在 lab 做。先確認 `syncPolicy.automated.prune: true`。

較安全的做法：暫時在 chart 註解掉 Service（或改成不會被需要的資源），push，看 Service 是否被刪掉；再還原。

或用這個較溫和的驗證：

1. UI 打開 Application → 記住目前資源清單
2. 暫時從 `templates/service.yaml` 整檔移到 repo 外（或改名不進 templates）
3. push → 看 Service 是否被 prune
4. 再還原檔案 → push → Service 回來

### Checklist

- [ ] 理解 prune=true / false 差在哪
- [ ] 做完後把 chart 還原，Application 回到 Healthy

---

## 0.9 實驗 D：Synced ≠ Healthy

刻意製造不健康（lab only）：

```bash
# 把 image 改成不存在的 tag（改 environments/dev/values.yaml）
# image.tag: "this-tag-does-not-exist-999"
# push 後觀察
```

預期：

- 可能仍顯示 Synced（Git 已套用）
- 但 Healthy 變成 Progressing / Degraded（拉映像失敗）

做完請改回 `1.27` 並 sync 恢復。

### Checklist

- [ ] 能區分 Synced（對齊 Git）與 Healthy（資源可用）

---

## 0.10 Phase 0 過關標準

全部勾完再進 Phase 1：

- [ ] 能自己安裝並登入 Argo CD
- [ ] 能說出 Application / Sync / OutOfSync / prune / selfHeal
- [ ] `demo-web-dev` 已用 GitOps 方式跑起來
- [ ] 完成實驗 A（改 Git）
- [ ] 完成實驗 B（selfHeal）
- [ ] 完成實驗 C（prune）或至少能口述流程
- [ ] 完成實驗 D（Synced vs Healthy）

### 一句話自測

> 「我不碰 kubectl apply 應用清單，只改 Git，叢集就會變成我想要的狀態。」  
> 若這句你能用操作證明，Phase 0 過關。

---

## 常見問題

### Application 一直 OutOfSync

```bash
argocd app diff demo-web-dev
argocd app get demo-web-dev
```

常見原因：valueFiles 路徑錯、repo 權限、targetRevision 分支名不對。

### valueFiles 相對路徑

你現在寫的是：

```yaml
path: charts/demo-web
helm:
  valueFiles:
    - ../../environments/dev/values.yaml
```

這是相對 `path`（chart 目錄）往上找，不要改成以 repo root 為準的絕對感覺路徑。

### 刪不掉 Application

你有加 finalizer。刪除時 Argo CD 會先清資源。若卡住：

```bash
kubectl -n argocd patch application demo-web-dev \
  --type json \
  -p '[{"op":"remove","path":"/metadata/finalizers"}]'
```

（lab 救急用，先搞懂再使用。）

---

## 下一步

Phase 0 過關後 → [Phase 1：Application 行為精熟](./phase-1.md)（尚未建立時可先看 README 總覽）
