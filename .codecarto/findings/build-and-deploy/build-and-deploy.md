# Build and Deploy

## 2026-08-19 — architecture phase

### Build tooling
- **pnpm 11.7.0** workspaces; `linkWorkspacePackages: true`; `allowBuilds` allowlist (esbuild, lefthook, node-pty, koffi); `patchedDependencies` (node-pty@1.2.0-beta.15).
- **TypeScript**: `tsc -b` project references (`tsconfig.host.json`, `tsconfig.client.json`) emit `lib/types`; `tsdown` bundles runtime to `lib/`; `vite build` for web frontend.
- **Build orchestration**: `scripts/build.ts` (`pnpm run build`), `--profile official` variant.

### Multi-target / multi-stage builds
- Host face (`DSH_BUILD_FACE=host`) and client face (`DSH_BUILD_FACE=client`) — separate tsdown bundles.
- Single-exe build for Python runtime: `scripts/build-exe-for-python-sdk.ts` (bundles the `dsh-jsonrpc-agent-pkg` closure).
- Native Landlock launcher: per-architecture npm packages (`entry`, `linux-x64`, `linux-arm64`).

### Output artifacts
- npm packages (`@deepseek-ai/dsh-*`, `@deepseek-ai/cordis*`), Python wheels (`deepseek-harness-sdk`, `deepseek-harness-runtime-bin`), single-exe runtime, web `dist/`.

### CI/CD
- **GitHub Actions** (`.github/workflows/`): ci.yml, release.yml, sandbox.yml, e2e.yml, e2b-e2e.yml, pi-ai-provider-e2e.yml, landlock-run.yml, landlock-run-release.yml, python-release.yml, release-vendor.yml, docs-pages.yml, issue-lifecycle.yml, issue-policy.yml, build-exe-for-python-sdk.yml, expected-filenames.yml.
- **GitLab CI** (`.gitlab-ci.yml`): Python wheel build/publish (manylinux_2_28, glibc ≤ 2.28, uv 0.11.23, smoke tests via `scripts/smoke-python-runtime.py`).

### Platform-specific packaging
- Linux: bwrap (bubblewrap) + Landlock sandbox; manylinux_2_28 wheels.
- macOS: Seatbelt sandbox; node-pty macOS spawn helper executable-bit restore.
- Windows: Windows ACL sandbox; ConPTY via node-pty; koffi for JSONL `MoveFileExW` write-through.

### Release
- Tag-driven: `v*` tags → npm publish + GitHub release; `python-v*` tags → Python wheels. `scripts/release/*` (bump, verify, pack, verify-packed-install, publish).

## 2026-08-19 — porting phase

Synthesis-level build additions:
- The build pipeline (pnpm + tsc project references + tsdown + vite) is source-specific; a reimplementation's build is incidental to the port.
- The single-exe Python runtime build (`scripts/build-exe-for-python-sdk.ts`) and the native Landlock launcher are optional surfaces.
