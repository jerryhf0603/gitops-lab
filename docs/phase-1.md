# Phase 1：Application 行為精熟

目標：理解 **手動 sync / diff / history / rollback / 排錯**，以及 prod 與 dev/uat 的差異。

---

## 你已完成

- [x] prod 移除 `automated` → 手動 sync
- [x] 改 Git 後 prod 變 OutOfSync，不會自動套用
- [x] 手動 scale prod → OutOfSync，不會 selfHeal 拉回
- [x] `argocd app diff / sync / history / rollback`
- [x] 故意用錯 image tag → Degraded → 排錯 → `git revert` 修復

---

## 核心觀念

| 概念 | 說明 |
|------|------|
| Manual sync | Git 變了只標 OutOfSync，要你按 Sync |
| selfHeal | 只有 `automated.selfHeal: true` 才會自動拉回 drift |
| rollback | 把**叢集**回到舊 revision，**不會改 Git** → 常變 OutOfSync |
| Synced vs Healthy | YAML 套用成功 ≠ Pod 一定正常 |
| patch | 跳過 Git 直接改叢集，GitOps 下僅救火用 |

---

## 下一步

👉 [Phase 2：App of Apps](./phase-2.md)
