# gitops-lab

用一個小專案，從 0 練習 **Argo CD + Helm + GitOps**。

Repo：`https://github.com/jerryhf0603/gitops-lab.git`

## 目錄說明

```text
bootstrap/               # root Application（唯一手動 apply）
applications/            # 子 Application（由 root 管理）
apps/nginx/              # 早期練習：純 YAML（可保留當對照）
charts/demo-web/         # Helm Chart（應用模板）
environments/            # 各環境 values（dev / uat / prod）
docs/                    # 學習關卡說明
```

## 學習路線

| Phase | 主題 | 文件 | 狀態 |
|-------|------|------|------|
| 0 | 從 0 安裝與驗證 GitOps 闭环 | [docs/phase-0.md](./docs/phase-0.md) | 完成 |
| 1 | Application sync 行為 | [docs/phase-1.md](./docs/phase-1.md) | 完成 |
| 2 | App of Apps | [docs/phase-2.md](./docs/phase-2.md) | 完成 |
| 3 | AppProject 權限邊界 | [docs/phase-3.md](./docs/phase-3.md) | **進行中** |
| 4 | ApplicationSet | 待補 | |
| 5 | sync-wave / hooks | 待補 | |
| 6 | Image Updater / Notifications（選修） | 待補 | |

## 現在請做

👉 **[Phase 3：AppProject](./docs/phase-3.md)**

```bash
kubectl config use-context k3d-argocd-lab
kubectl apply -f projects/demo.yaml
# 再把 applications/demo-web-*.yaml 的 project 改成 demo
```

> 注意：本機若有多個 kube context，務必先確認是 `k3d-argocd-lab`。
