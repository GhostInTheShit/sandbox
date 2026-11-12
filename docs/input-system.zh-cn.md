# 游戏开发基础 · 第 2 课：输入系统

> 本课只讲**一个知识点**：输入 = **动作名** + **三个状态**。
> 不需要网络知识，约 6 分钟读完。
> 前置：[第 1 课 一帧里发生了什么](game-loop-and-lifecycle.zh-cn.md)。

---

## 1. 动作不是按键

游戏不能直接问"玩家按了 W 没有"——因为玩家可能把手柄左摇杆往前推、可能把 W 键绑定到别的动作。

所以 s&box 用**动作（Action）**这一层抽象：

```
按键 / 摇杆 / 扳机   →  动作（"attack1"、"drop"…）   →  你的代码
```

- 动作名是字符串，比如 `"attack1"`（主攻击）、`"attack2"`（副攻击）、`"drop"`（丢弃）。
- 动作 → 按键的绑定，由玩家在**游戏内输入设置**里自己改（引擎功能）。
- 项目里用到的动作名可以从 [`Localization/en/input.json`](../Localization/en/input.json) 看到：键名 `input.action.attack1` 的**后半段就是动作名**，值是它的显示文字。常用的有：

| 动作 | 含义 |
|---|---|
| `forward` / `backward` / `left` / `right` | 前后左右 |
| `jump` / `duck` / `run` / `walk` | 跳跃 / 蹲 / 跑 / 走 |
| `attack1` / `attack2` | 主攻击 / 副攻击 |
| `reload` | 装填 |
| `use` | 使用 |
| `slot1` … `slot0` | 直接切到第 N 号武器槽 |
| `slotprev` / `slotnext` | 上一个 / 下一个武器 |
| `drop` | 丢弃当前武器 |
| `noclip` / `undo` / `flashlight` / `voice` / `score` | 飞行 / 撤销 / 手电 / 语音 / 计分板 |

代码里永远写动作名，不写按键名：

```csharp
if ( Input.Pressed( "attack1" ) ) { ... }
```

---

## 2. 三个状态：按下 / 按住 / 松开

同一个动作，在每一帧有三种状态：

| 写法 | 含义 | 什么时候用 |
|---|---|---|
| `Input.Pressed( "attack1" )` | **这一帧刚按下**（之前没按） | 半自动武器、切枪、开关 |
| `Input.Down( "attack2" )` | **这一帧正按住** | 瞄准、蓄力、持续效果 |
| `Input.Released( "attack1" )` | **这一帧刚松开** | 松开后装填、松手投掷 |

> 关键区别：`Pressed` 在一帧里只返回 `true` 一次。按住不放，第二帧就是 `false` 了。所以"按住持续生效"要用 `Down`。

### 项目里的真实例子

[`GlockWeapon.cs`](../Code/Weapons/GlockWeapon.cs) —— 半自动，按下就开一枪：

```csharp
protected override bool WantsPrimaryAttack() => Input.Pressed( "attack1" );
```

[`IronSightsWeapon.cs`](../Code/Game/Weapon/IronSightsWeapon.cs) —— 按住右键瞄准：

```csharp
var wantsAim = Input.Down( "attack2" ) && !IsReloading;
```

[`RpgWeapon.cs`](../Code/Weapons/Rpg/RpgWeapon.cs) —— 松开开火键后自动装填：

```csharp
if ( _hasFired && Input.Released( "attack1" ) )
{
    // 玩家松手了，开始装填
}
```

[`HandGrenadeWeapon.cs`](../Code/Weapons/HandGrenade/HandGrenadeWeapon.cs) —— 按下拉环开始"捏雷"，两个键都松开就投出：

```csharp
// 按下 attack1 或 attack2 → 开始烹饪（捏雷）
if ( !IsCooking && CanPrimaryAttack() && ( Input.Pressed( "Attack1" ) || Input.Pressed( "Attack2" ) ) )
{
    IsCooking = true;
    TimeSinceCooked = 0;
}

// 两个键都松开 → 投出去
if ( !Input.Down( "Attack1" ) && !Input.Down( "Attack2" ) )
{
    Throw( player );
}
```

> 注意：这里动作名写的是 `"Attack1"`（大写 A）。项目里两种写法都出现过（引擎可能对动作名大小写不敏感，也可能是历史遗留）；**建议你统一用 `input.json` 里的小写名**，避免自己混淆。

---

## 3. 模拟输入（手柄、摇杆、扳机）

按键只有"按 / 没按"，但摇杆和扳机有**行程**（0 到 1）。这类输入用模拟 API：

| 写法 | 含义 |
|---|---|
| `Input.AnalogLook` | 本帧视角转动量，含 `pitch`（俯仰）和 `yaw`（左右） |
| `Input.GetAnalog( InputAnalog.LeftStickX )` | 某个摇杆 / 扳机的行程，返回 0~1 |
| `Input.SetAction( "Forward", true )` | 程序化地"按下"一个动作 |

### 例子 1：用视角转量做变焦（[`CameraWeapon.cs`](../Code/Weapons/CameraWeapon.cs)）

