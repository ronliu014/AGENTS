# 代码库规范
- 代码库地址：https://github.com/moltbot/moltbot
- GitHub 问题/评论/PR 评论：使用字面量多行字符串或 `-F - <<'EOF'`（或 $'...'）实现真实换行；切勿嵌入 "\\n"。

## 项目结构与模块组织
- 源代码：`src/`（CLI 关联逻辑在 `src/cli`，命令在 `src/commands`，Web 提供者在 `src/provider-web.ts`，基础设施在 `src/infra`，媒体流水线在 `src/media`）。
- 测试文件：与源码同目录的 `*.test.ts`。
- 文档：`docs/`（包含图片、队列、树莓派配置）。构建产物存放在 `dist/`。
- 插件/扩展：存放于 `extensions/*`（工作区包）。仅插件依赖需放在插件目录的 `package.json` 中；除非核心代码使用，否则不要添加到根目录 `package.json`。
- 插件安装：会在插件目录执行 `npm install --omit=dev`；运行时依赖必须放在 `dependencies` 中。避免在 `dependencies` 中使用 `workspace:*`（会导致 npm install 失败）；请将 `moltbot` 放在 `devDependencies` 或 `peerDependencies` 中（运行时通过 jiti 别名解析 `clawdbot/plugin-sdk`）。
- 安装程序（从 `https://molt.bot/*` 提供）：存放于同级代码库 `../molt.bot`（包含 `public/install.sh`、`public/install-cli.sh`、`public/install.ps1`）。
- 消息渠道：重构共享逻辑（路由、白名单、配对、命令网关、入职引导、文档）时，需考虑**所有**内置渠道 + 扩展渠道。
  - 核心渠道文档：`docs/channels/`
  - 核心渠道代码：`src/telegram`、`src/discord`、`src/slack`、`src/signal`、`src/imessage`、`src/web`（WhatsApp web）、`src/channels`、`src/routing`
  - 扩展（渠道插件）：`extensions/*`（例如 `extensions/msteams`、`extensions/matrix`、`extensions/zalo`、`extensions/zalouser`、`extensions/voice-call`）
- 添加渠道/扩展/应用/文档时，需检查 `.github/labeler.yml` 确保标签覆盖完整。

## 文档链接规范（Mintlify）
- 文档托管于 Mintlify（docs.molt.bot）。
- `docs/**/*.md` 中的内部文档链接：使用根相对路径，不带 `.md`/`.mdx` 后缀（示例：`[配置](/configuration)`）。
- 章节交叉引用：在根相对路径上使用锚点（示例：`[钩子](/configuration#hooks)`）。
- 文档标题与锚点：标题中避免使用破折号和撇号，否则会破坏 Mintlify 锚点链接。
- 当 Peter 索要链接时，回复完整的 `https://docs.molt.bot/...` URL（而非根相对路径）。
- README（GitHub）：保留绝对文档 URL（`https://docs.molt.bot/...`），确保链接在 GitHub 上可正常访问。
- 文档内容需通用：不出现个人设备名/主机名/路径；使用占位符如 `user@gateway-host` 和“网关主机”。

## exe.dev 虚拟机操作（通用）
- 访问方式：稳定路径为 `ssh exe.dev` 后执行 `ssh vm-name`（假设 SSH 密钥已配置）。
- SSH 不稳定：使用 exe.dev 网页终端或 Shelley（网页代理）；长时间操作需保留 tmux 会话。
- 更新：`sudo npm i -g moltbot@latest`（全局安装需要 `/usr/lib/node_modules` 的 root 权限）。
- 配置：使用 `moltbot config set ...`；确保设置 `gateway.mode=local`。
- Discord：仅存储原始令牌（不带 `DISCORD_BOT_TOKEN=` 前缀）。
- 重启：停止旧网关并执行以下命令：
  `pkill -9 -f moltbot-gateway || true; nohup moltbot gateway run --bind loopback --port 18789 --force > /tmp/moltbot-gateway.log 2>&1 &`
- 验证：`moltbot channels status --probe`、`ss -ltnp | rg 18789`、`tail -n 120 /tmp/moltbot-gateway.log`。

