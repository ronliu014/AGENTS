# Python Project Guidelines
## Repository Guidelines
- Repo: Example `https://github.com/your-org/your-python-project` (replace with the actual repository URL)
- GitHub issues/comments/PR comments: Use literal multiline strings or `-F - <<'EOF'` (or `$'...'`) for real line breaks; never embed `\\n`.

## Project Structure & Module Organization
- Source code: `src/` (CLI logic in `src/cli`, command implementations in `src/commands`, web services in `src/web`, infrastructure code in `src/infra`, data processing pipelines in `src/pipelines`).
- Tests: Co-located `test_*.py` (follow pytest conventions); end-to-end tests are uniformly placed in `tests/e2e/` (if differentiation is needed).
- Docs: `docs/` (including images, queue descriptions, deployment configurations, etc.). Built documentation output is in `docs/_build/` (Sphinx build artifacts).
- Plugins/extensions: Placed in `extensions/*` (as installable subpackages). Plugin-specific dependencies are only in the `pyproject.toml`/`requirements.txt` of the plugin directory; do not add them to the root dependency files unless required by core code.
- Plugins: Run `pip install --no-dev .` in the plugin directory during installation; runtime dependencies must be in `install_requires` (setup.cfg/poetry). Avoid `editable` installation in runtime dependencies (pip may fail to parse); place core packages in `dev-dependencies` or `peer-dependencies` (supported by poetry), and resolve the plugin SDK via `importlib` aliases at runtime.
- Installers distribution path: If installation scripts are required, place them in `public/install.sh`/`public/install.ps1` (maintained in the sibling repo `../your-installer-repo`).
- Functional module extension: When refactoring shared logic (routing, allowlists, command permissions, onboarding processes, documentation), consider all built-in + extension modules (e.g., `extensions/msteams`, `extensions/matrix`, `extensions/voice`, etc.).
  - Core module docs: `docs/modules/`
  - Core module code: `src/telegram`, `src/discord`, `src/web` (FastAPI/Flask), `src/routing`, `src/channels`
  - Extension modules (plugins): `extensions/*`
- When adding new modules/extensions/docs, check `.github/labeler.yml` to ensure full label coverage.

## Docs Linking (Sphinx/MkDocs)
- Docs hosting: Example based on MkDocs (`docs.your-project.com`) or Read the Docs (`your-project.readthedocs.io`).
- Internal doc links (in `docs/**/*.md`/`.rst`): Root-relative paths without `.md`/`.rst` suffixes (example: `[Config](/configuration)`).
- Section cross-references: Root-relative paths + anchors (example: `[Hooks](/configuration#hooks)`).
- Doc headings & anchors: Avoid em dashes and apostrophes in headings to prevent broken anchor links in MkDocs/Read the Docs.
- When full links are required: Return the complete `https://docs.your-project.com/...` (instead of root-relative paths).
- README (GitHub): Keep absolute doc URLs (`https://docs.your-project.com/...`) to ensure links work properly on GitHub.
- Doc content guidelines: Generalize content without personal device/hostname/path references; use placeholders such as `user@deploy-host` and "deployment host".

## Deployment Host/VM Operations (General)
- Access method: Stable path `ssh deploy-host` then `ssh vm-name` (assuming SSH keys are configured).
- Unstable SSH: Use the deployment host's web terminal; keep a tmux session for long-term operations.
- Version update: `sudo pip install your-project@latest` (global installation requires write permission for `/usr/lib/python3/site-packages`); venv isolation is recommended: `source /opt/your-project/venv/bin/activate && pip install your-project@latest`.
- Configuration management: Use `your-project config set ...` (custom CLI) or modify the `.env` file directly; ensure `deploy.mode=local` is configured.
- Third-party keys (e.g., Discord): Store only the raw Token (without the `DISCORD_BOT_TOKEN=` prefix).
- Service restart (managed by systemd):
  ```bash
  sudo systemctl stop your-project-gateway || true
  sudo systemctl start your-project-gateway
  # Without systemd:
  pkill -9 -f your-project-gateway || true; nohup python -m src.gateway run --bind loopback --port 18789 --force > /tmp/your-project.log 2>&1 &
  ```
- Status verification:
  ```bash
  your-project status --probe
  ss -ltnp | rg 18789
  tail -n 120 /tmp/your-project.log
  ```

## Build, Test, and Development Commands
- Runtime baseline: Python **3.10+** (compatible with 3.11/3.12, maintain venv/conda environment adaptability).
- Dependency installation:
  - Preferred: Poetry: `poetry install`
  - Pip compatible: `pip install -r requirements-dev.txt`
  - Pipenv also supported: `pipenv install --dev`