```csharp
fov += Input.AnalogLook.pitch;   // 转视角 → 改焦距
roll -= Input.AnalogLook.yaw;
```

### 例子 2：手柄开车（[`Player.cs`](../Code/Player/Player.cs) 的 `PressSeatedGamepad`）

```csharp
// 手柄扳机 = 前进/后退，左摇杆 = 转向
var steer = Input.GetAnalog( InputAnalog.LeftStickX );
PressAnalog( "Forward",  Input.GetAnalog( InputAnalog.RightTrigger ), 0.05f );
PressAnalog( "Backward", Input.GetAnalog( InputAnalog.LeftTrigger ),  0.05f );
PressAnalog( "Left", -steer, 0.15f );
PressAnalog( "Right", steer, 0.15f );
```

`PressAnalog` 的实现（[`Player.cs`](../Code/Player/Player.cs)）：

```csharp
private void PressAnalog( string action, float value, float deadzone )
{
    // 去掉"死区"（摇杆轻微偏移不算），再归一化到 0~1
    var amount = ((value - deadzone) / (1.0f - deadzone)).Clamp( 0.0f, 1.0f );

    // 键盘按住这个动作 = 直接算满
    if ( Input.Down( action ) )
        amount = 1.0f;
    else if ( amount > 0.0f )
        Input.SetAction( action, true );      // 程序化"按下"这个动作

    // 顺便记住行程大小（给其他玩家看方向盘转多少）
    if ( !ActionAmounts.TryGetValue( action, out var last ) || last != amount )
        ActionAmounts[action] = amount;
}
```

> 座位上的"开车"没有专门写一套输入，而是把手柄映射成和 WASD 一样的动作（`Forward` / `Backward` / `Left` / `Right`）。这样**任何座位都能用手柄开**，不需要单独适配。

> `ActionAmounts` 是一个 `[Sync]` 字典，把"按了多少"同步给其他玩家，这样别人开车时你也能看到方向盘转多少。

---

## 4. 可配置的输入（座位 / 机械用）

普通武器直接用 `Input`。但**座位和机械**里的按钮是玩家自己配的（比如"机关的红色按钮"），所以要用 `ClientInput` 这种可在编辑器里配置的输入：

```csharp
// BaseSandboxWeapon 上声明的两个输入（在编辑器里配置）
public ClientInput ShootInput { get; set; }
public ClientInput SecondaryInput { get; set; }
```

用法和 `Input` 类似，也是 `Pressed()` / `Down()` / `Released()` 三个状态，但多一个 `IsEnabled`（判断这个输入有没有被接上）：

```csharp
// 座位/机械控制时，每帧把三个状态分发出去（BaseSandboxWeapon.OnContraptionControl）
protected virtual void OnContraptionControl()
{
    DispatchPrimary( ShootInput.Pressed(), ShootInput.Down(), ShootInput.Released() );
    DispatchSecondary( SecondaryInput.Pressed(), SecondaryInput.Down(), SecondaryInput.Released() );
}

// IsEnabled 还能判断"这把枪是不是被接进了机械"（BaseSandboxWeapon.OnCanPickup）
protected override bool OnCanPickup( Sandbox.BaseInventoryComponent inventory )
{
    var isLinked = GameObject.Root.GetComponentsInChildren<ManualLink>( true ).Any();
    return !isLinked && !ShootInput.IsEnabled && !SecondaryInput.IsEnabled;
}
```

> `ClientInput` 是引擎类型，不在本项目源码里；细节以引擎文档为准。

---

## 5. 常见坑

| 坑 | 说明 |
|---|---|
| 用 `Pressed` 做持续效果 | `Pressed` 只持续一帧，第二帧就 `false`，效果会"闪一下" |
| 动作名拼错 | 不会报错，只会永远返回 `false`（静默失败） |
| 在 `OnStart` 里读输入 | 只读一次就结束，应该放在每帧调用的事件里（如 `OnControl`） |
| 输入和表现混在一起 | 输入判断放 `Player.OnControl`，具体效果放武器/座位 |

> 输入处理的统一入口：[`Player.OnControl`](../Code/Player/Player.cs) → 依次调用 `inventory.OnControl`、`Inventory.HandleInput`、`Pump`、`HandleScrollInput` 等。想知道"按了切枪键会发生什么"，从这里追。

---

## 6. 小结

- 输入 = **动作名** + 三个状态：`Pressed` / `Down` / `Released`。
- 按键用 `Input`，摇杆/扳机用 `Input.GetAnalog` / `Input.AnalogLook`。
- 座位/机械的可配置输入用 `ClientInput`。
- 输入判断放在每帧调用的事件里，不要放 `OnStart`。

**本课跳过**：`[Sync]` 同步输入、主机如何处理座位输入（网络，最后一课讲）。

---

## 下一课

- [第 3 课：事件系统](event-system.zh-cn.md) — 游戏功能的"插线板"
- [第 4 课：功能设计实战](feature-design-patterns.zh-cn.md) — 组件 + 配置 + 自动发现
