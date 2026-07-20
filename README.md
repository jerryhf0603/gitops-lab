# gitops-lab

用一個小專案，從 0 練習 **Argo CD + Helm + GitOps**。

Repo：`https://github.com/jerryhf0603/gitops-lab.git`

## 目錄說明

```text
apps/nginx/              # 早期練習：純 YAML（可保留當對照）
charts/demo-web/         # Helm Chart（應用模板）
environments/            # 各環境 values（dev / uat / prod）
applications/            # Argo CD Application 清單
docs/                    # 學習關卡說明
```

## 學習路線

| Phase | 主題 | 文件 | 狀態 |
|-------|------|------|------|
| 0 | 從 0 安裝與驗證 GitOps 闭环 | [docs/phase-0.md](./docs/phase-0.md) | **從這裡開始** |
| 1 | Application sync 行為（手動/自動、retry） | 待補 | 等 Phase 0 過關 |
| 2 | App of Apps | 待補 | |
| 3 | AppProject 權限邊界 | 待補 | |
| 4 | ApplicationSet | 待補 | |
| 5 | sync-wave / hooks | 待補 | |
| 6 | Image Updater / Notifications（選修） | 待補 | |

## 現在請做

環境：`k3d` 叢集 `argocd-lab`（Argo CD 已安裝可跳過 0.2 apply）。

```bash
k3d cluster start argocd-lab   # 若已 start 可略過
kubectl config use-context k3d-argocd-lab
kubectl get pods -n argocd
```

👉 打開 **[Phase 0](./docs/phase-0.md)**，從 **0.5 / 0.6** 實驗開始（改 Git、selfHeal、prune）。

> 注意：本機若有多個 kube context，務必先確認是 `k3d-argocd-lab`，不要打到其他叢集。
