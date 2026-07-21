# Phase 2：App of Apps

目標：**叢集裡只手動建立一個 root Application**，其餘 Application 全部由 Git 管理。

預計時間：1～2 天

---

## 2.1 為什麼需要 App of Apps？

你現在的做法：

```text
kubectl apply -f applications/demo-web-dev.yaml
kubectl apply -f applications/demo-web-uat.yaml
kubectl apply -f applications/demo-web-prod.yaml
```

問題：

- Application 本身不在 GitOps 闭环裡
- 新增環境要手動 apply
- 刪除 YAML 不會自動刪 Application

App of Apps 解法：

```text
只 apply 一次 root
    ↓
root 監聽 applications/ 目錄
    ↓
自動建立 / 更新 / 刪除 子 Application
```

---

## 2.2 目錄結構

```text
gitops-lab/
├── bootstrap/
│   └── root.yaml          ← 唯一需要手動 apply 的檔案
├── applications/
│   ├── demo-web-dev.yaml  ← 子 Application（由 root 管理）
│   ├── demo-web-uat.yaml
│   └── demo-web-prod.yaml
├── charts/demo-web/
└── environments/
```

---

## 2.3 建立 root Application

檔案已放在 `bootstrap/root.yaml`，重點欄位：

| 欄位 | 值 | 意思 |
|------|-----|------|
| `source.path` | `applications` | 掃描這個目錄下所有 YAML |
| `source.directory.recurse` | `false` | 不遞迴子目錄 |
| `destination.namespace` | `argocd` | 子資源是 Application CR，建在 argocd ns |
| `automated.prune` | `true` | Git 刪了子 YAML → 子 Application 也刪 |

---

## 2.4 實作步驟

### Step 1：確認 context

```bash
kubectl config use-context k3d-argocd-lab
kubectl get applications -n argocd
```

目前應有：`demo-web-dev`、`demo-web-uat`、`demo-web-prod`（可能還有 `nginx`）。

> `nginx` 不在 `applications/` 裡，root **不會**管理它。

### Step 2：commit + push bootstrap

```bash
cd /Users/user/Git/gitops-lab
git add bootstrap/root.yaml docs/
git commit -m "新增: App of Apps root Application"
git push
```

### Step 3：手動 apply root（整個 lab 最後一次手動 apply Application）

```bash
kubectl apply -f bootstrap/root.yaml
```

### Step 4：觀察

```bash
kubectl get applications -n argocd
argocd app get root
argocd app list
```

UI 上應看到：

- `root` Application
- `root` 底下有子 Application（dev / uat / prod）

### Step 5：驗證 root 在管理子 Application

改一個無害的東西測試，例如在 `applications/demo-web-dev.yaml` 加 annotation：

```yaml
metadata:
  annotations:
    lab.phase: "2"
```

push 後看 `root` 是否自動 sync，子 Application 是否更新。

---

## 2.5 實驗 A：Git 新增子 Application

在 `applications/` 新增一個測試用 Application（或之後用 ApplicationSet 取代）。

觀察 root sync 後是否自動出現新 Application。

---

## 2.6 實驗 B：Git 刪除子 Application（prune）

> lab only，確認你理解 prune 行為。

1. 暫時把 `applications/demo-web-dev.yaml` 從 Git 刪除（或改名移出目錄）
2. push
3. 觀察 `root` sync 後 `demo-web-dev` Application 是否被刪除
4. 因 finalizer，連 `demo-dev` namespace 裡的資源也會被清掉
5. **做完記得還原檔案並 push**

### Checklist

- [ ] 理解刪 YAML → root prune → 子 Application 消失 → 工作負載被清

---

## 2.7 常見問題

### root 和子 Application 都 OutOfSync？

先看 root 是否 Synced，再逐個看子 Application。

### 子 Application 重複或衝突？

若先前手動 apply 過同名 Application，root 通常會**接管**同名資源，一般不會重複建立。用 `argocd app get` 確認 owner 關係。

### bootstrap/root.yaml 要不要放進 applications/？

**不要。** 否則 root 會試圖管理自己，容易循環。bootstrap 永遠在 `applications/` 外面。

---

## 2.8 過關標準

- [ ] 只手動 apply 過 `bootstrap/root.yaml`
- [ ] `applications/` 裡的 YAML 變更會透過 root 自動反映
- [ ] 能說出 root 與子 Application 的關係
- [ ] 知道 `nginx` 為何不受 root 管理（不在目錄裡）

---

## 下一步

Phase 2 過關後 → Phase 3：AppProject 權限邊界
