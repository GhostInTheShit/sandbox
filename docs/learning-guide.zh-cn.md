# s&box Sandbox 项目学习指南（新手向）

> 本文面向第一次接触 s&box / 本项目的开发者，目标是让你在读完之后：
> 1. 知道这个项目是干什么的、怎么跑起来；
> 2. 看懂它的代码组织方式和核心架构；
> 3. 能找到"某个功能在哪实现"，并知道按什么顺序深入。
>
> 阅读约定：文中所有文件路径都相对仓库根目录，例如 `Code/Player/Player.cs`。

---

## 目录

1. [这个项目是什么](#1-这个项目是什么)
2. [准备工作：如何运行](#2-准备工作如何运行)
3. [项目结构地图](#3-项目结构地图)
4. [s&box 核心概念（重点）](#4-sbox-核心概念重点)
5. [一次游戏的启动流程](#5-一次游戏的启动流程)
6. [核心子系统详解](#6-核心子系统详解)
7. [代码组织与编码约定](#7-代码组织与编码约定)
8. [新手学习路径](#8-新手学习路径)
9. [动手实践：常见任务怎么改](#9-动手实践常见任务怎么改)
10. [术语表](#10-术语表)
11. [进阶资源](#11-进阶资源)

---

## 1. 这个项目是什么

这是 **Facepunch 官方的 s&box 沙盒游戏**（[GitHub: Facepunch/sandbox](https://github.com/Facepunch/sandbox)），也就是 Garry's Mod 的精神续作。它不是一个"游戏引擎"，而是一个用 s&box 引擎制作的**完整游戏玩法模组（gamemode）**：

- 玩家可以在地图里生成各种道具（prop）、实体、NPC、武器；
- 用 **Toolgun（工具枪）** 创建约束、轮子、推进器、气球、按钮、灯光、摄像头等；
- 用 **Physgun（物理枪）** 抓取、旋转、冻结物体；
- 可以把一堆东西拼成"机械"（contraption），坐进椅子驾驶；
- 支持存档/读档、撤销、服务器限制、权限、封禁、NPC AI、载具、HUD、菜单等。

项目采用 **MIT 许可证**，官方明确说"欢迎学习并用到你自己的 s&box 游戏中"。所以它是学习 s&box 开发最好的范本之一。

### 工程配置速览（[`sandbox.sbproj`](../sandbox.sbproj)）

| 配置 | 值 | 含义 |
|---|---|---|
| `Type` | `game` | 这是一个游戏（而非 addon） |
| `Org` / `Ident` | `facepunch` / `sandbox` | 包标识 |
| `MaxPlayers` | 64 | 最大玩家数 |
| `TickRate` | 50 | 服务器逻辑帧率 |
| `StartupScene` | `scenes/sandbox.scene` | 启动场景 |
| `SystemScene` | `scenes/system.scene` | 系统场景（HUD、菜单等） |
| `MapList` | `facepunch.flatgrass` | 默认地图 |
| `Resources` | 一大串路径 | 发布时要打包的资源白名单 |

---

## 2. 准备工作：如何运行

### 2.1 前置要求

- 安装 **s&box 编辑器**（Steam 上的 s&box，或 Facepunch 提供的版本）；
- 会一点 **C#**（本项目 100% C# + Razor 界面）；
- 知道一点 **Unity/虚幻式的组件化思维**（GameObject + Component）即可，不需要会 s&box 引擎本身。

### 2.2 打开项目

直接双击 `sandbox.sbproj`（或在 s&box 编辑器里打开该目录）。打开后：

- 编辑器会自动生成 `.sbox/agent-guide.md`，里面有本项目安装的引擎路径、CLI、API 参考路径，**先看它**；
- 按 **F5 / 点击 Play** 即可运行游戏。

> 如果 `.sbox/agent-guide.md` 不存在：在 Windows 上检查环境变量 `FACEPUNCH_ENGINE`（或用户环境变量），找到引擎安装目录后运行 `bin/managed/sbox-cli.exe --help`，它会告诉你指南文件在哪。重新打开 `.sbproj` 可以重新生成本地指南。

### 2.3 建议的入门体验

先别急着读代码，进游戏玩 10 分钟（按键以游戏内"输入设置"为准，这里只写常见默认）：

1. 按 `Q` 打开生成菜单（Spawn Menu），生成一个道具、一个 NPC、一把枪；
2. 按 `1/2/3` 切换武器，掏出工具枪（Toolgun），用滚轮或工具菜单切换工具；
3. 用物理枪（Physgun）右键冻结一个道具，再左键抓起来；
4. 撤销（Undo）与清理（Cleanup）在生成菜单的 Utilities 页里；
5. 记分板、聊天、检查菜单等快捷键都能在输入设置里查到。

有了这些感性认识，再读代码会事半功倍。

---

## 3. 项目结构地图

```
sandbox/
├── Code/                  # 游戏逻辑（C#）
│   ├── Assembly.cs        # 全局 using
│   ├── GameLoop/          # 游戏管理、设置、权限、限制
│   ├── Player/            # 玩家、库存、伤害、撤销
│   ├── Weapons/           # 武器（枪械 / 近战 / 工具枪 / 物理枪）
│   ├── Game/              # 实体、控制、武器基类、后处理、音效
│   ├── Spawner/           # 生成管线（道具/实体/复制/建造/挂载）
│   ├── Builder/           # 建造工具的几何后端
│   ├── Npcs/              # NPC AI（含 README.md）
│   ├── Vehicles/          # 载具（四轮车）
│   ├── UI/                # HUD、菜单、聊天、记分板（Razor）
│   ├── Save/              # 存档系统
│   ├── Components/        # 通用组件（Ownable、ManualLink 等）
│   ├── Items/             # 拾取物（血包、弹药）
│   ├── Map/               # 地图实体（移动平台、触发器）
│   ├── Utility/           # 工具函数（HUD、字体、本地化文本等）
│   ├── FreeCam/           # 自由相机
│   └── Cleanup/           # 清理系统
├── Editor/                # 编辑器扩展（武器图标、预览等）
├── Assets/                # 美术与场景资源
│   ├── scenes/            # 场景文件（sandbox / system / tools / zoo …）
│   ├── prefabs/           # 引擎 prefab（player、spawnpoint）
│   ├── entities/          # 各实体 prefab
│   ├── weapons/           # 武器 prefab 与模型
│   ├── vehicles/buggy/    #  buggy 载具
│   ├── spawnlists/        # 生成清单（.spawn）
│   ├── models/ materials/ textures/ sounds/ fonts/ shaders/
├── Localization/          # 30 种语言的本地化 JSON
├── ProjectSettings/       # 项目设置
├── sandbox.sbproj         # 工程文件
└── docs/                  # 本文档
```

### 3.1 场景（Scenes）的作用

| 场景 | 作用 |
|---|---|
| [`Assets/scenes/sandbox.scene`](../Assets/scenes/sandbox.scene) | 主场景：只有一个 `MapInstance`，负责加载地图 `facepunch.flatgrass` |
| [`Assets/scenes/system.scene`](../Assets/scenes/system.scene) | 系统场景：`ScreenPanel` + `HudLayer`、`Inventory`、`SpawnMenuHost`、`Scoreboard`、`Notices` 等 UI 宿主 |
| `Assets/scenes/tools.scene` | 工具枪相关的场景内容 |
| `Assets/scenes/zoo.scene` | 展示/测试场景 |
| `Assets/scenes/npc_testing.scene` | NPC 测试场景 |

> 记住一个关键点：**`sandbox.scene` 几乎不装逻辑，逻辑全在 `Code/` 里**。场景主要提供地图和 UI 宿主。

### 3.2 资源文件类型速查

| 扩展名 | 含义 |
|---|---|
| `.prefab` | 预制体：可复用的 GameObject 模板（玩家、武器、实体） |
| `.scene` | 场景 |
| `.sent` | **Sandbox Entity**：脚本化实体的资源定义（见 [`ScriptedEntity.cs`](../Code/Game/Entity/ScriptedEntity.cs)） |
| `.spawn` | 生成清单（Spawnlist）：生成菜单里的一组内容 |
| `.vmdl` / `.vmat` | 模型 / 材质 |
| `.ag` | 动画图（Animation Graph） |
| `.sound` / `.sndef` | 声音事件 / 声音定义 |
| `.razor` + `.razor.cs` + `.scss` | UI 组件 + 后台代码 + 样式 |
| `.json`（Localization） | 本地化文本，键值对，`#key` 引用 |

---

## 4. s&box 核心概念（重点）

**这一节是全文最重要的部分。** 读懂这些概念，后面所有代码都能看懂。

### 4.1 GameObject 与 Component：一切皆组件

s&box 的世界由 `GameObject` 组成，`GameObject` 上挂 `Component`。你写的绝大多数代码都是 **Component 的子类**，通过重写生命周期方法工作：

```csharp
public sealed class MyThing : Component
{
    protected override void OnStart() { }      // 首次启用时
    protected override void OnUpdate() { }     // 每帧
    protected override void OnFixedUpdate() { } // 每个物理帧
    protected override void OnDestroy() { }    // 销毁时
}
```

- `Player`、`Prop`、武器、工具、NPC、载具，全都是 Component；
- 一个 GameObject 上可以挂多个组件，比如玩家身上有 `Player`、`PlayerController`、`PlayerInventory`、`Rigidbody`、各种 `MoveMode`；
- 组件之间用 `GetComponent<T>()` / `Components.Get<T>()` 互相查找。

> 参见 [`Code/Player/Player.cs`](../Code/Player/Player.cs)：`Player` 是一个 `Component`，同时实现了 `IDamageable`、`IPlayerEvents` 等接口。

### 4.2 GameObjectSystem：场景级"单例系统"

有些逻辑不属于某个 GameObject，而属于整个场景（比如"管理所有玩家"、"管理所有座椅"）。这类逻辑写成 `GameObjectSystem<T>`：

```csharp
public sealed partial class GameManager : GameObjectSystem<GameManager>
{
    public GameManager( Scene scene ) : base( scene ) { }
}
```

- 每个场景每种系统只有一个实例，用 `Scene.Get<T>()` 或 `Scene.GetSystem<T>()` 获取；
- 可以在构造函数里 `Listen( Stage.StartUpdate, ... )` 订阅更新阶段；
- 本项目里的系统：`GameManager`、`ControlSystem`、`LimitsSystem`、`UndoSystem`、`SaveSystem`、`CleanupSystem`、`NpcSystem`、`CameraViewSystem` 等。

> 参见 [`Code/GameLoop/GameManager.cs`](../Code/GameLoop/GameManager.cs)、[`Code/Game/ControlSystem/ControlSystem.cs`](../Code/Game/ControlSystem/ControlSystem.cs)。

### 4.3 特性（Attributes）：代码与编辑器/网络的桥梁

s&box 大量使用特性把 C# 成员暴露给编辑器、网络、控制台。**看到这些特性就要条件反射地理解它的作用**：

| 特性 | 作用 | 例子 |
|---|---|---|
| `[Property]` | 成员显示在编辑器检视面板，且会被序列化（存进场景/存档） | `Player.Health` |
| `[Sync]` | 属性在网络间同步（主机 → 客户端） | `Player.Health`、`Physgun._state` |
| `[Sync( SyncFlags.FromHost )]` | 明确只从主机同步 | `Npc.Health` |
| `[Rpc.Host]` | 客户端调用、主机执行 | `GameManager.RequestRespawn` |
| `[Rpc.Owner]` | 调用对象的拥有者执行 | `ToolMode.CheckContraptionStats` |
| `[Rpc.Broadcast]` | 广播给所有人（通常配合 `NetFlags.HostOnly`） | `GameManager.NotifyRpc` |
| `[ConCmd]` | 注册控制台命令 | `spawn`、`sandbox.toolgun.random` |
| `[ConVar]` | 注册控制台变量（设置项） | `sandbox.spawnprotection` |
| `[Title]` / `[Category]` / `[Icon]` | 编辑器里的显示名/分类/图标 | `[Title( "Four Wheel Vehicle" )]` |
| `[Group]` / `[Range]` / `[Order]` | 检视面板分组、范围、排序 | 工具设置 |
| `[RequireComponent]` | 自动添加依赖组件 | `Player.Controller` |
| `[ClientEditable]` | 客户端可编辑（如 physgun 颜色） | `Physgun.CanAim` |
| `[SignalInput]` | 线束（Wire）系统的输入端口 | `Physgun.ExtendSignal` |
| `[Hide]` | 在检视面板隐藏 | `Npc` 类本身 |

> 参见 [`Code/Player/Player.cs:15-27`](../Code/Player/Player.cs#L15-L27) 看 `[Property, Sync]` 的典型用法。

### 4.4 网络模型：主机权威（Host Authoritative）

s&box 是**主机权威**的多人架构：

- **主机（Host）**：跑真正的游戏逻辑，是唯一"说了算"的一方；
- **客户端（Client）**：渲染 + 预测，通过 RPC 请求主机做事；
- `Networking.IsHost`：当前是不是主机；
- `IsProxy`：当前对象是不是"别人的"（网络代理），本地玩家自己的对象 `IsProxy == false`；
- `NetworkSpawn()`：把对象生成到网络上，让所有人可见；
- `Connection`：一个玩家连接；
- `OwnerTransfer`：对象所有权转移策略。

**写代码时的直觉**：

- 会改变世界的操作（扣血、生成、删除）→ 只在主机执行，用 `if ( !Networking.IsHost ) return;` 守住；
- 纯表现（音效、粒子、屏幕特效）→ 可以广播给所有人；
- 需要同步的状态 → 加 `[Sync]`。

> 例子：[`Player.OnDamage`](../Code/Player/Player.cs#L543) 第一行就是 `if ( !Networking.IsHost ) return;`；[`GameManager.Spawn`](../Code/GameLoop/GameManager.Spawn.cs#L14) 客户端只播音效，真正生成在主机。

### 4.5 事件系统：解耦的"广播站"

项目大量使用接口事件来解耦。命名规律：

- `Global.I...Events`：全局事件，用 `Scene.RunEvent<T>( x => x.OnSomething() )` 触发；
- `Local.I...Events`：只发给本地玩家相关对象，用 `Local.I...Events.PostToGameObject( go, ... )`；
- 常见事件接口：`IPlayerEvents`、`IDamageEvents`、`IDeathEvents`、`ISaveEvents`、`ISpawnEvents`、`IToolActionEvents`、`IPhysgunEvent`、`ICarryEvent`、`IHudEvents`、`IConnectionEvents`、`IAdminEvents`。

> 例子：玩家死亡时 [`Player.Kill`](../Code/Player/Player.cs#L317) 调用 `NotifyDeath` 广播事件，UI、成就、统计各自监听，互不干扰。

### 4.6 存档 = JSON 差异补丁（diff/patch）

[`SaveSystem`](../Code/Save/SaveSystem.cs) 的设计很巧妙：它不保存整个场景，而是：

1. 记录场景**基线**（从场景文件加载时的状态）；
2. 保存时把当前场景序列化成 JSON，与基线做 **diff**，只存差异（patch）；
3. 读档时把 patch 应用到基线，重建场景，再切换场景。

额外保存：`[Sync]` 属性（`SyncState`）、网络所有权（`NetworkOwnership`）、需要的云资源包（`RequiredPackages`）、元数据（`Metadata`）。格式版本号 `CurrentSaveVersion = 2`。

### 4.7 UI：Razor 组件 + HUD 绘制

UI 分两套：

1. **Razor 组件**（`.razor`）：类似 React/Blazor 的声明式 UI，用于菜单、聊天、计分板等。例如 [`SpawnMenu.razor`](../Code/UI/SpawnMenu/SpawnMenu.razor)、[`Scoreboard.razor`](../Code/UI/Scoreboard.razor)、[`ChatBox.cs`](../Code/UI/Chat/ChatBox.cs)；
2. **HUD 绘制**：用 `Painter` 直接画，组织方式是 `HudLayer`（层）→ `HudCanvas`（画布）→ `HudSection`（分区，如血量、弹药、热键栏）。`HudElement` 枚举控制显隐。

> 参见 [`Code/UI/HudLayer.razor`](../Code/UI/HudLayer.razor)、[`Code/UI/HudSection.cs`](../Code/UI/HudSection.cs)、[`Code/Utility/HUD.cs`](../Code/Utility/HUD.cs)。

### 4.8 本地化

所有面向玩家的文本都在 `Localization/<语言>/` 下的 JSON 里，键是 `#xxx.yyy`。代码里用 `Game.Language.GetPhrase( "#key" )` 或 `[Title( "#key" )]` 引用。新增文本时记得加英文键。

---

## 5. 一次游戏的启动流程

把这一节在脑子里放一遍，你就有了整个项目的"主干"：

```
引擎启动
  └─ 加载 scenes/sandbox.scene（MapInstance 加载 facepunch.flatgrass）
  └─ 加载 scenes/system.scene（HUD、菜单、Inventory、Scoreboard…）
        │
        ▼
GameManager.OnHostInitialize()  ── 创建大厅（Public，32 人，"Sandbox"）
        │
        ▼ 有玩家连接
GameManager.INetworkListener.OnActive( connection )
  ├─ CreatePlayerInfo( connection )   → 生成 PlayerData（网络对象，归属该玩家）
  ├─ SpawnPlayer( playerData )        → 找 SpawnPoint → 触发 OnPlayerRespawning
  │     └─ 克隆 /prefabs/engine/player.prefab → NetworkSpawn 给该连接
  └─ Player.OnStart()                 → 本地玩家设置 LocalPlayer
        │
        ▼ 游戏中
玩家输入 → Player.OnControl() → 武器/工具/驾驶
        │
        ▼ 受到伤害
Player.OnDamage()（仅主机）→ 扣血 → 死亡时：
  ├─ GameManager.OnDeath()  → 击杀信息、统计、事件
  └─ Player.Kill()          → 布娃娃 + 观察者 + 销毁
        │
        ▼ 想复活
客户端 RequestRespawn RPC → 主机 SpawnPlayer
```

关键文件：

- [`Code/GameLoop/GameManager.cs`](../Code/GameLoop/GameManager.cs)：`OnHostInitialize`、`OnActive`、`SpawnPlayer`、`OnDeath`；
- [`Code/GameLoop/GameManager.Spawn.cs`](../Code/GameLoop/GameManager.Spawn.cs)：`Spawn`（生成道具/实体）；
- [`Assets/prefabs/engine/player.prefab`](../Assets/prefabs/engine/player.prefab)：玩家预制体（`Player` + `PlayerController` + `PlayerInventory` + `Rigidbody` + 各种 `MoveMode` + 身体模型）。

---

## 6. 核心子系统详解

### 6.1 GameManager —— 游戏总管

**职责**：大厅、玩家生成、死亡处理、RPC 工具方法、清理通知。

**关键文件**：[`Code/GameLoop/GameManager.cs`](../Code/GameLoop/GameManager.cs) 及其 partial：

| 文件 | 内容 |
|---|---|
| `GameManager.cs` | 大厅、生成玩家、死亡、属性修改 RPC、清理 |
| `GameManager.Spawn.cs` | `Spawn` RPC：解析 ident → 创建 spawner → 生成 → 记撤销 |
| `GameManager.Gravity.cs` | 全局重力 |
| `GameManager.Water.cs` | 水效果 |
| `GameManager.MenuBar.cs` | 菜单栏 |
| `GameManager.ContextMenu.cs` | 右键菜单 |
| `GameManager.Achievements.cs` | 成就 |
| `GameManager.PropImpactShake.cs` | 道具撞击震屏 |
| `GamePreferences*.cs` | 玩家偏好设置（HUD、Physgun） |
| `ServerSettings*.cs` | 服务器设置（ConVar + 自动设置界面） |
| `LimitsSystem.cs` | 每玩家生成上限 |
| `Permissions.cs` / `BanSystem/` | 权限与封禁 |

**要点**：`GameManager` 继承 `GameObjectSystem<GameManager>`，所以它是场景单例；它同时实现了一堆事件接口（`INetworkListener`、`ISceneStartup`、`ISaveEvents`…），这是本项目的常见写法。

### 6.2 Player —— 玩家

**职责**：生命值/护甲、伤害与死亡、布娃娃、noclip、驾驶、携带、观察者。

**关键文件**：[`Code/Player/Player.cs`](../Code/Player/Player.cs) 及 partial：

| 文件 | 内容 |
|---|---|
| `Player.cs` | 生命/护甲、伤害、死亡、布娃娃、noclip、控制入口 |
| `Player.Camera.cs` | 第一/第三人称相机 |
| `Player.Physgun.cs` | 物理枪相关玩家逻辑 |
| `Player.Drive.cs` | 道具驾驶 |
| `Player.Inventory.cs` / `.Ammo.cs` / `.Seat.cs` | 武器库存 |
| `Player.Undo.cs` | 撤销钩子 |
| `Player.Carry.cs` | 搬东西 |
| `Player.Flashlight.cs` | 手电 |
| `Player.Gib.cs` | 碎块 |
| `Player.Observer.cs` | 死亡观察 |
| `Player.Stats.cs` / `PlayerData.cs` | 统计与持久数据 |
| `Player.Loadout.cs` | 默认武器 |
| `Player.FallDamage.cs` | 摔落伤害 |
| `SandboxVoice.cs` | 语音 |

**要点**：

- `Health`、`Armour` 都带 `[Sync]`，所以所有客户端都能看到血量；
- `OnDamage` 只在主机执行，先跑 `IDamageEvents` 让监听者修改/取消伤害，再结算护甲、爆头加成，最后广播 `NotifyOnDamage`；
- 死亡流程：`OnDeath` → `Kill` → 创建布娃娃（`CreateRagdoll`）→ 创建观察者（`Ghost`）→ 销毁玩家对象。

### 6.3 武器体系

**继承链**（自下而上）：

```
引擎 BaseInventoryItem
  └─ 引擎 BaseCombatWeapon
       └─ BaseSandboxWeapon            ← Code/Game/Weapon/BaseSandboxWeapon/
            ├─ BaseBulletWeapon        ← 子弹武器基类
            │    ├─ GlockWeapon / Colt1911Weapon / RevolverWeapon
            │    ├─ Mp5Weapon / M4a1Weapon / ShotgunWeapon
            │    ├─ SniperWeapon / RpgWeapon
            │    └─ HandGrenadeWeapon
            ├─ MeleeWeapon             ← 近战（Crowbar、Fists）
            ├─ IronSightsWeapon        ← 机瞄
            └─ ScreenWeapon            ← 带屏幕的武器
                 ├─ Toolgun
                 ├─ CameraWeapon
                 └─ ScreenWeapon（摄像头/屏幕类）
```

**关键文件**：

- [`Code/Game/Weapon/BaseSandboxWeapon/BaseSandboxWeapon.cs`](../Code/Game/Weapon/BaseSandboxWeapon/BaseSandboxWeapon.cs)：沙盒武器基类，处理掉落（`DropIntoWorld`）、座位控制（`IPlayerControllable`）、NPC 瞄准等；
- `BaseSandboxWeapon.Kick.cs` / `.Reloading.cs` / `.ViewModel.cs` / `.WorldModel.cs`：后坐力、换弹、视图模型、世界模型；
- [`Code/Game/Weapon/BaseBulletWeapon/BaseBulletWeapon.cs`](../Code/Game/Weapon/BaseBulletWeapon/BaseBulletWeapon.cs)：子弹、弹道、弹药；
- [`Code/Game/Weapon/MeleeWeapon/MeleeWeapon.cs`](../Code/Game/Weapon/MeleeWeapon/MeleeWeapon.cs)：近战挥砍；
- [`Code/Game/Weapon/ViewModel/ViewModel.cs`](../Code/Game/Weapon/ViewModel/ViewModel.cs)：第一人称手臂/武器模型；
- [`Code/Player/PlayerInventory.cs`](../Code/Player/PlayerInventory.cs)：武器栏（切换、拾取、弹药合并、工具模式）。

**要点**：

- 武器是 GameObject，被 `PlayerInventory` 管理；拾取 = 克隆 prefab + `NetworkSpawn` + 挂到玩家身上；
- 掉落时 `DropIntoWorld` 把活对象丢到世界里，保留弹药和工具状态；
- 武器设置用 `[Property, ClientEditable]`，玩家可在游戏里改（如射速、伤害）。

### 6.4 Toolgun + ToolMode —— 工具枪（本项目灵魂）

**职责**：工具枪本身是一个 `ScreenWeapon`，上面挂着 **30+ 个工具模式（ToolMode）组件**，同一时刻只有一个激活。

**关键文件**：

- [`Code/Weapons/ToolGun/Toolgun.cs`](../Code/Weapons/ToolGun/Toolgun.cs)：武器本体；`CreateToolComponents()` 自动创建所有工具模式；`SetToolMode()` 切换；
- [`Code/Weapons/ToolGun/ToolMode.cs`](../Code/Weapons/ToolGun/ToolMode.cs)：**所有工具的基类**，理解它就理解了工具系统；
- `ToolMode.*.cs`：`Configuration`（设置）、`Cookies`（记忆用户设置）、`Effects`、`Helpers`、`SnapGrid`（吸附网格）；
- [`Code/Weapons/ToolGun/SnapGrid.*.cs`](../Code/Weapons/ToolGun/SnapGrid.cs)：网格吸附（平面/ mesh / 圆形 / 连接）；
- [`Code/Weapons/ToolGun/Toolgun.Effects.cs`](../Code/Weapons/ToolGun/Toolgun.Effects.cs)、`.Screen.cs`：特效与屏幕；
- [`Code/Weapons/ToolGun/Modes/`](../Code/Weapons/ToolGun/Modes/)：每个工具一个文件夹。

**ToolMode 基类提供的能力**：

| 成员 | 作用 |
|---|---|
| `OnControl()` | 每帧输入，工具的主逻辑 |
| `DrawHud( painter, crosshair )` | 画准星/预览 |
| `RegisterAction( input, name, callback )` | 注册动作（主/副/换弹等），自动派发 |
| `FireToolAction` / `FirePostToolAction` | 触发 `IToolActionEvents`，让插件/限制系统拦截 |
| `TraceIgnoreTags` / `TraceHitboxes` | 控制射线检测忽略什么 |
| `AbsorbMouseInput` | 吸收鼠标（用于旋转物体等） |
| `LoadCookies` / `SaveCookies` | 记忆工具设置 |
| `SnapGrid` | 网格吸附 |

**工具清单**（按生成菜单分组）：

- **Constraints（约束）**：Weld 焊接、Rope 绳、Elastic 弹性、Ball Socket 球窝、Slider 滑轨、No Collide 无碰撞、Keep Upright 保持直立、Hydraulic 液压、Muscle 肌肉、Winch 绞盘、Motor 马达、Wire 线束；
- **Tools（工具）**：Remover 删除、Colour 颜色、Material 材质、Resizer 缩放、Mass 质量、Gravity 重力、Unbreakable 无敌、Stacker 堆叠、Duplicator 复制、Trail 拖尾、Text 文字、Decal 贴花、Light 灯、Camera 摄像头、Speaker 扬声器、Emitter 粒子 emitter、Balloon 气球、Thruster 推进器、Button 按钮、Wheel 轮子、Hoverball 悬浮球；
- **Building（建造）**：Builder（任意形状建造块）；
- **Render（渲染）**：Camera 等。

**要点**：工具是"数据驱动 + 反射"的——`CreateToolComponents` 用 `Game.TypeLibrary.GetTypes<ToolMode>()` 自动发现所有工具类，所以**新增工具只需写一个 ToolMode 子类**，不需要改 Toolgun。

### 6.5 Physgun —— 物理枪

**职责**：抓取、移动、旋转、冻结物体，以及"重力抓取"（把物体吸过来再发射出去）。

**关键文件**：[`Code/Weapons/PhysGun/Physgun.cs`](../Code/Weapons/PhysGun/Physgun.cs) 及 partial：

| 文件 | 内容 |
|---|---|
| `Physgun.cs` | 抓取状态机、输入、冻结/解冻、发射 |
| `Physgun.Pull.cs` | 重力拉扯 |
| `Physgun.Hud.cs` | 光束与准星 |
| `Physgun.Effects.cs` | 特效 |
| `PhygunViewmodel.cs` / `PhygunWorldmodel.cs` | 视图/世界模型 |

**核心机制**：

- `GrabState`（`record struct`）记录抓的是什么、局部偏移、距离、旋转；`_state` 带 `[Sync]`，所以所有人都能看到别人抓着东西；
- 真正移动物体在 `OnFixedUpdate` 里的 `PhysicsHold.Hold/Move`（[`Code/Game/Physics/PhysicsHold.cs`](../Code/Game/Physics/PhysicsHold.cs)）；
- 冻结 = `body.MotionEnabled = false`；解冻时用 `PhysicsHold.GetConnectedBodies` 找出整个机械的所有刚体一起解冻；
- 抓取前会触发 `IPhysgunEvent.OnPhysgunGrab`，让物体拒绝被捡（比如别人的东西）；
- 颜色、光束等是 `[ClientEditable]`，每个玩家自己设置。

### 6.6 生成管线（Spawner）

**职责**：把"生成菜单里的一个条目"变成世界里的 GameObject。这是理解"东西是怎么被创造出来的"的关键。

**接口**：[`Code/Spawner/ISpawner.cs`](../Code/Spawner/ISpawner.cs)，定义了 `DisplayName`、`Bounds`、`Loading`、`DrawPreview`、`Spawn` 等。

**工厂**：[`Code/Spawner/ISpawner.Extensions.cs`](../Code/Spawner/ISpawner.Extensions.cs) 的 `ISpawner.Create(type, path, source, metadata)`：

| type | 实现 | 用途 |
|---|---|---|
| `prop` | `PropSpawner` | 云/本地模型 → `Prop` 组件 |
| `entity` / `sent` | `EntitySpawner` | `.sent` 实体 |
| `dupe` | `DuplicatorSpawner` | 复制粘贴的机械 |
| `builder` | `BuilderSpawner` | 建造块 |
| `mount` | `MountSpawner` | 挂载包（Mounts） |

**流程**（以道具为例）：

```
玩家在生成菜单点击
  → GameManager.Spawn( ident )  [Rpc.Broadcast]
      → 客户端：播生成音效 + 统计
      → 主机：解析 ident → ISpawner.Create → 等待 Loading
          → 触发 ISpawnEvents.OnSpawn（可取消）
          → spawner.Spawn( transform, player )
              → PropSpawner：new GameObject + Prop + Ownable + NetworkSpawn
          → 记入 UndoSystem
          → 触发 ISpawnEvents.OnPostSpawn
```

**要点**：

- `PropSpawner` 会给道具加 `Ownable`（归属），这是"道具保护"的基础；
- 没有物理数据的模型会加 `BoxCollider + Rigidbody`；
- `CloudLoadTracker` 负责显示加载进度；
- 生成受 `LimitsSystem` 限制（每人类别上限）。

### 6.7 存档系统（SaveSystem）

**关键文件**：[`Code/Save/SaveSystem.cs`](../Code/Save/SaveSystem.cs)。

**要点**：

- 只在主机运行；
- 保存的是**差异**：`Json.CalculateDifferences( baseline, current )`；
- 保存内容：`Version`、`SceneId`、`SceneSources`（场景来源）、`SceneProperties`、`Metadata`、`Patch`、`NetworkOwnership`、`SyncState`、`RequiredPackages`；
- 读档：挂载所需云资源 → patch 应用到基线 → 构造 `SceneFile` → `Game.ChangeScene` → 恢复 Sync 状态和所有权 → 触发 `ISaveEvents.AfterLoad`；
- 存档版本不匹配会拒绝加载（当前版本 2）。

### 6.8 NPC 系统

**关键文件**：[`Code/Npcs/`](../Code/Npcs/)，**先读 [`Code/Npcs/README.md`](../Code/Npcs/README.md)**（官方写的，质量很高）。

**核心设计**：NPC 每 tick"思考"一次：

1. **Senses（感知）** 收集信息（看到/听到什么）到 `Awareness`；
2. **`GetSchedule()`** 选择当前最想做的行为（优先级最高的 schedule）；
3. **Schedule（日程）** 是一串 **Task（任务）**（移动、看、开火、等待）；
4. 高优先级 schedule 会自动打断低优先级的。

**分层（Layers）**：`SensesLayer`、`NavigationLayer`、`AnimationLayer`、`SpeechLayer` 自动添加。

**具体 NPC**：`ScientistNpc`（科学家）、`CombatNpc`（战斗 NPC）、`RollerNpc`（滚球）。

**自定义 NPC 只需**：继承 `Npc`，重写 `Faction`、`SetupRelationships()`、`GetSchedule()`，然后给 prefab 加 `NavMeshAgent`、`SkinnedModelRenderer`、`Collider`、`Rigidbody`。

### 6.9 载具（FourWheelVehicle）

**关键文件**：[`Code/Vehicles/FourWheelVehicle.cs`](../Code/Vehicles/FourWheelVehicle.cs) 及 partial：

| 文件 | 内容 |
|---|---|
| `FourWheelVehicle.cs` | 座位、引擎开关、同步状态 |
| `FourWheelVehicle.Driver.cs` | 驾驶员输入 → 油门/刹车/转向/加速/手刹 |
| `FourWheelVehicle.Drivetrain.cs` | 引擎、变速箱、刹车、动力滑移 |
| `FourWheelVehicle.Wheels.cs` | 悬挂、防倾杆、抓地力 |
| `FourWheelVehicle.Rig.cs` | 模型姿态（轮子、悬挂、方向盘） |
| `FourWheelVehicle.IK.cs` | 驾驶员手脚 IK |
| `FourWheelVehicle.Tuning.cs` | 调校参数 |
| `VehicleCamera.cs` / `VehicleLights.cs` / `VehicleSounds.cs` | 相机 / 车灯 / 声音 |

**要点**：轮子不是刚体，而是"从车身向下打射线，在接触点施加悬挂力"。载具通过 `BaseChair`（座位）+ [`ControlSystem`](../Code/Game/ControlSystem/ControlSystem.cs) 被驾驶：ControlSystem 每帧找到座位上的玩家，调用该机械上所有 `IPlayerControllable.OnControl()`。

### 6.10 UI 与 HUD

**关键文件**：

- [`Code/UI/SpawnMenuHost.razor`](../Code/UI/SpawnMenuHost.razor)：生成菜单宿主；
- [`Code/UI/SpawnMenu/`](../Code/UI/SpawnMenu/)：生成菜单各标签页（Props / Entities / NPCs / Weapons / Dupes / Mounts / Spawnlists / Saves / HUD / Tools / Utilities）；
- [`Code/UI/Inventory/Inventory.razor`](../Code/UI/Inventory/Inventory.razor)：武器热键栏；
- [`Code/UI/Vitals/Vitals.razor`](../Code/UI/Vitals/Vitals.razor)：血量/护甲/弹药；
- [`Code/UI/Scoreboard.razor`](../Code/UI/Scoreboard.razor)：计分板；
- [`Code/UI/Chat/`](../Code/UI/Chat/)：聊天框；
- [`Code/UI/Notices/`](../Code/UI/Notices/)：通知；
- [`Code/UI/ContextMenu/`](../Code/UI/ContextMenu/)：右键检查菜单（Inspector）；
- [`Code/UI/Widgets/`](../Code/UI/Widgets/)：通用控件（按钮、颜色、下拉、滑条…）；
- [`Code/UI/Theme.scss`](../Code/UI/Theme.scss)：主题。

**要点**：Razor 文件 = HTML 风格模板 + `@code { }` 后台；`.scss` 是样式；HUD 用 `HudSection.Paint( Painter )` 绘制。

### 6.11 其他值得知道的子系统

| 系统 | 位置 | 作用 |
|---|---|---|
| 控制/座椅 | [`Code/Game/ControlSystem/`](../Code/Game/ControlSystem/) | 每帧把座位上的玩家输入喂给机械 |
| 限制 | [`Code/GameLoop/LimitsSystem.cs`](../Code/GameLoop/LimitsSystem.cs) | 每玩家各类对象上限 |
| 撤销 | [`Code/Player/UndoSystem/UndoSystem.cs`](../Code/Player/UndoSystem/UndoSystem.cs) | 按玩家栈记录操作，UI 可查 |
| 清理 | [`Code/Cleanup/CleanupSystem.cs`](../Code/Cleanup/CleanupSystem.cs) | 清理世界、恢复基线 |
| 权限 | [`Code/GameLoop/Permissions.cs`](../Code/GameLoop/Permissions.cs) | 玩家权限 |
| 封禁 | [`Code/Game/BanSystem/BanSystem.cs`](../Code/Game/BanSystem/BanSystem.cs) | 封禁列表 |
| 摄像头 | [`Code/Game/Entity/CameraEntity.cs`](../Code/Game/Entity/CameraEntity.cs)、`CameraViewSystem`、`CameraFeed` | 世界摄像头 + 画面输出 |
| 后处理 | [`Code/Game/PostProcessing/`](../Code/Game/PostProcessing/) | 色调映射、特效 |
| 语音 | [`Code/Player/SandboxVoice.cs`](../Code/Player/SandboxVoice.cs) | 语音聊天 |
| 自由相机 | [`Code/FreeCam/`](../Code/FreeCam/) | 旁观/截图相机 |
| 建造 | [`Code/Builder/`](../Code/Builder/) | Builder 工具的几何生成 |
| 信号/线束 | [`Code/Game/ControlSystem/SignalSystem.cs`](../Code/Game/ControlSystem/SignalSystem.cs)、`SeatSignals.cs` | Wire 工具的输入输出 |
| 地图实体 | [`Code/Map/`](../Code/Map/) | `FuncMover`、`TriggerPush`、`TriggerTeleport`、`BaseToggle` |
| 拾取物 | [`Code/Items/Pickups/`](../Code/Items/Pickups/) | 血包、弹药 |

---

## 7. 代码组织与编码约定

### 7.1 partial class：按功能拆文件

一个大类会拆成多个文件，用 `partial` 合并。命名规则 `类名.功能.cs`：

```
Player.cs              ← 核心
Player.Camera.cs       ← 相机
Player.Drive.cs        ← 驾驶
Player.Physgun.cs      ← 物理枪
PlayerInventory.cs     ← 库存
PlayerInventory.Ammo.cs
PlayerInventory.Seat.cs
```

**好处**：每个文件职责单一，改什么找什么。`GameManager`、`FourWheelVehicle`、`Toolgun`、`BaseSandboxWeapon`、`Npc` 都这么做。

### 7.2 命名

- 类/属性/方法：`PascalCase`；
- 私有字段：`_camelCase`（如 `_state`、`_held`）；
- 常量：`PascalCase`（如 `HoldSoundFarDistance`）或 `_greedy` 风格；
- 接口以 `I` 开头：`ISpawner`、`IPlayerEvents`；
- 事件接口以 `Events` 结尾：`IDamageEvents`；
- 特性里显示的文本用 `#localization.key`。

### 7.3 格式

- **Tab 缩进**（见 [`.editorconfig`](../.editorconfig)）；
- CRLF 换行；
- 大括号换行（Allman 风格）；
- 表达式体属性/访问器优先；
- 模式匹配优先（`is not null` 等）。

### 7.4 全局 using

[`Code/Assembly.cs`](../Code/Assembly.cs) 里集中放 `global using`（`System`、`Sandbox`、`System.Linq` 等），新文件不用重复写。

### 7.5 注释风格

项目注释非常详尽，尤其是解释"为什么这么做"（网络、物理、动画陷阱）。建议：

- 写清楚**为什么**，而不是复述代码；
- 网络相关代码注明"只在主机" / "广播"；
- 复杂机制在文件顶部写一段总览（参考 `FourWheelVehicle.cs` 顶部）。

---

## 8. 新手学习路径

建议按这个顺序读，每一步都动手验证：

### 第 1 步：跑起来，建立感性认识（0.5 天）

- 打开项目，运行，玩一遍所有工具（第 2.3 节）；
- 打开 `Assets/scenes/sandbox.scene` 和 `system.scene`，看看里面有什么。

### 第 2 步：理解主干（1-2 天）

按顺序读：

1. [`Code/GameLoop/GameManager.cs`](../Code/GameLoop/GameManager.cs) —— 游戏怎么开始、玩家怎么生成；
2. [`Code/Player/Player.cs`](../Code/Player/Player.cs) —— 玩家怎么回事、怎么受伤死亡；
3. [`Assets/prefabs/engine/player.prefab`](../Assets/prefabs/engine/player.prefab) —— 玩家身上挂了哪些组件；
4. [`Code/GameLoop/GameManager.Spawn.cs`](../Code/GameLoop/GameManager.Spawn.cs) —— 东西怎么生成。

### 第 3 步：武器（1-2 天）

1. [`Code/Game/Weapon/BaseSandboxWeapon/BaseSandboxWeapon.cs`](../Code/Game/Weapon/BaseSandboxWeapon/BaseSandboxWeapon.cs)；
2. 一个最简单的武器：[`Code/Weapons/CrowbarWeapon.cs`](../Code/Weapons/CrowbarWeapon.cs) 或 [`Code/Weapons/FistsWeapon.cs`](../Code/Weapons/FistsWeapon.cs)；
3. [`Code/Weapons/GlockWeapon.cs`](../Code/Weapons/GlockWeapon.cs) → [`Code/Game/Weapon/BaseBulletWeapon/BaseBulletWeapon.cs`](../Code/Game/Weapon/BaseBulletWeapon/BaseBulletWeapon.cs)；
4. [`Code/Player/PlayerInventory.cs`](../Code/Player/PlayerInventory.cs)。

### 第 4 步：工具枪（2-3 天，本项目精华）

1. [`Code/Weapons/ToolGun/ToolMode.cs`](../Code/Weapons/ToolGun/ToolMode.cs) —— **反复读**；
2. [`Code/Weapons/ToolGun/Toolgun.cs`](../Code/Weapons/ToolGun/Toolgun.cs)；
3. 简单工具：`Modes/RemoverTool.cs`、`Modes/ColourTool.cs`、`Modes/WeldTool.cs`；
4. 进阶工具：`Modes/RopeTool.cs`、`Modes/Thruster/`、`Modes/Wheel/`、`Modes/Builder/`。

### 第 5 步：物理枪与生成（1-2 天）

1. [`Code/Weapons/PhysGun/Physgun.cs`](../Code/Weapons/PhysGun/Physgun.cs)；
2. [`Code/Spawner/ISpawner.cs`](../Code/Spawner/ISpawner.cs) + [`PropSpawner.cs`](../Code/Spawner/PropSpawner.cs)。

### 第 6 步：存档 / 撤销 / 限制（1 天）

[`SaveSystem.cs`](../Code/Save/SaveSystem.cs)、[`UndoSystem.cs`](../Code/Player/UndoSystem/UndoSystem.cs)、[`LimitsSystem.cs`](../Code/GameLoop/LimitsSystem.cs)。

### 第 7 步：NPC（1-2 天）

先读 [`Code/Npcs/README.md`](../Code/Npcs/README.md)，再读 `Npc.cs`、`Schedules/`、`Tasks/`。

### 第 8 步：UI / 载具 / 其他（按需）

HUD、生成菜单、`FourWheelVehicle`、`ControlSystem`。

---

## 9. 动手实践：常见任务怎么改

### 9.1 修改武器伤害 / 射速

1. 找到武器类，如 [`Code/Weapons/GlockWeapon.cs`](../Code/Weapons/GlockWeapon.cs)；
2. 伤害通常在子弹逻辑（`BaseBulletWeapon`）或武器的 `[Property]` 里；
3. 射速看 `FireRate` / `ReloadTime` 之类属性；
4. 改完直接热重载（编辑器支持代码热重载），进游戏测试。

### 9.2 新增一个工具（ToolMode）

1. 在 [`Code/Weapons/ToolGun/Modes/`](../Code/Weapons/ToolGun/Modes/) 下新建 `MyTool.cs`；
2. 继承 `ToolMode`，加 `[Title( "#tool.name.mytool" )]`、`[Category( ... )]`、`[Icon( ... )]`；
3. 重写 `OnControl()`，用 `RegisterAction` 注册动作，或直接用 `Input.Pressed("attack1")`；
4. 重写 `DrawHud` 画预览；
5. 在 `Localization/en/tools.json` 加名称和提示；
6. **不需要改 Toolgun** —— 反射会自动发现它。

### 9.3 新增一个实体（Entity）

1. 写一个 `Component` 子类（如 `ThrusterEntity`）；
2. 在 `Assets/entities/` 下做 prefab，挂上你的组件；
3. 用 `[AssetType]` 定义 `.sent` 资源（参考 [`ScriptedEntity.cs`](../Code/Game/Entity/ScriptedEntity.cs)），或在生成菜单里直接引用 prefab；
4. 实体会被 `EntitySpawner` 生成。

### 9.4 新增一个服务器设置（ConVar）

1. 打开 [`Code/GameLoop/ServerSettings.cs`](../Code/GameLoop/ServerSettings.cs)；
2. 加一个静态属性，加 `[ConVar( ... )]` + `[Title]` + `[Group]` + `[Range]`；
3. 它会自动出现在游戏内"服务器设置"界面，并被保存。

### 9.5 新增一个 NPC

1. 复制 `Assets/entities/sents/npc/` 下的 prefab；
2. 写一个 `Npc` 子类，重写 `Faction`、`SetupRelationships`、`GetSchedule`；
3. 给 prefab 加 `NavMeshAgent`、`SkinnedModelRenderer`、`Collider`、`Rigidbody`；
4. 参考 [`Code/Npcs/README.md`](../Code/Npcs/README.md) 和 `Examples/ExampleNpc.cs`。

### 9.6 加一条本地化文本

1. 在 `Localization/en/xxx.json` 加键值；
2. 代码里用 `Game.Language.GetPhrase( "#key" )`；
3. 其他语言可以后补（缺失会回退）。

---

## 10. 术语表

| 术语 | 含义 |
|---|---|
| **prefab** | 预制体，可复用的 GameObject 模板 |
| **scene** | 场景，包含 GameObject 和场景属性 |
| **sent** | Sandbox Entity，脚本化实体资源 |
| **spawnlist** | 生成清单，生成菜单里的一组内容 |
| **dupe** | 复制粘贴出来的机械/建筑 |
| **contraption** | 机械：用约束拼起来的一堆物体 |
| **constraint** | 约束：焊接、绳、球窝、滑轨等连接 |
| **prop** | 道具，世界里的物理模型 |
| **toolgun** | 工具枪，用来创建/修改物体的工具 |
| **physgun** | 物理枪，用来抓取/移动物体 |
| **mount** | 挂载，把另一个包的内容挂进来 |
| **host** | 主机，权威服务器 |
| **proxy** | 网络代理对象，非本地拥有 |
| **tick** | 逻辑帧，本项目 50Hz |
| **RPC** | 远程过程调用 |
| **convar / concommand** | 控制台变量 / 命令 |
| **razor** | s&box 的 UI 框架（.razor 文件） |
| **painter** | HUD 绘制器 |
| **animation graph (ag)** | 动画图，驱动骨骼动画 |
| **NavMesh** | 导航网格，NPC 寻路用 |

---

## 11. 进阶资源

- **s&box 官方文档**：[sbox.facepunch.com](https://sbox.facepunch.com)（API 参考、指南）；
- **本项目仓库**：[github.com/Facepunch/sandbox](https://github.com/Facepunch/sandbox)（欢迎 PR，注意遵守代码风格）；
- **在线游玩**：[sbox.game/facepunch/sandbox](https://sbox.game/facepunch/sandbox)；
- **本文档涉及的重点文件**：
  - [`Code/GameLoop/GameManager.cs`](../Code/GameLoop/GameManager.cs)
  - [`Code/Player/Player.cs`](../Code/Player/Player.cs)
  - [`Code/Weapons/ToolGun/ToolMode.cs`](../Code/Weapons/ToolGun/ToolMode.cs)
  - [`Code/Weapons/PhysGun/Physgun.cs`](../Code/Weapons/PhysGun/Physgun.cs)
  - [`Code/Spawner/ISpawner.cs`](../Code/Spawner/ISpawner.cs)
  - [`Code/Save/SaveSystem.cs`](../Code/Save/SaveSystem.cs)
  - [`Code/Npcs/README.md`](../Code/Npcs/README.md)
  - [`Code/Vehicles/FourWheelVehicle.cs`](../Code/Vehicles/FourWheelVehicle.cs)
  - [`Code/UI/HudLayer.razor`](../Code/UI/HudLayer.razor)

---

> 祝学习愉快！这个项目代码质量很高、注释详尽，是学习 s&box 开发的最佳教材之一。
> 有不懂的地方，先在新手学习路径里找对应文件，再对照游戏内的实际表现，通常都能想明白。
