# Player.cs 讲解 · 第 1 课：认识 Player（新手版）

> 这是一套**渐进课程**的第 1 课。
> 本课只讲**一个知识点**：`Player` 是挂在玩家身上、负责"收输入 + 存数据"的组件。
> 全文约 10 分钟读完；**网络（Sync / Rpc / IsProxy）放到最后一课**，本课基本不涉及。
> 前置知识：知道什么是 GameObject / Component（见[学习指南](learning-guide.zh-cn.md)第 3 节）。

---

## 本课只讲一件事

打开 [`Code/Player/Player.cs`](../Code/Player/Player.cs)，你会看到一个 700 行的大文件，不用慌。它做的事可以概括成三类：

| 职责 | 例子 |
|---|---|
| **存数据** | 血量、护甲、背包、玩家数据 |
| **每帧收输入** | 按键 → 切枪、丢枪、飞行、撤销 |
| **被引擎回调** | 落地、跳、受伤、死亡 |

本课只深入第 1、2 类里最基础的部分。其他的（受伤、死亡、相机、网络）都会单独开一课。

---

## 1. 类声明：Player 是什么

```csharp
// sealed：不允许被继承
// partial：这个类被拆成多个文件（Player.cs、Player.Camera.cs、Player.Static.cs……）
public sealed partial class Player : Component, Component.IDamageable, PlayerController.IEvents,
    Global.ISaveEvents, IKillSource, IHudEvents, IDamageBlame
{
```

逐个解释：

| 名字 | 是什么 |
|---|---|
| `Component` | 引擎的**组件基类**。继承它，这个类就能挂在 GameObject 上，并被引擎每帧回调 |
| `Component.IDamageable` | 一个接口，意思是"**我能被伤害**"。子弹/爆炸打到你时，引擎会回调 `OnDamage` |
| `PlayerController.IEvents` | 移动控制器的接口，引擎会在固定时机回调（见第 3 节） |
| `Global.ISaveEvents` | 存档 / 读档时回调 |
| `IKillSource` | 击杀信息：名字、SteamId、击杀数 |
| `IHudEvents` | 告诉 HUD "现在要隐藏哪些界面元素" |
| `IDamageBlame` | 查"这次伤害算在谁头上" |

> 接口先不用深究，只要记住：**接口 = 引擎在合适时机回调你的钩子**。

`partial` 意味着这个类分散在多个文件里：

| 文件 | 内容 |
|---|---|
| [`Player.cs`](../Code/Player/Player.cs) | 本课内容：血量、输入、死亡流程 |
| [`Player.Camera.cs`](../Code/Player/Player.Camera.cs) | 第一 / 第三人称相机（下一课） |
| [`Player.Static.cs`](../Code/Player/Player.Static.cs) | 静态查找：`FindLocalPlayer()`、`Player.For( id )` |
| [`Player.Drive.cs`](../Code/Player/Player.Drive.cs) | 开车 |
| [`Player.Physgun.cs`](../Code/Player/Player.Physgun.cs) | 物理枪 |
| [`Player.Undo.cs`](../Code/Player/Player.Undo.cs) | 撤销 |

---

## 2. 最小的属性例子：Health

```csharp
[Property, Range( 0, 100 ), Sync( SyncFlags.FromHost )]
public float Health { get; set; } = 100;
```

这一行是理解整个项目的钥匙。拆开看：

| 部分 | 含义 |
|---|---|
| `[Property]` | 这个属性会**显示在编辑器面板里**，可以在 prefab 中配置 |
| `[Range( 0, 100 )]` | 面板里显示成 **0~100 的滑块** |
| `[Sync( SyncFlags.FromHost )]` | **网络同步**：主机改了，所有客户端都会收到。现在只要记住这一句，细节最后一课讲 |
| `public float Health { get; set; }` | 普通的 C# 属性 |
| `= 100` | **默认值**：玩家生成时血量是 100 |

旁边还有 `MaxHealth`、`Armour`、`MaxArmour`，写法完全一样：

```csharp
[Property, Range( 0, 100 ), Sync( SyncFlags.FromHost )]
public float Armour { get; set; } = 0;   // 护甲默认 0
```

> 记住这个套路：**数据 = `[Property]` + 默认值 +（需要同步就加 `[Sync]`）**。项目里 90% 的属性都是这个形状。

---

## 3. 玩家身上的"零件"

