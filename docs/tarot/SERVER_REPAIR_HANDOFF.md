# 星语塔罗 · 另一台设备的服务端修补交接

更新：2026-10-25。已在原开发设备找到完整邮件后端并补齐塔罗服务端实现。本文记录源码恢复、接入进度与后续验收，不代表线上 AI 已启用。

## 2026-10-25 接入进度

- 独立 API 仓库为 `boringHub/external-api`，本机 `main` 的 `5ef9e44`、`d18556a`、`acccdaf` 三个提交此前没有推送，包含邮箱验证、分享、回执、反馈、邮件队列及 `0001`–`0007` 迁移。另一台设备拿到的 `ae580be` 确实是旧基线。
- 2026-10-25 已将上述三个提交及塔罗接入提交 [`fa6b6a4`](https://github.com/boringHub/external-api/commit/fa6b6a4) 推送至 API 仓库 `main`。另一台设备应先拉取该版本；源码已同步不代表线上 Worker 已更新。
- Cloudflare 只读核对：`notify-api` 当前版本 `b661043c-3573-478c-b736-160176d62087`，2026-10-18 由 Wrangler 上传；远程 `love` 已应用 `0001`–`0007`。部署元数据与迁移核对不能单独证明源码逐字节一致。
- `services/notify-api/src/tarot/` 已实现三条签名接口、78 张受控中文牌义、六牌阵校验、OpenAI 兼容 Chat Completions 流适配、原子幂等/额度、加密恢复缓存、生成中 purge 和租约/定时清理。独立迁移为 `migrations-tarot/0001_create_tarot.sql`，不应用到原邮件库。
- 后端 `npm run check`、91 项本地测试和 `wrangler deploy --dry-run` 通过，包括原有 61 项邮件/身份/分享/反馈测试；新增测试使用隔离 D1 和模拟供应商，不证明真实中转站及 Android 已可用。
- App `npm test` 的 138 项测试及 `npm run build` 通过；本次 App 仅更新开发文档与未发布日志，没有构建或发布 APK。
- `npm run deploy` 已改为要求已提交且同步 GitHub 的干净 `main`，把源码 SHA 写入 Worker 版本说明；远程迁移改为显式执行，避免再次只部署本机源码。
- 当前没有创建生产 TAROT D1、执行远程迁移、替换线上 Worker 或发布 APK。中转站协议/地址、模型、价格、预算、开通设备及隐私政策仍待配置；App 与服务端保持默认关闭。恢复/清理在后端关闭生成后仍可用，但实际需要 TAROT 绑定。

### 同日真实中转联调补充

用户已在 API 本机忽略文件中配置中转站、模型和 Key。真实测试发现所选模型默认先输出推理，短额度测试没有公开答复；显式发送 `thinking: {type: "disabled"}` 后流式文本、`[DONE]` 和 usage 正常。API 增加可选服务端变量 `TAROT_THINKING_MODE=disabled`，其他供应商默认不发送该字段，配置模板不包含真实地址或密钥。

使用真实受控单牌提示词与虚构桌面整理问题实测：首字约 2.4 秒，13.7 秒完成，798 字，输入 418 / 输出 491 tokens。API 回归现为 92 项测试，类型检查与打包预检通过。这只证明供应商边界，未验证线上签名链路、生产 TAROT D1 和 Android；价格、预算、设备开通、数据政策及 tokenizer 上界仍待核对。上文“中转站与模型待配置”的状态已由本段更新，AI 开关继续关闭。

准确配置名和上线步骤以 API 仓库 `services/notify-api/TAROT.md` 为准。`TAROT_DAILY_BUDGET_MICROS` 以及输入/输出每百万 token 单价统一使用货币百万分之一，配置实际币种和价格版本；提示词版本固定于源码 `tarot-v1.0`。当前适配器只支持经验证的非推理文本模型及 OpenAI 兼容流协议，其他中转协议须先补适配。

下文保留 2026-10-23 的问题分析与验收要求，其中“源码缺失 / 服务端未实施”描述的是当时另一台设备的状态。

## 2026-10-23 历史结论与影响

App 已具备本地抽牌、牌义、历史，以及尚未启用的 AI 客户端接口。调用链固定为 **App → 自有 API → 模型中转站**，模型标识、上游地址、密钥和预算只由服务器管理。暂不确定具体模型，不需要在 App 中填写 GPT、DeepSeek 等型号。

本机的 API 是 App 仓库旁边的独立仓库 `../external-api`，不是 `diary-app/external-api`。2026-10-23 核对时，其 HEAD 为 `ae580be`（`fix: allow Capacitor API requests`），此前核对的 GitHub 主分支也是这一早期基线。`services/notify-api/src/index.ts` 只见早期注册、设备状态、撤销及事件路由；迁移目录只有 `0001`–`0003`，配置只有早期 `EVENTS` D1。缺少 App 当前使用的完整邮箱验证、日记分享、反馈、回执和邮件队列实现。

这说明目前拿到的源码不足以安全扩展线上 Worker，**不能由此断定之前采用了哪种部署方式**。从另一台设备的工作目录手动部署、存在未提交改动、使用另一分支或另一份仓库，都可能造成差异。源码没有提交本身不会让已经部署的服务立即停止，但会影响后续复现、修复和安全发布。

2026-10-23 当次没有改动或部署 `external-api`，没有运行远程 D1 迁移，也没有替换线上邮件服务。AI 关闭期间仍可完成全部本地仪式与记录；在线解读、服务器恢复缓存和服务器清理功能需要后续接入验收。

## 一、先找到实际部署源码

在原来开发/部署邮件服务的设备上操作：

1. 找到当时执行部署的项目目录，检查终端工作目录、编辑器项目和 Git 分支。先保留全部未提交文件及本地配置，核对 `.gitignore` 后备份；不要先用当前旧仓库覆盖它。
2. 查看该目录的 `package.json` 部署脚本、`wrangler.jsonc` / `wrangler.toml`、源码入口、迁移目录及环境覆盖配置。记录实际 Worker 名称、环境、域名、D1、Queue 生产者/消费者、定时任务和其他绑定。数据库 ID 属于配置元数据；密钥值不能进入仓库、文档、截图或日志。
3. 到 Cloudflare 控制台查看该 Worker 的当前部署、版本时间及构建记录，区分 Git 构建和本地上传。把源码目录、Git 提交或未提交改动与实际版本对应起来。仅看 App 仓库是否提交，不能判断独立 API 仓库是否部署。
4. 先运行下列本地只读命令；Wrangler 命令从实际服务目录运行，沿用项目锁定版本及真实环境参数。

   ```powershell
   git status --short
   git log -8 --oneline
   git remote -v
   npx --no-install wrangler deployments list
   npx --no-install wrangler versions list
   npx --no-install wrangler secret list
   ```

   `secret list` 仅用于核对名称。不要导出 Secret 的值。控制台若没有关联 Git 构建，仍须继续寻找原始代码，不能把部署产物当成完整 TypeScript 源码与迁移历史。
5. 将完整源码、迁移、锁文件、脱敏配置模板及必要测试提交到 API 仓库；密钥和本机 `.dev.vars` 留在安全配置中。记录“源码提交 ↔ 线上版本”的对应关系。缺失迁移或配置时继续追查，不猜测编号后直接应用生产数据库。

特别注意：本机旧 `services/notify-api/package.json` 的 `deploy` 会先执行远程 D1 migration，再部署 Worker。核对完成前不要运行该脚本，也不要直接从本机旧副本部署。

## 二、先证明原邮件功能能复现

在隔离的测试 Worker、测试 D1 和测试队列中还原完整服务。根据实际源码核对 App 的现有请求，不凭文档推断线上路由。至少验证：

- 设备签名注册、重复注册、错误签名和撤销行为；旧 App 仍可使用。
- 邮箱验证码、确认、更换、关闭和解绑；使用者收件邮箱与固定日记分享邮箱相互独立。
- 日记分享、反馈建议、消费者投递、去重、失败退避、回执与已读状态。
- 当前邮件接口的请求体限制、Origin allowlist、Secret、Queue binding 和消费者全部保持兼容。

完成源码对齐和这组回归后，再往同一个 API 的独立模块中增加塔罗。不得用早期 `index.ts` 替换完整服务。

## 三、塔罗接口与实施约束

详细约束以 [API_AND_DATA.md](API_AND_DATA.md) 为准；App 实现见 [IMPLEMENTATION.md](IMPLEMENTATION.md)。2026-10-25 服务端已补齐本地实现，线上状态见本文开头。

| 方法 | App 实际路径 | 行为 |
| --- | --- | --- |
| POST | `/v1/tarot/chat/stream` | 首解 / 追问，返回自有 SSE |
| GET | `/v1/tarot/requests/:requestId` | 用户主动恢复单个请求，不触发模型 |
| POST | `/v1/tarot/readings/:readingId/purge` | 无 body，幂等清理该设备该记录的缓存 |

接入文件：`src/features/tarot/services/tarotAiClient.ts`、`domain/contextBuilder.ts`、`domain/types.ts`、`data/spreads.ts` 和共享的 `src/services/signedTransport.ts`。

实际牌组使用 **`deckId=rws-1909`、`deckVersion=1.0.0`、`catalogVersion=rws-zh-v1`**。素材及完整清单在 `public/tarot/decks/rws-1909/1.0.0/manifest.json`，牌阵稳定 ID 在 `data/spreads.ts`。服务端应复制/生成受版本管理的白名单，不能接收客户端传来的任意牌义或提示词。

实施约束：

1. 复用原 P-256 设备签名、中间件、设备撤销和 Origin allowlist。请求签名按实际 method/path/body，GET/purge 的 body 为空字符串。不能依赖通知邮箱、旧共享 token 或客户端传入的模型密钥授权 AI。
2. 在读取 body **之前**按路由选择上限：只有塔罗生成接口允许 96 KiB，不能放宽原邮件接口限制。严格校验 schema、UUID、Unicode 长度、六种场景与牌阵对应关系、唯一卡牌和稳定牌位顺序。
3. 新增独立 `TAROT` D1 及适用迁移，保留现有身份库；AI 不进邮件 Queue，不给任何邮箱发送塔罗内容。D1 迁移编号根据恢复后的真实仓库确定。
4. 服务端组装系统规则、版本化中文牌义和请求白名单。只接收已抽定的牌，不让模型抽牌或更换朝向。客户端不上传完整牌序、私人笔记、显示标题、其他日记、图片或通知邮箱。
5. 接通中转站时先核对其真实协议，再写供应商适配层。上游地址由服务端 allowlist 选择；模型名、超时、输出上限和费用配置均可切换。只把公开回答文本映射为 `meta/delta/done/error`，不透传密钥、原始错误或内部推理。
6. 原子实现请求占位、设备并发锁和设备/IP/全局预算预留。按设备 + requestId 幂等，同 ID 不同请求体拒绝；不确定结果不得静默重调模型。上游调用与数据库无法形成一个事务，必须保留 `unknown` 及用户明确的新尝试行为。
7. 先加密保存完整回复和完成状态，再发 `done`。恢复缓存只允许所属设备读取，24 小时内有效；请求元数据与 purge tombstone 按契约保留，不记录问题和聊天正文日志。
8. purge 先写 tombstone，阻止仍在生成的任务晚到后重建缓存。定时清理过期密文；隐私说明需包含实际备份/Time Travel 和中转站留存政策。

实际服务端配置名称（全部先留空或关闭，不表示已可用；完整模板见 API 的 `.dev.vars.example`）：

```dotenv
TAROT_AI_ENABLED=false
TAROT_PROVIDER=
TAROT_MODEL=
TAROT_DEVICE_ALLOWLIST=[]
TAROT_PROVIDER_URL=
TAROT_PROVIDER_ALLOWED_ORIGINS=[]
TAROT_DAILY_BUDGET_MICROS=
TAROT_INPUT_PRICE_MICROS_PER_MILLION=
TAROT_OUTPUT_PRICE_MICROS_PER_MILLION=
TAROT_PRICE_VERSION=
TAROT_BUDGET_CURRENCY=
# 上游基地址仅在服务端配置，并受 allowlist 限制。
# 以下名称通过 Worker Secret 管理，不能在此填写真实值：
# TAROT_PROVIDER_API_KEY
# TAROT_CACHE_ENCRYPTION_KEYS
# TAROT_HMAC_KEY
```

未配置模型、Secret、可承受预算或设备开通策略时必须 fail closed。提示词版本固定在受版本管理的源码中；模型 Key、缓存密钥和摘要密钥各自独立。

## 四、App 何时开启 AI

目前 `.env.example` 为：

```dotenv
VITE_NOTIFY_API_ORIGIN=
VITE_TAROT_AI_ENABLED=false
VITE_TAROT_AI_PROVIDER=
VITE_TAROT_AI_POLICY_URL=
```

已有邮件 API 地址继续沿用，不要清除原设备的有效配置。只为塔罗留空新增项。App 不需要填写模型名、上游地址或模型 Key。

服务端先以 AI 关闭状态部署，通过隔离环境和设备白名单测试后，再在 App 构建环境打开塔罗开关，并填写真实服务商名称及 HTTPS 数据政策地址。涉及中转站时应说明真正接收数据的中转站和模型服务，不用一个模型品牌代替完整披露。同意版本在 `tarotConsent.ts`；更换数据接收方或政策时需要重新确认，重大处理变化也应更新 `CONSENT_VERSION`。

浏览器预览保持离线模式；当前真实设备签名路径仅允许 Android。打开历史不会自动生成 AI 解读；用户通过“同意并开始解读 / 让 AI 解读”主动触发。恢复按需点击，不轮询，不因回到前台自动重生成。

## 五、启用前的验收和回退

- 使用 Mock provider 验证同 ID 10 并发只调用一次上游；不同 ID 同设备并发限制、临界预算并发、失败预留释放与未知结果结算。
- 验证错误签名、撤销设备、跨设备读取、同 ID 换 body、超龄请求、缓存过期/已清理均不会重调模型。
- 验证中文逐字节、CRLF、多行 data、心跳、缺少 done、length 结束、断流、超时、取消及恢复替换部分文字。
- 验证生成中删除的 tombstone、离线清理、过期清理与密钥轮换；本机删除后晚回包不能复活记录。
- Android 真机验证 fetch + ReadableStream 的首字延迟、持续增量、停止、切后台、网络断开及键盘/系统返回。浏览器和 sql.js 测试不能替代该项。
- 完成 30+ 人工内容案例；复核牌名/方向/牌位、购物预算、A/B 对称分析、无必然预测或额外卡牌。对当前原创中文牌义做人工审阅。
- 再跑完整邮件回归。验证上一版 Android 覆盖升级保留真实日记及塔罗记录；发布另按 `RELEASE.md`，不先卸载。

有问题先关闭服务端 `TAROT_AI_ENABLED`，保留新增表、离线记录和原邮件接口。不要通过删除数据库回退。当前会话没有完成以上服务端或 Android 真机验收，也没有发布 APK。

## 给另一台设备的开发助手

可直接复制以下任务：

> 先拉取 boringHub/shiguangjian 和 boringHub/external-api，再读两边 AGENTS.md、本文及 API 的 services/notify-api/TAROT.md。2026-10-25 已找回完整邮件源码并补齐塔罗签名 SSE、恢复、purge、独立迁移、原子幂等/额度及加密缓存，后端 91 项测试和类型检查通过；生产仍是旧 Worker，TAROT D1 和真实中转配置尚未创建。下一步核对实际网关协议、非推理模型、计费、预算、设备白名单与数据政策，在隔离环境验证供应商及邮件流程，再配置真实 TAROT D1/Secret。使用要求 main 已提交并同步 GitHub 的部署脚本，记录源码 SHA 与 Worker 版本。App deckId=rws-1909，默认关闭 AI；模型和密钥只在服务器配置。完成 Android 真机流式及内容验收后再启用；不重复实现现有模块，不把旧邮件源码覆盖生产。