## 构建、测试与开发命令
- 运行时基准：Node **22+**（确保 Node + Bun 路径均可正常工作）。
- 安装依赖：`pnpm install`
- 提交前钩子：`prek install`（执行与 CI 相同的检查）
- 也支持：`bun install`（修改依赖/补丁时，需保持 `pnpm-lock.yaml` 与 Bun 补丁同步）。
- TypeScript 执行优先使用 Bun（脚本、开发、测试）：`bun <file.ts>` / `bunx <工具>`。
- 开发环境运行 CLI：`pnpm moltbot ...`（bun）或 `pnpm dev`。
- Node 仍支持运行构建产物（`dist/*`）和生产环境安装。
- Mac 打包（开发）：`scripts/package-mac-app.sh` 默认使用当前架构。发布检查清单：`docs/platforms/mac/release.md`。
- 类型检查/构建：`pnpm build`（tsc）
- 代码检查/格式化：`pnpm lint`（oxlint）、`pnpm format`（oxfmt）
- 测试：`pnpm test`（vitest）；覆盖率：`pnpm test:coverage`

## 编码风格与命名规范
- 开发语言：TypeScript（ESM）。优先使用严格类型；避免 `any`。
- 格式化/代码检查通过 Oxlint 和 Oxfmt 实现；提交前需运行 `pnpm lint`。
- 复杂或非直观逻辑需添加简短代码注释。
- 保持文件简洁；提取辅助函数，而非创建“V2 版本”副本。CLI 选项和依赖注入遵循现有模式（通过 `createDefaultDeps`）。
- 目标：文件行数控制在 ~700 行以内（仅为指导原则，非硬性限制）。拆分/重构可提升清晰度或可测试性时执行。
- 命名：产品/应用/文档标题使用 **Moltbot**；CLI 命令、包/二进制文件、路径、配置键使用 `moltbot`。

## 发布渠道（命名）
- stable（稳定版）：仅打标签发布（例如 `vYYYY.M.D`），npm 分发标签为 `latest`。
- beta（测试版）：预发布标签 `vYYYY.M.D-beta.N`，npm 分发标签为 `beta`（可能不包含 macOS 应用）。
- dev（开发版）：`main` 分支最新提交（无标签；需 git checkout main）。

## 测试规范
- 框架：Vitest + V8 覆盖率阈值（行/分支/函数/语句覆盖率 70%）。
- 命名：测试文件与源码同名，后缀为 `*.test.ts`；端到端测试为 `*.e2e.test.ts`。
- 修改业务逻辑后，推送前需运行 `pnpm test`（或 `pnpm test:coverage`）。
- 测试工作线程数不要超过 16；已验证此上限最优。
- 真实环境测试（使用真实密钥）：`CLAWDBOT_LIVE_TEST=1 pnpm test:live`（仅 Moltbot）或 `LIVE=1 pnpm test:live`（包含提供者真实测试）。Docker 测试：`pnpm test:docker:live-models`、`pnpm test:docker:live-gateway`。入职引导 Docker 端到端测试：`pnpm test:docker:onboard`。
- 完整测试套件及覆盖范围：`docs/testing.md`。
- 纯测试代码的新增/修复通常**无需**更新变更日志，除非修改了用户可见行为或用户明确要求。
- 移动端测试：使用模拟器前，先检查是否有已连接的真实设备（iOS + Android），优先使用真实设备。

