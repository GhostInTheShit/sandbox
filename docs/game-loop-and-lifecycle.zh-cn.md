# 游戏开发基础 · 第 1 课：一帧里发生了什么

> 本课只讲**一个知识点**：游戏就是一个循环，引擎每一帧都会调用你的代码。
> 不需要任何网络知识，约 8 分钟读完。
> 前置：知道什么是 GameObject / Component（见[学习指南](learning-guide.zh-cn.md)第 3 节）。

---

## 1. 游戏就是"循环播放的动画片"

- 屏幕每秒刷新几十次，每次刷新显示一张画面，一张画面就是**一帧（Frame）**。
- 游戏引擎的工作，就是一遍又一遍地重复做三件事：

```
读取输入  →  更新世界  →  画出画面  →  再读输入 → ...
```

- 你的代码不是"从第一行跑到最后一行就结束"，而是**挂在循环上**：引擎在每帧的固定位置回头调用你写的方法。

> 记住这句话：**游戏不会自己"往下走"，它只是每帧调用你。** 你负责在每次被调用时改变世界的状态。

---

## 2. 组件的生命周期：引擎什么时候调用你

继承 `Component` 的类，可以重写下面这些方法（叫"钩子"）：

| 钩子 | 什么时候调用 | 典型用途 |
|---|---|---|
| `OnStart` | 组件第一次启用时，**只调用一次** | 初始化、缓存组件、读取配置 |
| `OnEnabled` / `OnDisabled` | 组件被启用 / 禁用时 | 打开 / 关闭功能 |
| `OnUpdate` | **每一帧**（渲染帧） | 表现、动画参数、每帧检查 |
| `OnFixedUpdate` | 固定节奏（和物理 / tick 相关） | 物理、需要稳定间隔的逻辑 |
| `OnPreRender` | 渲染之前 | 相机、渲染相关 |
| `OnDestroy` | 组件销毁时 | 清理、恢复状态 |
| `OnParentDestroy` | 父对象被销毁时 | 防止自己跟着被销毁 |

### 真实例子 1：`OnUpdate`（每帧）

[`Npc.cs`](../Code/Npcs/Npc.cs)：

```csharp
protected override void OnUpdate()
{
    if ( IsProxy ) return;          // 网络：先跳过（第 5 课再讲）

    CategorizeGround();             // 判断脚踩在哪
    UpdateAirborne();               // 判断是不是在空中

    // 不在空中才思考（省性能）
    if ( !IsAirborne )
        TickSchedule();
}
```

### 真实例子 2：`OnFixedUpdate`（固定节奏）

[`PlayerInventory.Seat.cs`](../Code/Player/PlayerInventory.Seat.cs)：

```csharp
protected override void OnFixedUpdate()
{
    base.OnFixedUpdate();

    if ( !Networking.IsHost ) return;

    // 检查玩家是不是坐在座位上（不需要每渲染帧都查，固定节奏就够）
    var seat = GetComponentInParent<ISitTarget>( false );
    ...
}
```

### 真实例子 3："只调用一次"的初始化钩子

[`BaseSandboxWeapon.cs`](../Code/Game/Weapon/BaseSandboxWeapon/BaseSandboxWeapon.cs) 用的是引擎物品钩子 `OnAdded`（武器被加进背包时调用一次）：

```csharp
protected override void OnAdded( Sandbox.BaseInventoryComponent inventory )
{
    IsDropped = false;      // 进了背包，就不再是"掉落物"
    base.OnAdded( inventory );
}
```

> 项目里有很多"只调用一次"的钩子：组件的 `OnStart`、物品的 `OnAdded`、工具的 `OnStart`。它们都适合放初始化代码。

---

## 3. 一帧里大致发生什么（简化理解）

```
① 读输入    玩家按了什么键
② 模拟      物理、固定节奏的逻辑（OnFixedUpdate）
③ 每帧更新  动画、表现、每帧检查（OnUpdate）
④ 相机合成  决定这一帧从哪看、看多大
⑤ 渲染      把画面画出来（OnPreRender）
```

项目里对应的钩子：

