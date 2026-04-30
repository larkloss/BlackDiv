# BlackDiv v1.1.0 USEC-Friendly Fork

## 项目概述
基于 [TacticalToaster/BlackDiv](https://github.com/TacticalToaster/BlackDiv) **v1.1.0** 的修改版本。
让 Black Division 阵营对 USEC（包含玩家和 NPC bot）保持友好，不主动开火、不警告。
其他阵营（BEAR、Scav、Rogues、Infected）的敌对关系保持原样。

## 上游版本说明（v1.1.0 重构）
v1.1.0 是上游 PR #5 的大重构：
- **Hunt 行为整体移到了 MoreBotsAPI v2.0+ 内部**（Plugin 端不再有 HuntManager / BotHuntManager / HuntTargetAction 等）
- 新增 `Server/Resources/config.jsonc` 提供运行时配置
- 新增 `BlackDivFaction.RevengeAfterRaids = false`
- MoreBotsAPI 依赖升级到 >= 2.0.0

## 用户信息
- SPT 安装路径：`C:\EFT\SPT`
- 用户角色：USEC
- 源码路径：`C:\Users\Larkl\OneDrive\文档\Claude\Projects\EFT\BlackDiv-modified`
- 当前分支：`claude/v1.1.0-usec-friendly`（worktree 在 `.claude/worktrees/v1.1.0-usec-friendly/`）

## 已完成的修改

### 1. 服务端阵营关系 (`Server/Mod.cs`)
- 删除 `factionService.AddEnemyByFaction(typeList, "usec")` — BD 不再视 USEC 为敌
- 删除 `factionService.AddEnemyByFaction("usec", "blackdiv")` — USEC 不再视 BD 为敌
- 新增 `factionService.AddFriendlyByFaction(typeList, "usec")` — BD 主动对 USEC 友好
- 新增 `factionService.AddFriendlyByFaction("usec", "blackdiv")` — USEC 主动对 BD 友好
- BEAR / savage / rogues / infected 的敌对保持原样

### 2. Bot AI 行为 (`Server/Resources/db/bots/sharedTypes/blackDiv.json`)
- 4 个难度（easy / normal / hard / impossible）的 `DEFAULT_USEC_BEHAVIOUR` 全部改 `"Neutral"`
  - 原值：easy/hard 为 `Warn`，normal 为 `Warn`，impossible 为 `AlwaysEnemies`
- `DEFAULT_BEAR_BEHAVIOUR` 保持 `"AlwaysEnemies"`
- 效果：BD 对 USEC 玩家完全无视，不警告、不攻击

### 3. 编译配置 (`Plugin/Plugin.csproj`、`Prepatch/Prepatch.csproj`)
- `Plugin.csproj` 所有 HintPath 从相对路径改为绝对路径 `C:\EFT\`，PostBuild 输出到 `..\output\`
- `Prepatch.csproj` 同上改为绝对路径，AssemblyName 从 `BlackDiv` 改为 `BlackDivPrepatch`
  （解决 NuGet "Ambiguous project name" 报错——和 Plugin 重名）
- 这些改动只影响 fork 本地编译，不改变功能

## 故意没做的事
- ❌ Hunt patch（不阻止 BD 对玩家触发 Hunt）
  - 原因：Hunt 只控制导航，不开火；既然 BD 已经对 USEC 友好，让它跑过来也不会打你
  - 用户不介意 BD 偶尔过来转一圈
- ❌ BD 主动跟随玩家
  - 原因：v1.0.1 时实现过，v1.1.0 因为 Hunt 移到 MoreBotsAPI 失效；用户有别的方案

## 编译与部署

### Server 端 (BlackDivServer.dll)
```bash
cd Server
dotnet build -c Release
# 产物: Server/bin/Release/BlackDiv/BlackDivServer.dll
```

### Client 端 (BlackDiv.dll + BlackDivPrepatch.dll)
```bash
cd Plugin
dotnet build Plugin.csproj -c Release
# 产物:
#   output/BlackDiv.dll
#   output/BlackDivPrepatch.dll
```

### 数据文件（不用编译）
- `Server/Resources/db/bots/sharedTypes/blackDiv.json`
- `Server/Resources/config.jsonc`

### 部署到 SPT 安装目录
| 源文件 | 目标 |
|---|---|
| `Server/bin/Release/BlackDiv/BlackDivServer.dll` | `C:\EFT\SPT\user\mods\BlackDivServer\BlackDivServer.dll` |
| `output/BlackDiv.dll` | `C:\EFT\BepInEx\plugins\BlackDiv\BlackDiv.dll` |
| `output/BlackDivPrepatch.dll` | `C:\EFT\BepInEx\patchers\BlackDivPrepatch.dll` |
| `Server/Resources/db/bots/sharedTypes/blackDiv.json` | `C:\EFT\SPT\user\mods\BlackDivServer\db\bots\sharedTypes\blackDiv.json` |
| `Server/Resources/config.jsonc` | `C:\EFT\SPT\user\mods\BlackDivServer\config.jsonc` |

⚠️ **首次升级注意**：
- 旧版 prepatch 文件名是 `BlackDiv.dll`（在 `BepInEx\patchers\`），新版改名为 `BlackDivPrepatch.dll`，**先删旧的**再放新的
- v1.1.0 必须有 `config.jsonc`，缺了会启动崩溃

## config.jsonc 可调字段

```jsonc
{
  "debug": { "logs": false },
  "spawns": {
    "chance": 25,           // Hunt 事件触发时 BD 实际刷新的概率（%）
    "minTime": 0.1,         // Hunt 最早触发时间（占整局 raid 时长比例）
    "maxTime": 0.8,         // Hunt 最晚触发时间（占整局 raid 时长比例）
    "labsGateChances": 20,  // Labs EXFIL 触发时 BD 出现概率（%）
    "labsStartChance": 15,  // Labs 开局 BD 常规刷新概率（%）
    "huntMaps": [...]       // 启用 Hunt 的地图列表
  }
}
```

| 想要的效果 | 改法 |
|---|---|
| 每局必出 BD | `chance: 100` |
| BD 很早出现 | `maxTime: 0.3` |
| Labs 必出 BD | `labsStartChance: 80, labsGateChances: 80` |
| 某地图不出 BD | 从 `huntMaps` 删除地图名 |

改 `config.jsonc` 不用重新编译，重启 SPT 即可。

## MoreBotsAPI FactionService 参考

| 方法 | 说明 |
|---|---|
| `AddEnemyByFaction(types, faction)` | 设为敌对（对应 JSON: AlwaysEnemies） |
| `AddWarnByFaction(types, faction)` | 设为警告（对应 JSON: Warn） |
| `AddFriendlyByFaction(types, faction)` | 设为友好（共享战斗感知，不互相攻击） |
| `AddRevengeByFaction(types, faction)` | 被攻击才还手 |

JSON `DEFAULT_*_BEHAVIOUR` 可用值：
- `"AlwaysEnemies"` — 始终敌对
- `"Warn"` — 靠近警告，不主动打
- `"Neutral"` — 完全无视
- ~~`"AlwaysFriendly"`~~ — **不存在**

## 历史分支

| 分支 | 说明 |
|---|---|
| `claude/elegant-goldberg` | 基于 v1.0.1，5 项改动包括 BD 持续跟随玩家功能。v1.1.0 后该分支的 Plugin 改动完全失效（文件被删除） |
| `claude/v1.1.0-usec-friendly` | **当前**。基于 v1.1.0，简化为只做 USEC 友好 |

## 同目录的另一个项目
`Tarkov-1.0-Backport-main` — WTT Content Backport 项目，之前修改了 `BackportJunkDisabler.cs` 的 `_itemsToBlacklist`，去掉 9 个 Black Division 物品（胸挂/板甲 4 个、夜视 1 个、armband 4 个），只保留 3 个非 BD 条目（LP Gamma, Battle Worn Gamma, LP Fanny），让 BD 物品不被加入奖励黑名单。