## 提交与拉取请求（PR）规范
- 使用 `scripts/committer "<msg>" <file...>` 创建提交；避免手动执行 `git add`/`git commit`，确保暂存区范围可控。
- 提交信息简洁、面向动作（例如：`CLI: 为 send 命令添加详细日志标志`）。
- 相关修改分组提交；避免将无关重构打包在一起。
- 变更日志流程：最新发布版本保持在顶部（无 `Unreleased` 章节）；发布后，升级版本号并新增顶部章节。
- PR 需总结修改范围、说明已执行的测试，并提及所有用户可见的变更或新标志。
- PR 评审流程：收到 PR 链接时，通过 `gh pr view`/`gh pr diff` 评审；**不要**切换分支。
- PR 评审会议：优先使用单个 `gh pr view --json ...` 批量获取元数据/评论；仅必要时运行 `gh pr diff`。
- 粘贴 GitHub 问题/PR 链接并开始评审前：先执行 `git pull`；若存在本地修改或未推送的提交，需停止操作并提醒用户。
- 目标：合并 PR。提交记录清晰时优先**变基（rebase）**；提交历史混乱时使用**合并压缩（squash）**。
- PR 合并流程：从 `main` 创建临时集成分支，将 PR 分支合并到该分支（提交记录清晰优先变基；复杂度/冲突较高时允许普通合并），修复问题，添加变更日志（包含 PR 编号 + 致谢），最终提交前**本地执行完整检查**（`pnpm lint && pnpm build && pnpm test`），提交后合并回 `main`，删除临时分支，切回 `main`。重要：确保贡献者信息出现在 git 提交记录中！
- 若评审 PR 后参与了后续开发，需通过合并/压缩方式合入（禁止直接提交到 main），并始终将 PR 作者列为共同贡献者。
- 处理 PR 时：在变更日志中添加包含 PR 编号的条目，并致谢贡献者。
- 处理问题时：在变更日志条目中引用问题编号。
- 合并 PR 时：在 PR 评论中说明具体操作，并附上 SHA 哈希值。
- 合并新贡献者的 PR 时：将其头像添加到 README 的“感谢所有贡献者（clawtributors）”缩略图列表。
- 合并 PR 后：若贡献者未出现在列表中，执行 `bun scripts/update-clawtributors.ts`，然后提交更新后的 README。

## 快捷命令
- `sync`：若工作区有未提交修改，提交所有变更（选择合理的规范提交信息），然后执行 `git pull --rebase`；若变基冲突且无法解决则停止；否则执行 `git push`。

### PR 工作流（评审 vs 合入）
- **评审模式（仅提供 PR 链接）**：读取 `gh pr view/diff`；**不要**切换分支；**不要**修改代码。
- **合入模式**：从 `main` 创建集成分支，合入 PR 提交（线性历史优先**变基**；复杂度/冲突较高时允许**普通合并**），修复问题，添加变更日志（+ 致谢 + PR 编号），提交前**本地执行完整检查**（`pnpm lint && pnpm build && pnpm test`），提交后合并回 `main`，然后执行 `git switch main`（合入后切勿停留在主题分支）。重要：确保贡献者信息出现在 git 提交记录中！

## 安全与配置提示
- Web 提供者的凭据存储在 `~/.clawdbot/credentials/`；若登出需重新执行 `moltbot login`。
- 树莓派会话默认存储在 `~/.clawdbot/sessions/`；基础目录不可配置。
- 环境变量：参考 `~/.profile`。
- 切勿提交或发布真实电话号码、视频或生产环境配置值。文档、测试和示例中使用明显的假占位符。
- 发布流程：执行任何发布工作前，务必阅读 `docs/reference/RELEASING.md` 和 `docs/platforms/mac/release.md`；文档已覆盖的常规问题，无需重复询问。

## 故障排除
- 品牌重塑/迁移问题或遗留配置/服务警告：执行 `moltbot doctor`（参考 `docs/gateway/doctor.md`）。