- Pre-commit hooks: `pre-commit install` (runs the same checks as CI, e.g., black/isort/flake8).
- Run dev version CLI:
  - Poetry: `poetry run your-project ...`
  - Venv: `source venv/bin/activate && python -m src.cli ...`
- Packaging & building:
  - Source distribution/Wheel: `poetry build` or `python -m build`
  - Desktop app packaging (if needed): `pyinstaller src/main.spec` (Mac/Linux/Windows)
- Type checking: `mypy src/` (enable strict mode with `strict = true`).
- Code formatting/checking:
  - Formatting: `black src/ tests/` + `isort src/ tests/`
  - Static checking: `flake8 src/ tests/ --max-line-length=120`
- Test execution: `pytest` (coverage: `pytest --cov=src --cov-report=term --cov-fail-under=70`).

## Coding Style & Naming Conventions
- Language standard: Python 3.10+ (mandatory type hints, avoid `Any`; use `typing`/`typing_extensions` for complex types).
- Formatting/checking: Based on Black + Isort + Flake8; must run `pre-commit run --all-files` before submission.
- Code comments: Add concise comments for complex/non-intuitive logic; add docstrings for key functions/classes (follow Google or NumPy style).
- File guidelines: Keep files concise, extract helper functions instead of copying and pasting "V2 version" code; reuse existing patterns for CLI options and dependency injection.
- File line count: Target a single file with less than ~700 lines of code (guideline only); refactor immediately if splitting improves readability/testability.
- Naming rules:
  - Product/doc headings: Use **YourProject** (full project name).
  - CLI commands/package names/paths/config keys: Use `your-project` (lowercase hyphen) or `your_project` (lowercase underscore, compliant with Python package standards).
  - In-code naming: Follow PEP8 — variables/functions use `snake_case`, classes use `CamelCase`, constants use `UPPER_SNAKE_CASE`.

## Release Channels (Naming)
- stable: Tagged releases only (example `vYYYY.M.D`), PyPI tag `latest`.
- beta: Pre-release tags `vYYYY.M.D-beta.N`, PyPI tag `beta` (desktop packaging artifacts may be omitted temporarily).
- dev: Latest commits on the `main` branch (no tag; check out the main branch directly for testing).

## Testing Guidelines
- Test framework: pytest (coverage threshold 70% for lines/branches/functions/statements).
- Naming conventions: Test files have the same name as the source code with the `test_*.py` suffix; end-to-end tests are labeled `*_e2e_test.py`.
- Pre-submission check: After modifying business logic, must run `pytest` (or `pytest --cov`).
- Test concurrency: Do not set the number of test worker threads above 16 (verified that excessive threads cause resource conflicts).
- Real environment testing (including keys):
  ```bash
  LIVE_TEST=1 pytest tests/live/
  # Docker environment testing:
  docker-compose run test-service pytest tests/docker/
  ```
- Test documentation: Complete test coverage description is in `docs/testing.md`.
- Changelog rules: Pure test additions/fixes do not require a Changelog entry unless they affect user-visible behavior or are explicitly requested by the user.
- Mobile testing: Prefer real devices (iOS/Android) when available; use simulators only if no real devices are present.

## Commit & Pull Request Guidelines
- Submission standard: Use `cz commit` (Commitizen) or manually follow the Conventional Commits standard; avoid manual `git add`/`git commit` to ensure only relevant changes are in the staging area.
- Commit messages: Concise and action-oriented (example: `CLI: add verbose flag to send command`).
- Change grouping: Merge related changes into a single commit; avoid mixing unrelated refactoring into the same commit.
- Changelog workflow: Keep the latest released version at the top (no `Unreleased` section); update the version number and create a new top section after release.
- PR description: Summarize the scope of changes, test execution status, and note user-visible changes/new parameters.
- PR review process:
  - After receiving the PR link, review via `gh pr view`/`gh pr diff`; **do not switch branches** and **do not modify code**.
  - Run `git pull` before review; if there are uncommitted local changes/conflicts, pause the review and inform the user.
- PR merge target: Prioritize merging PRs; use **rebase** for clear commit history, and **squash** for messy history.
- PR merge process:
  1. Create a temporary branch from `main`.
  2. Merge the PR branch into the temporary branch (prefer rebase to ensure linear history; allow merge for complex conflicts).
  3. Fix issues and add Changelog entries (including PR number + contributor thanks).
  4. Run full local checks (`pre-commit run --all-files && pytest`).
  5. Commit and merge back to `main`, delete the temporary branch, and switch back to `main`.