| 阶段 | 在哪 | 例子 |
|---|---|---|
| ① 输入 | `PlayerController.IEvents.PreInput` → `Player.OnControl` | 检查切枪、丢枪、飞行 |
| ③ 每帧更新 | `Player.OnUpdate` | 更新动画状态（`b_noclip`、`holdtype`） |
| ④ 相机 | `Player.Camera.cs` 的 `ModifyCamera` | 第一/第三人称、视角倾斜 |
| ⑤ 渲染前 | `CameraEntity.OnPreRender` | 更新摄像头画面的取景（`Feed`） |

> 这是帮你建立直觉的简化版，真实引擎的细节顺序可能略有不同。**重要的是知道每个钩子适合放什么代码。**

---

## 4. 时间：帧与计时器

做游戏离不开时间。项目里最常用的三个工具：

| 工具 | 含义 | 例子 |
|---|---|---|
| `Time.Delta` | 上一帧到这一帧过了多少**秒** | 平滑插值、移动 |
| `TimeSince` | 从我上次设置它开始，过了多少秒（**累计**） | 冷却、间隔 |
| `TimeUntil` | 离我设置的时刻还有多少秒（**倒计时**） | 延迟执行 |

### 例子 1：间隔（`TimeSince`）

[`BaseBulletWeapon.cs`](../Code/Game/Weapon/BaseBulletWeapon/BaseBulletWeapon.cs) 用 `TimeSince` 限制枪声广播频率：

```csharp
private TimeSince _timeSinceNoise;

void EmitGunfireNoise( ShotEffect shot )
{
    // 距上次广播不到 0.25 秒就跳过
    if ( !Networking.IsHost || shot.NoEvents || _timeSinceNoise < 0.25f ) return;

    _timeSinceNoise = 0;   // 重置计时器，重新计时
    ...
}
```

### 例子 2：冷却（引擎武器也是这个套路）

[`MeleeWeapon.cs`](../Code/Game/Weapon/MeleeWeapon/MeleeWeapon.cs)：

```csharp
// 命中和挥空用不同的冷却时间（代替固定的 FirePrimary 冷却）
SetNextPrimaryFire( tr.GameObject.IsValid() ? SwingDelay : MissSwingDelay );
```

引擎里 `NextPrimaryFire` 就是一个"下一次最早能开火的时间"，到点前 `WantsPrimaryAttack` 会被引擎挡掉。

### 例子 3：延迟执行（`Invoke`）

[`ViewModel.cs`](../Code/Game/Weapon/ViewModel/ViewModel.cs)：

```csharp
// 0.5 秒后再做某事
Invoke( 0.5f, () =>
{
    Renderer?.Set( "b_deploy_new", true );
    Renderer?.Set( "b_pull", false );
} );
```

### 例子 4：平滑插值（`Time.Delta`）

[`BaseSandboxWeapon.cs`](../Code/Game/Weapon/BaseSandboxWeapon/BaseSandboxWeapon.cs)：

```csharp
// 每帧朝目标转一点，转多快取决于帧间隔
_cursorLook = Rotation.Slerp( _cursorLook, target, Time.Delta * 10f );
```

> **为什么用 `Time.Delta`？** 因为不同电脑的帧率不一样。用"每帧固定转 10%"会导致高帧率电脑转得飞快；用 `Time.Delta * 10f` 才是"每秒转 10%"，和帧率无关。

---

## 5. 小结

- 游戏 = 循环；你的代码挂在引擎的钩子上。
- `OnStart` 一次；`OnUpdate` 每帧；`OnFixedUpdate` 固定节奏；`OnDestroy` 清理。
- 经验法则：**表现放 `OnUpdate`，稳定逻辑放 `OnFixedUpdate`，初始化放 `OnStart`**。
- 时间三件套：`Time.Delta`（帧间隔）、`TimeSince`（累计计时）、`TimeUntil`（倒计时）。
- 本项目 `sandbox.sbproj` 里的 `TickRate` 是 50，表示模拟每秒跑 50 次。

**本课跳过**：`IsProxy` / `Networking.IsHost`（网络，最后一课讲）。

---

## 下一课

- [第 2 课：输入系统](input-system.zh-cn.md) — 按键怎么变成动作
- [第 3 课：事件系统](event-system.zh-cn.md) — 游戏功能的"插线板"
- [第 4 课：功能设计实战](feature-design-patterns.zh-cn.md) — 组件 + 配置 + 自动发现
