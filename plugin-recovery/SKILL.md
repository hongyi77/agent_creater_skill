---
name: plugin-recovery
description: >-
  在 Claude Code 升级后检查和恢复插件状态。当插件列表为空、插件状态显示 "failed to load"、/plugin 命令报错、或用户反馈插件丢失/禁用时触发。
  也适用于手动恢复了 installed_plugins.json 但需要重建市场配置的场景。
  升级后首次启动时如果检测到插件异常，主动建议运行此技能。
  注意：此技能仅修复已安装过但因升级丢失注册信息的插件（检查 settings.json 中的 extraKnownMarketplaces 和 installed_plugins.json），不会安装新插件。
---

# Plugin Recovery Skill

## 问题背景

Claude Code 大版本升级时，插件系统的数据格式可能发生变化，导致：
- `installed_plugins.json` 被清空或格式不兼容
- marketplace 配置丢失（旧版不写入 `extraKnownMarketplaces`）
- `settings.json` 中 `enabledPlugins` 残留对不存在插件的引用
- 此前添加过的市场不再出现在 `claude plugin marketplace list` 中

## 检查清单

按顺序执行以下检查，不跳跃，每步完成后才进入下一步。

### 1. 检查网络连接

先确认是否能访问 GitHub，因为后续恢复操作需要拉取 marketplace：

```
curl -sk -o /dev/null -w "%{http_code}" https://github.com --connect-timeout 10
# 返回 200 说明网络连通
# 返回 exit code 35 或 000 说明网络不通
```

如果网络不通，**不要继续**，告诉用户：
> "GitHub 当前无法访问，需要先解决网络问题才能恢复插件。请启动代理或切换网络后重试。"

如果用户有代理环境（如 Clash/v2ray），但代理端口未设置，可检查：
- 环境变量 `HTTPS_PROXY` / `HTTP_PROXY`（常见值 `http://127.0.0.1:7890`）

### 2. 检查当前插件状态

运行以下命令获取完整状态：

```
claude plugin list
claude plugin list --json
claude plugin marketplace list
```

关键判断：
- **`No plugins installed`** 且 `installed_plugins.json` 中 `plugins: {}` — 注册信息丢失，需要恢复
- **marketplace list 为空** — 市场配置丢失，需要重新添加
- **`enabledPlugins` 在 settings.json 中但实际未安装** — 清理脏数据

### 3. 读取并检查 installed_plugins.json

```
cat ~/.claude/plugins/installed_plugins.json
```

正常时应包含与插件名对应的版本和路径信息。如果 `plugins: {}` 说明注册信息已全部丢失。

**如果文件内容为空或仅剩 `{"version":2,"plugins":{}}`：**
- 检查 npm 或文件系统历史中是否有备份（参考第 6 步）
- 从 `settings.json` 的 `enabledPlugins`（如果还存在）推断之前安装过的插件
- 从文件修改时间确认是否是升级后首次出现此状态

### 4. 检查 settings.json 中的残留引用

```
cat ~/.claude/settings.json
```

注意以下字段：
- `enabledPlugins` — 如果包含不存在的插件引用，清理掉（设为 `{}`）
- `extraKnownMarketplaces` — 如果已包含正确的 marketplace 配置，可跳过重新添加步骤

### 5. 恢复市场配置

如果 marketplace list 为空或缺少 `claude-plugins-official`：

```
claude plugin marketplace add anthropics/claude-plugins-official
```

**重要：** 必须使用正确的 GitHub 仓库格式 `anthropics/claude-plugins-official`，不是 `claude-plugins-official`。

添加成功后验证：
```
claude plugin marketplace list
# 应显示 claude-plugins-official
```

### 6. 恢复已注册的插件

**从备份恢复（推荐）：** 如果存在备份文件，优先使用：
```
# 查找备份
ls ~/.claude/plugins/installed_plugins*.bak
ls ~/.claude/file-history/**/  # 可能在 file-history 中有旧版本
```

**从记忆恢复：** 如果用户记得之前装过哪些插件，直接按名称重新安装：
```
claude plugin install <plugin-name>@claude-plugins-official
```

**从 enabledPlugins 推断：** 如果 `settings.json` 的 `enabledPlugins` 还有残留（但插件已不存在），记录这些插件名并尝试重新安装：
```
# 收集 enabledPlugins 中的插件名（过滤掉不存在的）
claude plugin install <每个发现的插件名>@claude-plugins-official
```

### 7. 清理无效引用

恢复安装完成后，检查 `settings.json` 是否还有无效引用：
```
# 移除 enabledPlugins 中未安装的插件名
# 确保 enabledPlugins 只包含 claude plugin list 确认已安装的插件
```

## 已知常见问题

1. **`Repository not found`** — marketplace 名称不对，正确的 GitHub 仓库是 `anthropics/claude-plugins-official`
2. **`Plugin not found in marketplace`** — marketplace cache 可能过期，尝试 `claude plugin marketplace update` 之后再试
3. **`SSH authentication failed`** — 改用 HTTPS 方式，`claude plugin marketplace add` 会自动 fallback 到 HTTPS
4. **`Failed to connect to github.com port 443`** — 网络不通，参考步骤 1
5. **升级后 `installed_plugins.json` 为空但插件文件还在** — `marketplaces/` 和 `skills/` 目录可能仍有缓存，但注册信息确实需要重建

## 边界情况处理

- **多 marketplace 源**：如果用户配置了其他 marketplace（如 `claude-code-plugins`、`skills` 包等），也需要检查并恢复
- **项目级插件**：检查项目级 `.claude/settings.json` 中是否声明了 `enabledPlugins`，项目级和用户级的都需要恢复
- **插件 scope**：恢复时注意用户级（`--scope user`）和项目级（`--scope project`）的区别
- **不安装新插件**：本技能只恢复升级前已有的插件，不涉及新插件安装推荐