- Contributor attribution:
  - If you modify the PR yourself after review, set the original author as a co-contributor when merging.
  - After merging a PR from a new contributor, add their avatar to the "Contributors" section of the README.
  - Run `python scripts/update-contributors.py` to automatically update the contributor list and submit the changes.

## Shorthand Commands
- `sync`: If the working directory has changes, commit all modifications (automatically generate compliant Conventional Commits messages), execute `git pull --rebase`; pause if rebase conflicts cannot be resolved; otherwise execute `git push`.

### PR Workflow (Review vs Land)
- **Review mode (PR link only)**: Only read `gh pr view/diff`; **do not switch branches** and **do not modify code**.
- **Landing mode**:
  1. Create an integration branch from `main`.
  2. Import PR commits (prefer rebase to ensure linear history; allow **merge** for complex conflicts).
  3. Fix issues and supplement the Changelog (including thanks + PR number).
  4. Run full local verification (`pre-commit run --all-files && pytest`).
  5. Commit and merge back to `main`, execute `git switch main` (never stay on a feature branch after merging).
  Key requirement: Contributor information must appear in the git commit history!

## Security & Configuration Tips
- Credential storage: Web service credentials are stored in `~/.your-project/credentials/`; re-run `your-project login` if logged out.
- Session storage: Optional default session files are placed in `~/.your-project/sessions/`; the base directory is not configurable.
- Environment variables: Loaded based on `python-dotenv`, configuration files are placed in `~/.profile` or the project root `.env` (**do not submit .env**).
- Sensitive information: Never submit/publish real phone numbers, keys, or configuration values; use obvious fake data in docs/tests/examples (e.g., `user@example.com`, `fake-token-123456`).
- Release process: Must read `docs/reference/RELEASING.md` before release; do not ask repetitive questions for routine issues covered in the documentation.

## Troubleshooting
- Migration/legacy configuration warnings: Execute `your-project doctor` (refer to `docs/gateway/doctor.md`).
- Dependency conflicts: Execute `poetry lock --no-update` or `pip check` to troubleshoot; prefer using Poetry lock files to ensure environment consistency.
- Type checking errors: Confirm that the `mypy` configuration (`[tool.mypy]` in `pyproject.toml`) matches the code annotations; use `TypeGuard`/`cast` to explicitly label complex types.

## Project-Specific Notes
- Packaging notes: "mac app" refers to the macOS desktop app packaged based on PyQt/Tkinter.
- Dependency modification: **Never edit the `site-packages` directory directly** (for global/venv/conda installations); implement modifications via fork + patches if needed, and record patches in `patches/`.
- PyPI release:
  - Must execute `twine` release commands in a fresh tmux session.
  - Login authentication: `twine login --repository pypi` (use API Token).
  - Release: `twine upload dist/*`.
  - Verification: `pip install your-project==<version> --userconfig $(mktemp)`.
  - Close the tmux session after release.
- Configuration verification: Use `pydantic` to verify configuration files/environment variables to avoid runtime configuration errors.
- Logging standards: Uniquely use the `logging` module (instead of `print`); log level/format configuration is in `src/logging.py`.
- Multi-developer collaboration security:
  - Do not create/apply/delete `git stash` entries unless explicitly requested (including `git pull --rebase --autostash`).
  - Execute `git pull --rebase` to integrate the latest changes before `push` (**never discard others' work**).
  - Only submit your own changes when executing `commit`; group commits by logic when using `commit all`.
  - Do not create/delete/modify `git worktree`; do not switch branches unless explicitly requested.
- Formatting change handling:
  - If staged/unstaged changes are only formatting modifications, resolve them automatically without confirmation.
  - If `commit/push` is already requested, automatically stage formatting modifications and merge them into the same commit (a separate small commit if necessary), no additional confirmation required.
  - Only confirm with the user when changes involve business logic (data/behavior).
- Version number locations:
  - `pyproject.toml` (Poetry)/`setup.cfg` (setuptools): Core version number.
  - `apps/android/build.gradle.kts`: `versionName`/`versionCode` (if mobile support is available).
  - `apps/ios/Info.plist`: `CFBundleShortVersionString`/`CFBundleVersion` (if iOS app is available).
  - `docs/install/updating.md`: Pinned PyPI version number.
- App restart: "Restart iOS/Android apps" means recompile, install and restart, not just kill the process.
- Device check: Prior to testing, verify connected real devices (iOS/Android) first, then consider simulators.
- Release permission: **Never modify the version number without explicit authorization**; must obtain clear permission before executing PyPI release operations.
