# Python 项目开发规范
## 代码库规范
- 代码库地址：示例 `https://github.com/your-org/your-python-project`（替换为实际仓库地址）
- GitHub 问题/评论/PR 评论：使用字面量多行字符串或 `-F - <<'EOF'`（或 `$'...'`）实现真实换行；禁止嵌入 `\\n` 转义字符。

## 项目结构与模块组织
- 源代码：统一放在 `src/` 目录（CLI 核心逻辑在 `src/cli`，命令实现在 `src/commands`，Web 服务在 `src/web`，基础设施代码在 `src/infra`，数据处理流水线在 `src/pipelines`）。
- 测试文件：与源码同目录创建 `test_*.py`（遵循 pytest 规范）；端到端测试统一放在 `tests/e2e/` 目录（如需区分测试类型）。
- 文档：所有文档放在 `docs/` 目录（包含图片、队列说明、部署配置等）；Sphinx 构建后的文档产物输出在 `docs/_build/`。
- 插件/扩展：放在 `extensions/*` 目录（作为可独立安装的子包）；插件专属依赖仅声明在插件目录的 `pyproject.toml`/`requirements.txt` 中；除非核心代码依赖，否则禁止添加到根目录依赖文件。
- 插件安装与依赖：插件安装时执行 `pip install --no-dev .`（插件目录下）；运行时依赖必须声明在 `install_requires` 中（setup.cfg/Poetry 配置）。避免在运行时依赖中使用 `editable` 安装方式（pip 可能解析失败）；将核心包放在 `dev-dependencies` 或 `peer-dependencies`（Poetry 支持），运行时通过 `importlib` 别名解析插件 SDK。
- 安装脚本分发：若需提供安装脚本，放在 `public/install.sh`/`public/install.ps1`（建议在同级仓库 `../your-installer-repo` 中维护）。
- 功能模块扩展原则：重构共享逻辑（路由、白名单、命令权限、引导流程、文档）时，必须兼容所有内置模块 + 扩展模块（如 `extensions/msteams`、`extensions/matrix`、`extensions/voice` 等）。
  - 核心模块文档：`docs/modules/`
  - 核心模块代码：`src/telegram`、`src/discord`、`src/web`（FastAPI/Flask）、`src/routing`、`src/channels`
  - 扩展模块（插件）：`extensions/*`
- 新增模块/扩展/文档时，需检查 `.github/labeler.yml` 确保标签覆盖完整。

## 文档链接规范（Sphinx/MkDocs）
- 文档托管：推荐基于 MkDocs（`docs.your-project.com`）或 Read the Docs（`your-project.readthedocs.io`）托管。
- 内部文档链接（`docs/**/*.md`/`.rst` 中）：使用根相对路径，不带 `.md`/`.rst` 后缀（示例：`[配置](/configuration)`）。
- 章节交叉引用：根相对路径 + 锚点（示例：`[钩子](/configuration#hooks)`）。
- 文档标题与锚点：标题中避免使用破折号和撇号，防止 MkDocs/Read the Docs 锚点链接失效。
- 需提供完整链接时：返回全量 `https://docs.your-project.com/...`（而非根相对路径）。
- README（GitHub 端）：保留绝对文档 URL（`https://docs.your-project.com/...`），确保 GitHub 上链接可正常访问。
- 文档内容规范：内容通用化，不出现个人设备名/主机名/路径；使用占位符如 `user@deploy-host`、“部署主机”。

## 部署主机/虚拟机操作规范（通用）
- 访问方式：固定路径 `ssh deploy-host` 后执行 `ssh vm-name`（假设 SSH 密钥已配置完成）。
- SSH 连接不稳定：使用部署主机的 Web 终端；长时间操作需保留 tmux 会话防止断开。
- 版本更新：全局安装执行 `sudo pip install your-project@latest`（需 `/usr/lib/python3/site-packages` 写权限）；**推荐使用 venv 环境隔离**：`source /opt/your-project/venv/bin/activate && pip install your-project@latest`。
- 配置管理：使用自定义 CLI 命令 `your-project config set ...` 或直接修改 `.env` 文件；确保已配置 `deploy.mode=local`。
- 第三方密钥（如 Discord）：仅存储原始 Token（无 `DISCORD_BOT_TOKEN=` 前缀）。
- 服务重启（systemd 管理）：
  ```bash
  sudo systemctl stop your-project-gateway || true
  sudo systemctl start your-project-gateway
  # 无 systemd 环境时执行：
  pkill -9 -f your-project-gateway || true; nohup python -m src.gateway run --bind loopback --port 18789 --force > /tmp/your-project.log 2>&1 &
  ```
- 状态验证：
  ```bash
  your-project status --probe
  ss -ltnp | rg 18789
  tail -n 120 /tmp/your-project.log
  ```

## 构建、测试与开发命令
- 运行时基线：Python **3.10+**（兼容 3.11/3.12 版本，保持 venv/conda 环境适配）。
- 依赖安装（三种方式，任选其一）：
  - 首选 Poetry：`poetry install`
  - 兼容 Pip：`pip install -r requirements-dev.txt`
  - 支持 Pipenv：`pipenv install --dev`