一个玩家 GameObject 上挂着好几个组件，`Player` 只是其中之一：

```csharp
// 这个 GameObject 上必须有 PlayerController，没有的话引擎会自动加上
[RequireComponent]
public PlayerController Controller { get; set; }

// 第三人称时渲染的身体（第一人称时你看不到它）
[Property]
public GameObject Body { get; internal set; }

// 眼睛的位置和朝向 —— 第一人称相机就放在这里
public Transform EyeTransform => Controller.EyeTransform;
```

| 零件 | 作用 |
|---|---|
| `Player` | 本类：血量、输入、死亡 |
| `PlayerController` | 引擎组件：**移动、跳跃、视角、物理**（走路 logic 在这里，不在 Player.cs） |
| `Body` | 你第三人称看到的身体模型 |
| `EyeTransform` | 眼睛位置，开枪、相机都用它 |
| `PlayerInventory` | 背包（另一个组件，见[武器系统讲解](weapon-system-walkthrough.zh-cn.md)） |

> 一个常见疑问：**"走路代码在哪？"** 不在 `Player.cs`，而在引擎的 `PlayerController`。`Player.cs` 只负责一件事：**允许不允许移动**（见第 4 节第一行）。

---

## 4. 每帧做什么：OnUpdate

引擎每一帧都会调用一次 `OnUpdate`（Frame = 画面刷新一次，通常一秒 60 次）。`Player.cs` 里的内容：

```csharp
protected override void OnUpdate()
{
    // 1. 告诉动画模型"现在在做什么状态"
    if ( Controller.IsValid() && Controller.Renderer.IsValid() )
    {
        var seated = GetComponentInParent<ISitTarget>( false ) is not null;   // 是不是坐在座位上
        Controller.Renderer.Set( "b_noclip", IsNoclipping && !seated );        // 告诉模型：在飞

        // 空手时，把持枪姿势重置掉
        if ( GetComponent<PlayerInventory>()?.ActiveWeapon.IsValid() != true )
        {
            Controller.Renderer.Set( "holdtype", "none" );
            Controller.Renderer.Set( "holdtype_handedness", 0 );
        }
    }

    // 2. 只有"你自己"才需要做的事
    if ( IsLocalPlayer )
    {
        UpdateSeatedWeapons();      // 座位上的武器
        UpdateSeatedBodyVisibility();
        DrawSeatedWeaponHud();      // 画座位武器的准星
        UpdatePhysgunColors();      // 物理枪颜色
        NpcWeapon = GamePreferences.NpcWeapon;
    }
}
```

两个名词：

- **`Renderer.Set("名字", 值)`**：给动画模型传一个参数，动画图会读它来决定播什么动作。比如 `b_noclip = true` 时模型会播飞行姿势。这些"名字"是模型和动画图约定的，不是随便起的。
- **`IsLocalPlayer`**：这个玩家**是不是你自己**（你屏幕前这个）。别人的玩家对象上，这个值是 `false`。

第 2 段为什么只对"你自己"执行？因为 HUD、物理枪颜色这些是**本地表现**，每个客户端只管自己的屏幕。

---

## 5. 输入怎么变成动作：OnControl

引擎每帧在"处理输入之前"回调一次 `PreInput`，`Player.cs` 用它来检查按键：

```csharp
void PlayerController.IEvents.PreInput()
{
    OnControl();   // 每帧检查一次玩家按了什么
}
```

`OnControl` 的内容，按顺序列出来就是一张表：

```csharp
void OnControl()
{
    // ① 先允许移动（插件可以把它改成 false 来禁止移动）
    _allowMovement = true;
    Local.IPlayerEvents.Post( static x => x.OnMove( ref _allowMovement ) );
    Controller.UseInputControls = _allowMovement;

    // ② 自杀（动作名 "die"，默认按键以游戏内输入设置为准）
    if ( Input.Pressed( "die" ) )
    {
        KillSelf();
        return;
    }

    // ③ 正在开车 → 后面的都不管
    if ( ControlDriving() )
        return;

    // ④ 座位上的手柄输入（略）
    var seated = _cachedSeat is not null;
    if ( seated ) PressSeatedGamepad();
    else if ( ActionAmounts.Count > 0 ) ActionAmounts.Clear();

    // ⑤ 切换飞行（noclip）
    if ( Input.Pressed( "noclip" ) )
        ToggleNoclip();

    // ⑥ 撤销
    if ( Input.Pressed( "undo" ) )
        ConsoleSystem.Run( "undo" );

    // ⑦ 搬东西
    GetComponent<PlayerCarry>()?.OnControl();

    // ⑧ 背包：丢武器
    var inventory = GetComponent<PlayerInventory>();
    inventory?.OnControl();

    // ⑨ 数字键切枪
    var weaponSelection = Scene.Get<Inventory>();
    weaponSelection?.HandleInput();

    // ⑩ 手动泵动（霰弹枪上膛）
    if ( inventory.IsValid() && inventory.ManualPumping )
        inventory.Pump();

    // ⑪ 滚轮切枪
    weaponSelection?.HandleScrollInput();
}
```

