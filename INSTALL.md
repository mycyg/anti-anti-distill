# 安装指南

## Claude Code

> **重要**：Claude Code 从 **git 仓库根目录** 的 `.claude/skills/` 查找 skill。请在正确的位置执行。

### 安装到当前项目

```bash
# 在 git 仓库根目录执行
mkdir -p .claude/skills
git clone <repo-url> .claude/skills/anti-anti-distill
```

### 安装到全局

```bash
git clone <repo-url> ~/.claude/skills/anti-anti-distill
```

## OpenClaw

```bash
git clone <repo-url> ~/.openclaw/workspace/skills/anti-anti-distill
```

## 无额外依赖

本 Skill 仅使用 Claude Code 内置工具（Read、Write、Edit、Bash、Glob、Grep），无需安装 Python 依赖或配置 API 凭证。

## 验证安装

在 Claude Code 中输入：

```
/anti-anti-distill
```

如果看到检测提示，说明安装成功。

## 配合使用

### 配合 colleague-skill 使用

```bash
# 安装同事 skill 创建器
git clone https://github.com/titanwings/colleague-skill .claude/skills/create-colleague

# 安装反反蒸馏检测器
git clone <repo-url> .claude/skills/anti-anti-distill
```

流程：
1. 用 `/create-colleague` 让同事生成 Skill 文件
2. 同事提交后，用 `/anti-anti-distill` 检测是否被清洗
3. 根据审计报告决定是否退回补充