## 代理专属说明
- 术语："makeup" = "mac 应用"。
- 切勿编辑 `node_modules`（包括全局/Homebrew/npm/git 安装的版本）。更新会覆盖修改。技能说明需放在 `tools.md` 或 `AGENTS.md`。
- Signal 相关："update fly" 对应命令 → `fly ssh console -a flawd-bot -C "bash -lc 'cd /data/clawd/moltbot && git pull --rebase origin main'"`，然后执行 `fly machines restart e825232f34d058 -a flawd-bot`。
- 处理 GitHub 问题/PR 时，任务结束需打印完整 URL。
- 回答问题时，仅提供高可信度答案：需在代码中验证；切勿猜测。
- 切勿更新 Carbon 依赖。
- 所有包含 `pnpm.patchedDependencies` 的依赖必须使用精确版本（禁止 `^`/`~`）。
- 依赖补丁（pnpm 补丁、覆盖、或本地修改）需明确审批；默认禁止操作。
- CLI 进度展示：使用 `src/cli/progress.ts`（`osc-progress` + `@clack/prompts` 加载动画）；禁止手动实现加载动画/进度条。
- 状态输出：保留表格 + ANSI 安全换行（`src/terminal/table.ts`）；`status --all` = 只读/可粘贴，`status --deep` = 探测模式。
- 网关当前仅作为菜单栏应用运行；未安装独立的 LaunchAgent/辅助标签。通过 Moltbot Mac 应用或 `scripts/restart-mac.sh` 重启；验证/终止需使用 `launchctl print gui/$UID | grep moltbot`，而非假设固定标签。**macOS 调试时，通过应用启动/停止网关，禁止使用临时 tmux 会话；交接前终止所有临时隧道。**
- macOS 日志：使用 `./scripts/clawlog.sh` 查询 Moltbot 子系统的统一日志；支持跟踪/尾部/分类过滤，且需要 `/usr/bin/log` 的免密 sudo 权限。
- 若本地有共享约束规则，需先评审；否则遵循本代码库指南。
- SwiftUI 状态管理（iOS/macOS）：优先使用 `Observation` 框架（`@Observable`、`@Bindable`），而非 `ObservableObject`/`@StateObject`；除非兼容性要求，否则禁止新增 `ObservableObject`，修改相关代码时需迁移现有用法。
- 连接提供者：添加新连接时，需更新所有 UI 界面和文档（macOS 应用、Web UI、移动端（如有）、入职引导/概览文档），并添加匹配的状态 + 配置表单，确保提供者列表和设置同步。
- 版本号位置：`package.json`（CLI）、`apps/android/app/build.gradle.kts`（versionName/versionCode）、`apps/ios/Sources/Info.plist` + `apps/ios/Tests/Info.plist`（CFBundleShortVersionString/CFBundleVersion）、`apps/macos/Sources/Moltbot/Resources/Info.plist`（CFBundleShortVersionString/CFBundleVersion）、`docs/install/updating.md`（固定 npm 版本）、`docs/platforms/mac/release.md`（APP_VERSION/APP_BUILD 示例）、Peekaboo Xcode 项目/Info.plist（MARKETING_VERSION/CURRENT_PROJECT_VERSION）。
- **重启应用**：“重启 iOS/Android 应用”指重新构建（重新编译/安装）并启动，而非仅终止/启动。
- **设备检查**：测试前，先验证已连接的真实设备（iOS/Android），再考虑模拟器/仿真器。
- iOS Team ID 查找：`security find-identity -p codesigning -v` → 使用 Apple Development（…）对应的 TEAMID。备用方案：`defaults read com.apple.dt.Xcode IDEProvisioningTeamIdentifiers`。
- A2UI 包哈希：`src/canvas-host/a2ui/.bundle.hash` 为自动生成；忽略非预期变更，仅必要时通过 `pnpm canvas:a2ui:bundle`（或 `scripts/bundle-a2ui.sh`）重新生成。哈希值需单独提交。
- 发布签名/公证密钥在代码库外管理；遵循内部发布文档。
- 公证认证环境变量（`APP_STORE_CONNECT_ISSUER_ID`、`APP_STORE_CONNECT_KEY_ID`、`APP_STORE_CONNECT_API_KEY_P8`）需已配置在环境中（参考内部发布文档）。
- **多代理安全规则**：除非明确要求，否则**禁止**创建/应用/删除 `git stash` 条目（包括 `git pull --rebase --autostash`）。假设可能有其他代理在工作；保持无关的未完成工作不变，避免跨域状态变更。
- **多代理安全规则**：用户要求“push”时，可执行 `git pull --rebase` 集成最新变更（禁止丢弃其他代理的工作）。用户要求“commit”时，仅提交自身修改。用户要求“commit all”时，按分组提交所有变更。
- **多代理安全规则**：除非明确要求，否则**禁止**创建/删除/修改 `git worktree` 检出（或编辑 `.worktrees/*`）。
- **多代理安全规则**：除非明确要求，否则**禁止**切换分支/检出其他分支。
- **多代理安全规则**：运行多个代理是允许的，只要每个代理有独立会话。
- **多代理安全规则**：遇到未识别文件时，继续执行；聚焦自身修改，仅提交这些变更。
- 代码检查/格式化变更处理：
  - 若暂存+未暂存的差异仅为格式化修改，自动解决无需询问。
  - 若已请求提交/推送，自动暂存并将仅格式化的后续修改纳入同一提交（必要时可创建小型后续提交），无需额外确认。
  - 仅当变更涉及语义（逻辑/数据/行为）时，才需询问用户。
