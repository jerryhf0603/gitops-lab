# Phase 3：AppProject 權限邊界

目標：離開 `project: default`，用 **AppProject** 限制「誰可以部署到哪、能用哪些 repo / 資源」。

預計時間：1～2 小時

---

## 3.1 Application vs AppProject

| 概念 | 角色 |
|------|------|
| Application | 單一部署單位（某個 Git path → 某個 namespace） |
| AppProject | **一組 Application 的圍籬**（允許的 repo、namespace、資源類型） |

沒有 AppProject 時大家都塞在 `default`，等於幾乎不設限。正式環境一定要拆 project。

---

## 3.2 本 lab 要做的事

```text
1. 建立 AppProject: demo
2. 把 demo-web-dev / uat / prod 改成 project: demo
3. （可選）故意寫錯 destination，確認被擋
```

檔案：

```text
projects/demo.yaml              ← AppProject 定義
applications/demo-web-*.yaml    ← project: demo
```

---

## 3.3 實作步驟

### Step 1：確認 context

```bash
kubectl config use-context k3d-argocd-lab
```

### Step 2：建立 AppProject

```bash
kubectl apply -f projects/demo.yaml
kubectl get appprojects -n argocd
argocd proj get demo
```

### Step 3：把三個子 Application 改成 `project: demo`

編輯 `applications/demo-web-dev.yaml`、`demo-web-uat.yaml`、`demo-web-prod.yaml`：

```yaml
spec:
  project: demo   # 從 default 改成 demo
```

`root` 可以暫時留在 `default`（它負責管 Application 清單）。

### Step 4：commit + push

```bash
git add projects/ applications/
git commit -m "新增: AppProject demo 並遷移子 Application"
git push
```

因為你有 App of Apps（`root` 自動 sync），子 Application 的 `project` 欄位會被 root 更新。

### Step 5：驗證

```bash
argocd app get demo-web-dev
argocd app get demo-web-uat
argocd app get demo-web-prod
```

應看到 `Project: demo`。

---

## 3.4 實驗 A：故意踩線（確認會被擋）

在某個 Application（建議用 **dev**）暫時改：

```yaml
destination:
  namespace: kube-system   # 不在 demo project 允許清單
```

push 後觀察：

```bash
argocd app get demo-web-dev
```

預期：Sync 失敗或報錯，訊息類似 **destination namespace is not permitted**。

做完 **改回 `demo-dev`** 並 push。

### Checklist

- [ ] 錯誤 destination 被 AppProject 擋下
- [ ] 改回後恢復 Synced / Healthy

---

## 3.5 實驗 B：（可選）錯誤 repo

暫時把 `repoURL` 改成別的 GitHub repo，push。

預期：被 `sourceRepos` 擋下。

做完改回。

---

## 3.6 過關標準

- [ ] 存在 AppProject `demo`
- [ ] 三個 `demo-web-*` 的 `project` 都是 `demo`
- [ ] 能說出 AppProject 擋的三件事：destination / sourceRepos / resource kinds
- [ ]（建議）完成實驗 A

---

## 核心一句話

> **AppProject = Application 的 ACL：限制能從哪裡部署、部署到哪裡、能建什麼資源。**

---

## 下一步

Phase 3 過關後 → Phase 4：ApplicationSet（用一份模板產生多環境 Application）
