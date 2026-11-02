---
name: npm-publish
description: >-
  用本机 NPM_TOKEN 发布 npm 包。用户要 npm publish、发到 npm、打 dist-tag、
  或发布流程跳过了 NPM_TOKEN 时使用。Windows 上进程环境经常没有用户级
  NPM_TOKEN，~/.npmrc 里的旧令牌会 401；GitHub Actions 的 secrets.NPM_TOKEN
  也是另一处，仓库没配就会跳过发布。
metadata:
  short-description: 用用户级 NPM_TOKEN 发布 npm
---

# npm-publish

## 何时使用

用户要求把当前包发布到 npm，或发布日志出现 `NPM_TOKEN is not configured; skipping npm publish`。

不要用于只提交代码、只推 Git tag、或只查包信息。

## 令牌从哪来

三处互不相通，不要混用：

1. **当前进程环境** `NPM_TOKEN` 或 `NODE_AUTH_TOKEN`。DeepSeek Harness 拉起的命令经常没有用户级变量。
2. **Windows 用户环境** `[Environment]::GetEnvironmentVariable('NPM_TOKEN','User')`，没有再查 `Machine`。本机有效令牌在这里。
3. **GitHub Actions** `secrets.NPM_TOKEN`。只给 CI 用。本地有用户变量，不代表仓库 secret 已配置。

`~/.npmrc` 的 `//registry.npmjs.org/:_authToken` 可能已失效。`npm whoami` 返回 401 时不要改用它，也不要把它打印出来。

## 发布

先确认 `package.json` 的版本还没在 registry 上。预发布版本不要覆盖 `latest`；包里 `publishConfig.tag` 是 `alpha` 时，发布参数用 `--tag alpha`。

在 PowerShell 里取令牌、写临时 npmrc、发布、删掉临时文件。命令文本里不要出现令牌值：

```powershell
$token = [Environment]::GetEnvironmentVariable('NPM_TOKEN','User')
if (-not $token) { $token = [Environment]::GetEnvironmentVariable('NPM_TOKEN','Machine') }
if (-not $token) { throw 'NPM_TOKEN missing' }
$npmrc = Join-Path $env:TEMP ('npmrc-publish-' + [guid]::NewGuid().ToString('n'))
try {
  Set-Content -Path $npmrc -Value "registry=https://registry.npmjs.org/`n//registry.npmjs.org/:_authToken=$token" -Encoding ascii
  npm publish --userconfig $npmrc --tag alpha
} finally {
  Remove-Item -LiteralPath $npmrc -Force -ErrorAction SilentlyContinue
}
```

进程里已经有非空 `NPM_TOKEN` 时，用进程里的值，不必再读用户环境。

`npm publish` 可能对 PUT 返回 202 并退出 0，版本要过一会儿才出现。用一份**不含** `~/.npmrc` 的临时配置复查：

```powershell
npm view <name> versions dist-tags --userconfig <empty-npmrc> --json
```

空配置只写 `registry=https://registry.npmjs.org/`。列表里有这个版本，且预期的 dist-tag 指向它，才算发布完成。

## CI

本地用用户环境发布成功，不会给 GitHub 仓库写入 `NPM_TOKEN`。Actions 仍会打印 `skipping npm publish`，直到在仓库 Settings → Secrets 里单独配置 `NPM_TOKEN`。不要把用户令牌写进仓库、日志或 workflow 文件。

插件编写与市场收录规范见 `dsh-plugin-creator` 技能（源 `D:\tools\skills\dsh-plugin-creator`）。