- Lobster 样式规范：使用 `src/terminal/palette.ts` 中的共享 CLI 配色方案（禁止硬编码颜色）；入职引导/配置提示和其他 TTY UI 输出需按需应用该配色。
- **多代理安全规则**：报告聚焦自身修改；除非确实受阻，否则避免约束规则声明；多个代理修改同一文件时，若安全则继续执行；仅在相关时，在结尾简要说明“存在其他文件”。
- 问题排查：结论前需阅读相关 npm 依赖源码和所有本地相关代码；目标是定位高可信度根因。
- 代码风格：复杂逻辑添加简短注释；可行时保持文件行数在 ~500 行以内（按需拆分/重构）。
- 工具模式约束（google-antigravity）：工具输入模式中避免 `Type.Union`；禁止 `anyOf`/`oneOf`/`allOf`。字符串列表使用 `stringEnum`/`optionalStringEnum`（Type.Unsafe 枚举），可选类型使用 `Type.Optional(...)` 而非 `... | null`。顶级工具模式需为 `type: "object"` 并包含 `properties`。
- 工具模式约束：避免在工具模式中使用原生 `format` 属性名；部分验证器会将 `format` 视为保留关键字并拒绝该模式。
- 要求打开“会话”文件时，打开树莓派会话日志（路径：`~/.clawdbot/agents/<agentId>/sessions/*.jsonl`，使用系统提示 Runtime 行中的 `agent=<id>` 值；无指定 ID 时使用最新日志），而非默认的 `sessions.json`。若需其他机器的日志，通过 Tailscale SSH 连接并读取相同路径。
- 禁止通过 SSH 重建 macOS 应用；重建必须直接在 Mac 本机执行。
- 禁止向外部消息渠道（WhatsApp、Telegram）发送流式/部分回复；仅最终回复可发送至这些渠道。流式/工具事件仍可发送至内部 UI/控制渠道。
- 语音唤醒转发提示：
  - 命令模板需保持为 `moltbot-mac agent --message "${text}" --thinking low`；`VoiceWakeForwarder` 已对 `${text}` 做 shell 转义。禁止添加额外引号。
  - launchd 的 PATH 环境变量范围极小；确保应用的启动代理 PATH 包含标准系统路径和 pnpm 二进制目录（通常为 `$HOME/Library/pnpm`），以便通过 `moltbot-mac` 调用时能解析 `pnpm`/`moltbot` 二进制文件。
- 手动执行包含 `!` 的 `moltbot message send` 消息时，使用下文提到的 HereDoc 模式避免 Bash 工具的转义问题。
- 发布约束：无操作员明确同意，禁止修改版本号；执行任何 npm 发布/发布步骤前，务必征求许可。

## NPM + 1Password（发布/验证）
- 使用 1password 技能；所有 `op` 命令必须在全新 tmux 会话中执行。
- 登录：`eval "$(op signin --account my.1password.com)"`（应用已解锁 + 集成已开启）。
- 一次性验证码（OTP）：`op read 'op://Private/Npmjs/one-time password?attribute=otp'`。
- 发布：`npm publish --access public --otp="<otp>"`（从包目录执行）。
- 无本地 npmrc 副作用的验证：`npm view <包名> version --userconfig "$(mktemp)"`。
- 发布后终止 tmux 会话。