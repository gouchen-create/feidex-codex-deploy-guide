# Contributing

This repository is a living deployment guide for Feidex + Feishu + Codex on new Windows computers.

## Required Update Rule

When a deployment encounters a problem that is not covered by `README.md`, and Codex finds and verifies a fix, update the guide before finishing the deployment.

Add the new experience to the most relevant section with:

- Trigger condition
- Original error or key log lines
- Root cause
- Copy-pasteable fix commands
- Verification command and passing standard

## Secret Safety

Never commit:

- `app_secret`
- Full `config.toml`
- GitHub tokens
- Feishu/Lark access tokens
- Private keys
- Local runtime logs that include secrets

If configuration examples are needed, use placeholders such as `APP_SECRET_HERE`.

## Pull Request Title

Use this format:

```text
docs: 补充 <电脑/环境/报错关键词> 部署排障经验
```
