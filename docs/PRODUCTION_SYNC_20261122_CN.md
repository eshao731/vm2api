# 2026-11-22 生产同步与部署核对

## 代码与分支

- 通过 `fetch --all --prune` 拉取社区仓库，社区 `main` 收尾提交为 `13e36c721e461531625a3fa123f764731be7ff45`，版本 `1.4.173`。
- 生产分支为个人 fork 的 `codex/production`，合并提交 `dd638741846fbb1aa5dc636c593be3de685183a1` 已推送到 `fork/codex/production`，没有向社区仓库推送。
- 社区更新相对上次生产基线包含 1,280 个提交；合并无冲突。原有官方 Codex Responses 透传定制的 7 个源码、测试和前端文件逐项比较后全部保留，配置开关仍默认关闭。
- 本地只有 `codex/production` 一个分支和一个 worktree；没有可安全删除的冗余本地分支或副本。两份原有未跟踪部署文档保留且 SHA-256 未变化：`docs/DEPLOY_CODEX_CPA_A_CN.md`、`docs/DEPLOY_CODEX_CPA_A_STATUS_CN.md`。

## 验证

- Codex 定向回归：84/84 通过。
- 前端：177 个测试文件、1,071 项测试通过；格式检查和生产构建通过。
- `worker`、`api-kernel`、`redact` 的 Go vet/test 通过；部署 Python 测试 78 项通过；Node 语法和 Biome 格式检查通过。
- Linux 隔离全量回归：416 个文件，6,068 项测试，6,027 通过、3 失败、38 跳过、0 取消。三个失败在纯社区源码基线中同样复现：`panel-vm-seed-settings.test.mjs`、`routing-runtime-load.test.mjs`、`telemetry-status.test.mjs`；没有为掩盖它们修改产品代码。另将受测试进程内存上限影响的 `redaction-runtime.test.mjs` 单独重跑，4/4 通过。
- 社区基础镜像的 479 个运行源码文件、生产镜像加入定制后的 824 个源码/前端文件均通过摘要校验；固定 payload 的 23 个运行文件通过校验。

## 线上部署

- 目标为 CPA-A 专用 KVM 客体 `/opt/vm2api`；切换前暂停 3 个可调度槽并确认在途请求为 0，控制面停止后创建加密一致性备份，切换完成再恢复原调度状态。槽位容器未重建。
- 运行镜像：`vm2api:production-dd638741`。
- 镜像 ID：`sha256:d5468752d1d0585d0c53ed651aa34b2c7de53e9278ea44d58add4ae15aeaad2d`。
- 官方基础镜像 digest：`sha256:6b6a764850864beef853b0577657186ccba797d7919739435f96c8feaafd9c88`。
- 版本接口返回 `current=latest=1.4.173`、`update_available=false`；健康接口 200；未授权管理接口 401；容器重启数 0、OOM=false。
- 宿主 `cpa`、`new-api`、`codex2api`、`nginx`、`vm2api-vm`、`vm2api-vm-firewall` 均 active；公网 `https://my-vm2api.uu6.top/cc/` 返回 200；宿主 38822/38887 仍只绑定 IPv4 回环。
- 新版启动时按官方默认补写了 6 个缺失的 pool 调度字段，原有配置值、非镜像环境变量、槽位状态和槽位容器均保持不变。Codex `official_passthrough` 仍为 false。
- 本次备份：客体 `/root/vm2api-backups/20261121T162819Z-pre-production-dd638741/state.tar.gz.enc`，恢复密钥同目录 `restore.key`；切换脚本已验证密文可读取并具备回滚路径。

## 真实请求

- 切换后 zc1、zc3 的 `gpt-6-luna` Responses 请求均返回 HTTP 200、文本 `OK`。
- zc2 返回上游 HTTP 401 `token_revoked`（invalidated oauth token）；这是该槽现有凭证状态，未在部署中修改凭证或调度设置。三个槽位仍保持 running、schedulable=true。

## 清理

- 删除了本次临时验证镜像 `vm2api:verify-20261122` 和旧的中间生产镜像 `vm2api:production-8a88845a`；保留当前镜像、上一个生产镜像、旧版回滚镜像及加密备份。
- 没有删除槽位容器、账号数据、凭证、工作区或其他服务目录。部署源和验证日志保留在客体 `/opt/vm2api-release-20261122/` 与本地 `.local/production-sync-20261122/`。