| # | 代码 | 作用 |
|---|---|---|
| ① | `Controller.UseInputControls = _allowMovement` | 允许 / 禁止移动 |
| ② | `Input.Pressed("die")` | 自杀 |
| ③ | `ControlDriving()` | 开车时跳过后面所有检查 |
| ⑤ | `Input.Pressed("noclip")` | 切换飞行模式 |
| ⑥ | `Input.Pressed("undo")` | 撤销上一个操作 |
| ⑦ | `PlayerCarry.OnControl()` | 搬运动作 |
| ⑧ | `inventory.OnControl()` | 丢武器（动作名 `drop`） |
| ⑨ | `Inventory.HandleInput()` | 数字键切枪 |
| ⑪ | `HandleScrollInput()` | 滚轮切枪 |

> `Input.Pressed("drop")` 里的 `"drop"` 是**动作名**，不是按键名。玩家实际按的是哪个键，在游戏内的"输入设置"里改。

---

## 6. 走一遍：按丢武器键会发生什么

这是把第 4、5 节串起来的最小例子。

**第 1 步**：引擎回调 `PreInput()` → `OnControl()`（第 5 节）。

**第 2 步**：执行到 `inventory?.OnControl()`，进入 [`PlayerInventory.cs`](../Code/Player/PlayerInventory.cs)：

```csharp
public void OnControl()
{
    if ( Input.Pressed( "drop" ) && ActiveWeapon.IsValid() )
        DropActiveWeapon();   // 扔出当前武器
}
```

**第 3 步**：`DropActiveWeapon()` → `Drop( ActiveWeapon )` → 武器被扔到世界坐标（具体流程见[武器系统讲解](weapon-system-walkthrough.zh-cn.md)第 2.4 节）。

**第 4 步**：同帧第 ⑨ 步 `Inventory.HandleInput()` 检查数字键，如果此时手里没枪了，就切换到背包里最好的那把。

整个链路就是：**按键 → `OnControl` → 背包 → 武器**。项目里大部分"按键功能"都是这个形状。

---

## 7. 本课小结

- `Player` 是一个 **Component**，挂在玩家 GameObject 上，和 `PlayerController`、`PlayerInventory` 并列。
- **属性 = 数据 + `[Property]` + 默认值**；需要别的玩家看到就加 `[Sync]`（先记住，以后讲）。
- **`OnUpdate` 每帧执行**：用来更新动画状态、处理本地表现。
- **`OnControl` 每帧检查输入**：按顺序处理 移动开关 → 自杀 → 开车 → 飞行 → 撤销 → 搬运 → 丢枪 → 切枪。
- 移动本身在引擎的 `PlayerController` 里，`Player.cs` 只决定"让不让移动"。

**本课故意跳过内容**（以后单独开课）：

| 主题 | 为什么跳过 |
|---|---|
| 血量 / 受伤 / 死亡细节 | 涉及伤害事件和网络，内容较多 |
| 相机（第一/第三人称） | 独立且完整的一块 |
| 布娃娃 / 幽灵 | 死亡流程的一部分 |
| 驾驶 | 独立系统 |
| `Sync` / `Rpc` / `IsProxy` | 网络，最后一课 |

---

## 下一课（预告）

按同样的"一个知识点"粒度，我建议的顺序是：

1. **第 2 课：血量与受伤**（护甲怎么算、爆头怎么算、死亡流程骨架）
2. **第 3 课：相机**（第一/第三人称切换、视角倾斜、座位相机）
3. **第 4 课：死亡与布娃娃**（尸体、幽灵、观战）
4. **第 5 课（进阶）：网络是怎么回事**（`Sync` / `Rpc` / `IsProxy` / 主机与客户端）

看完本课告诉我，我接着写第 2 课。
