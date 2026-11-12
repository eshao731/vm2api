# 2026-11-13 生产同步与部署核对

## 代码与分支

- 社区主线：`c865ab6cab37498ad5d6039a3032a79a8eb78aae`，应用版本 `1.4.077`；已通过 `fetch --all --prune` 拉取，并在发布前用 `ls-remote` 复核。
- 生产分支：个人 fork 的 `codex/production`；合并提交 `8a88845ab021807e84c8cdd40e4b8c984515966e` 已推送，没有向社区仓库推送。相对旧社区基线纳入 257 个提交，合并无冲突，旧生产历史保留。
- 逐项比较生产定制涉及的 7 个源码、测试和前端文件的增删行，原定制全部保留：官方 Codex Responses 内容/采样参数透传、凭据剥离、会话和缓存键重建、配置校验及面板开关。开关仍默认关闭。
- 本地原来只有生产分支和 `main`，两者均指向旧生产提交。确认旧 `main` 是新生产分支的祖先后删除这一冗余分支。当前只保留本地生产分支，仓库仅一个 worktree，未发现多余工作副本。
- 原有两份未跟踪部署文档 `DEPLOY_CODEX_CPA_A_CN.md`、`DEPLOY_CODEX_CPA_A_STATUS_CN.md` 保留，SHA-256 核对未改。其他远程功能分支未批量删除。

## 验证

- Codex 定向回归 78/78 通过。
- 前端 79 个测试文件、626 项测试通过；格式检查和构建通过。
- `worker`、`api-kernel`、`redact` 的 `go vet ./...`、`go test ./...` 均通过。Node 语法、Biome（727 个文件）和 Go 格式检查通过。
- Linux / Node 22.23.5 全量隔离验证：359 个文件、4,720 项测试，4,712 通过、3 失败、5 跳过、0 取消。未将结果标记为全绿。
- 三项失败在同一环境下恢复为纯社区源码后复现，基线复核 30 项中 27 通过、3 失败：`panel-vm-seed-settings.test.mjs` 原生配置发布、`routing-runtime-load.test.mjs` 远程 VM 发布统计、`telemetry-status.test.mjs` 停止槽遥测状态。此次未修改这些模块以掩盖失败。
- Linux 部署/Python 测试 40/40 通过；macOS 的 `/var` 符号链接和 BSD `sed` 差异通过 Linux 实测排除，没有据此修改产品代码。
- `v1.4.077` 固定 payload 的 22 个文件摘要全部匹配；生产镜像中 700 个源码、脚本、Docker 支持文件、前端文件和清单与合并结果/本地构建逐字节一致。

## 线上部署

2026-11-13 01:43:34（北京时间）完成升级。运行代码为 `8a88845a`，后续文档提交不改变运行代码。目标为 CPA-A 专用 KVM 客体 `/opt/vm2api`，管理域名 `my-vm2api.uu6.top`。

- 运行镜像：`vm2api:production-8a88845a`。
- 镜像 ID：`sha256:81416134f8418d478999088461b24341a9312df5f0987574afa80724d7293d80`。
- 官方基础镜像固定 digest：`sha256:0c8897dd62b7c1de544ec07e9aaca7ba7fd6ab801a83c54ae80f0fdd8e046775`，仅覆盖三个定制协议模块和本地构建的前端。
- Codex kernel SHA-256：`ec9abb0160eaa0c8fda474c52fcfab592a4a2b905da4e4525e14fee9fb61376d`，与新清单一致。
- 切换前确认无在途请求，停止控制面取得一致备份，再重建控制面；槽位容器保持运行，没有删除或重建槽位。
- 加密状态备份：客体 `/root/vm2api-backups/20261112T174305Z-pre-production-8a88845a/state.tar.gz.enc`，恢复密钥在同目录 `restore.key`。已验证密文可解密读取归档；旧生产镜像保留供回滚。
- `routing.json` 摘要不变，`.env` 除镜像选择项外不变，槽位运行及人工调度设置保持原样。
- 管理 HTTPS 200，`/health` 为 200 / `ok`，模型目录 200（29 个模型），未授权管理请求 401。版本接口返回 `current=latest=1.4.077`、`update_available=false`；容器重启数 0、无 OOM。
- CPA、New API、Codex2API、Nginx、专用 VM 和隔离防火墙均 active；宿主 38822 / 38887 仍只绑定 IPv4 回环。

## 真实推理限制

升级前唯一 Codex 槽位 `QWEzzz` 已为 `schedulable=false`。本次保留该设置，没有重新启用账号或改变调度策略。升级前、后真实 `gpt-6-luna` Responses 请求都返回 HTTP 503、`no_codex_vm`，因此本次不能宣称真实上游推理验收通过。后续需在具备可调度账号后另行验收。

## 清理与复现

本次隔离验证/社区基线镜像及临时源码、前端传输包已清理；保留新旧生产镜像、加密备份、生产构建源和验证日志。没有批量清理槽位镜像、账号数据或其他服务。

本地记录位于忽略目录 `.local/production-sync-20261113/`，包括完整 Linux 日志、社区失败复核、前端/Go 检查结果、定制保留比较、部署脚本和线上清单。客体构建源为 `/opt/vm2api-release-20261113/`；部署日志 `/tmp/vm2api-deploy-production-20261113.log`，全量测试日志 `/tmp/vm2api-linux-unit-20261113.log`。