- 预提交钩子：`pre-commit install`（与 CI 执行相同检查，如 black/isort/flake8 代码格式化与检查）。
- 运行开发版 CLI：
  - Poetry 环境：`poetry run your-project ...`
  - Venv 环境：`source venv/bin/activate && python -m src.cli ...`
- 打包构建：
  - 源码包/Wheel 包：`poetry build` 或 `python -m build`
  - 桌面应用打包（如需）：`pyinstaller src/main.spec`（支持 Mac/Linux/Windows 跨平台）
- 类型检查：`mypy src/`（开启严格模式 `strict = true`）。
- 代码格式化/静态检查：
  - 自动格式化：`black src/ tests/` + `isort src/ tests/`（代码格式化 + 导入排序）
  - 静态检查：`flake8 src/ tests/ --max-line-length=120`
- 测试执行：`pytest`（覆盖率检测：`pytest --cov=src --cov-report=term --cov-fail-under=70`）。

## 编码风格与命名规范
- 语言规范：Python 3.10+（强制使用类型注解，避免使用 `Any`；复杂类型使用 `typing`/`typing_extensions` 模块）。
- 格式化/检查：基于 Black + Isort + Flake8 实现；代码提交前必须执行 `pre-commit run --all-files`。
- 代码注释：复杂/非直观逻辑添加简洁注释；核心函数/类必须添加 docstring 文档（遵循 Google 风格或 NumPy 风格）。
- 文件规范：保持文件简洁，提取辅助函数而非复制粘贴“V2 版本”代码；CLI 选项和依赖注入复用现有项目模式。
- 文件行数：目标单文件代码量低于 700 行（仅指导原则，非硬性限制）；若拆分能提升可读性/可测试性，立即重构。
- 命名规则：
  - 产品/文档标题：使用 **YourProject**（项目全称，首字母大写）。
  - CLI 命令/包名/路径/配置键：使用 `your-project`（小写连字符）或 `your_project`（小写下划线，符合 Python 包规范）。
  - 代码内命名：严格遵循 PEP8 规范——变量/函数用 `snake_case`，类用 `CamelCase`，常量用 `UPPER_SNAKE_CASE`。

## 发布渠道（命名规范）
- stable（稳定版）：仅打标签发布（示例 `vYYYY.M.D`），PyPI 标签为 `latest`。
- beta（测试版）：预发布标签 `vYYYY.M.D-beta.N`，PyPI 标签为 `beta`（可暂不发布桌面打包产物）。
- dev（开发版）：`main` 分支最新提交（无标签；需测试时直接 checkout main 分支）。

## 测试规范
- 测试框架：使用 pytest（行/分支/函数/语句覆盖率阈值 70%）。
- 命名规范：测试文件与源码同名，后缀为 `test_*.py`；端到端测试标注为 `*_e2e_test.py`。
- 提交前检查：修改业务逻辑后，必须运行 `pytest`（或 `pytest --cov` 检测覆盖率）。
- 测试并发：测试工作线程数不超过 16（已验证过多线程会导致资源冲突）。
- 真实环境测试（含密钥）：
  ```bash
  LIVE_TEST=1 pytest tests/live/
  # Docker 环境测试：
  docker-compose run test-service pytest tests/docker/
  ```
- 测试文档：完整的测试覆盖说明见 `docs/testing.md`。
- 变更日志规则：纯测试代码的新增/修复无需添加 Changelog，除非影响用户可见行为或用户明确要求。
- 移动端测试：优先使用真实设备（iOS/Android），无真实设备时再使用模拟器。

## 提交与拉取请求（PR）规范
- 提交规范：使用 `cz commit`（Commitizen）或手动遵循 Conventional Commits 规范；避免手动 `git add`/`git commit`，确保暂存区仅包含相关变更。
- 提交信息：简洁、面向动作（示例：`CLI: 为 send 命令添加详细日志标志`）。
- 变更分组：相关变更合并为一个提交；禁止将无关重构混入同一提交。
- 变更日志流程：最新发布版本保持在文档顶部（无 `Unreleased` 章节）；发布后更新版本号并新建顶部章节。
- PR 描述：总结变更范围、说明已执行的测试，注明用户可见的变更/新增参数。
- PR 评审流程：
  - 收到 PR 链接后，通过 `gh pr view`/`gh pr diff` 进行评审；**禁止切换分支**、**禁止修改代码**。
  - 评审前执行 `git pull`；若本地有未推送变更/冲突，暂停评审并告知用户。
- PR 合并原则：优先合并 PR；提交记录清晰时使用 **rebase**，提交历史混乱时使用 **squash**。
- PR 合并流程：
  1. 从 `main` 分支创建临时集成分支。
  2. 将 PR 分支合并到临时分支（优先 rebase 保证线性历史；复杂冲突时允许普通 merge）。
  3. 修复问题、添加 Changelog（包含 PR 编号 + 感谢贡献者）。
  4. 本地执行全量检查（`pre-commit run --all-files && pytest`）。
  5. 提交后合并回 `main`，删除临时分支，切回 `main` 分支。
- 贡献者归属：
  - 若评审后自行修改 PR 代码，合并时需将原作者设为共同贡献者。
  - 合并新贡献者的 PR 后，将其头像添加到 README 的“Contributors”区域。
  - 运行 `python scripts/update-contributors.py` 自动更新贡献者列表，并提交该变更。

## 快捷命令
- `sync`：若工作区有未提交变更，自动提交所有修改（生成合规的 Conventional Commits 提交信息），执行 `git pull --rebase`；若 rebase 冲突无法解决则暂停；否则执行 `git push`。

### PR 工作流（评审 vs 合入）
- **评审模式（仅提供 PR 链接）**：仅读取 `gh pr view/diff` 内容；**禁止切换分支**、**禁止修改代码**。
- **合入模式**：
  1. 从 `main` 分支创建集成分支。
  2. 合入 PR 提交（优先 rebase 保证线性历史；复杂冲突时允许 **普通 merge**）。
  3. 修复问题、补充 Changelog（含致谢语 + PR 编号）。
  4. 本地执行全量校验（`pre-commit run --all-files && pytest`）。
  5. 提交后合并回 `main`，执行 `git switch main`（合入后绝不留在特性分支）。
  核心要求：贡献者信息必须出现在 git 提交记录中！

## 安全与配置规范
- 凭据存储：Web 服务凭据统一存储在 `~/.your-project/credentials/`；若登出需重新执行 `your-project login`。
- 会话存储：默认可选会话文件放在 `~/.your-project/sessions/`；基础目录不允许自定义配置。
- 环境变量：基于 `python-dotenv` 加载，配置文件放在 `~/.profile` 或项目根目录 `.env`（**禁止提交 .env 文件到仓库**）。
- 敏感信息：绝不提交/发布真实手机号、密钥、配置值；文档/测试/示例中使用明显的假数据（如 `user@example.com`、`fake-token-123456`）。
- 发布流程：发布前必须阅读 `docs/reference/RELEASING.md`；文档已覆盖的常规问题，禁止重复提问。

## 故障排除
- 迁移/遗留配置警告：执行 `your-project doctor` 命令排查（参考 `docs/gateway/doctor.md`）。
- 依赖冲突：执行 `poetry lock --no-update` 或 `pip check` 排查；优先使用 Poetry 锁文件保证开发环境一致性。
- 类型检查错误：确认 `mypy` 配置（`pyproject.toml` 中 `[tool.mypy]` 节点）与代码注解匹配；复杂类型使用 `TypeGuard`/`cast` 明确标注。

## 项目专属说明
- 打包备注：“mac app” 特指基于 PyQt/Tkinter 打包的 macOS 桌面应用。
- 依赖修改规范：**禁止直接编辑 `site-packages` 目录**（全局/venv/conda 安装均禁止）；需修改依赖时通过 fork 仓库 + 补丁实现，补丁记录在 `patches/` 目录。
- PyPI 发布规范：
  - 必须在全新的 tmux 会话中执行 `twine` 发布命令。
  - 登录认证：`twine login --repository pypi`（使用 PyPI API Token）。
  - 发布包：`twine upload dist/*`。
  - 发布验证：`pip install your-project==<version> --userconfig $(mktemp)`。
  - 发布完成后关闭 tmux 会话。
- 配置校验：使用 `pydantic` 验证配置文件/环境变量，避免运行时配置错误。
- 日志规范：统一使用 Python 内置 `logging` 模块（禁止使用 `print`）；日志级别/格式配置在 `src/logging.py`。
- 多开发者协作安全规范：
  - 除非明确要求，否则不创建/应用/删除 `git stash` 条目（包括 `git pull --rebase --autostash`）。
  - 执行 `push` 前，先执行 `git pull --rebase` 整合最新变更（**绝不丢弃他人工作成果**）。
  - 执行 `commit` 时仅提交自身变更；使用 `commit all` 时按逻辑分组提交。
  - 不创建/删除/修改 `git worktree`；除非明确要求，否则不切换分支。
- 格式化变更处理：
  - 若暂存/未暂存变更仅为代码格式化修改，自动解决无需确认。
  - 若已请求 `commit/push`，自动暂存格式化修改并合并到同一提交（必要时可单独提交小型变更），无需额外确认。
  - 仅当变更涉及业务逻辑（数据/行为）时，才向用户确认。
- 版本号存放位置：
  - `pyproject.toml`（Poetry）/`setup.cfg`（setuptools）：项目核心版本号。
  - `apps/android/build.gradle.kts`：`versionName`/`versionCode`（若有移动端支持）。
  - `apps/ios/Info.plist`：`CFBundleShortVersionString`/`CFBundleVersion`（若有 iOS 应用）。
  - `docs/install/updating.md`：固定的 PyPI 版本号。
- 应用重启：“重启 iOS/Android 应用”指重新编译、安装并启动，而非仅杀死进程。
- 设备检查：测试前优先验证已连接的真实设备（iOS/Android），再考虑使用模拟器。
- 发布权限：**无明确授权绝不擅自修改版本号**；执行 PyPI 发布操作前必须获得清晰的权限确认。
