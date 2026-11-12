# 武器系统讲解（中文注释版）

> 本文讲解 s&box Sandbox 的武器系统：从引擎基类 `Sandbox.BaseCombatWeapon` 到游戏基类 [`BaseSandboxWeapon`](../Code/Game/Weapon/BaseSandboxWeapon/BaseSandboxWeapon.cs)，再到子弹枪、近战、机瞄、第一人称视模、投掷物与具体武器。
> 建议对照原文阅读，文中代码片段带有中文注释。
> 前置阅读：[学习指南](learning-guide.zh-cn.md)、[GameManager 讲解](gamemanager-walkthrough.zh-cn.md)。

---

## 目录

1. [总览与继承链](#1-总览与继承链)
2. [核心基类 BaseSandboxWeapon](#2-核心基类-basesandboxweapon)
3. [背包、弹药与死亡掉落](#3-背包弹药与死亡掉落)
4. [后坐力与布娃娃冲击](#4-后坐力与布娃娃冲击)
5. [子弹枪 BaseBulletWeapon](#5-子弹枪-basebulletweapon)
6. [近战 MeleeWeapon](#6-近战-meleeweapon)
7. [机瞄 IronSightsWeapon](#7-机瞄-ironsightsweapon)
8. [第一人称视模 ViewModel](#8-第一人称视模-viewmodel)
9. [投掷物与爆炸](#9-投掷物与爆炸)
10. [具体武器与新增实践](#10-具体武器与新增实践)
11. [总结与阅读建议](#11-总结与阅读建议)

---

## 1. 总览与继承链

### 继承链

```
Sandbox.BaseInventoryItem                         （引擎）背包物品：DisplayName、Slot、Value、ViewModel、WorldModel
  └─ Sandbox.BaseCombatWeapon                     （引擎）战斗武器：弹道、弹药、装填、射击、枪口、动画事件
       └─ BaseSandboxWeapon                       （游戏）沙盒集成：Owner、座位控制、拾取丢弃、摄像机、HUD
            ├─ BaseBulletWeapon                   （游戏）后坐力、枪声、布娃娃冲击、子弹碰撞规则
            │    ├─ IronSightsWeapon              （游戏）机瞄：FOV 缩放、散布收窄
            │    │    ├─ GlockWeapon / Colt1911Weapon / RevolverWeapon / M4a1Weapon / ShotgunWeapon
            │    ├─ Mp5Weapon                     （子弹 + 榴弹发射器）
            │    └─ SniperWeapon                  （开镜狙击）
            ├─ MeleeWeapon                        （近战挥砍）
            │    ├─ CrowbarWeapon
            │    └─ FistsWeapon
            ├─ HandGrenadeWeapon                  （cooking 手雷）
            ├─ RpgWeapon                          （制导火箭）
            └─ ScreenWeapon                       （带屏幕的武器）
                 ├─ Toolgun
                 └─ Physgun
```

一句话概括分工：**引擎负责"一把枪怎么开火、怎么装填、弹道怎么算"；游戏层负责"这把枪在沙盒里怎么被捡起来、被丢掉、被座位控制、被打死时怎么处理、视模怎么摆动"。**

### 文件地图

| 文件 | 内容 |
|---|---|
| [`Code/Game/Weapon/BaseSandboxWeapon/BaseSandboxWeapon.cs`](../Code/Game/Weapon/BaseSandboxWeapon/BaseSandboxWeapon.cs) | 游戏武器基类（本文第 2 节） |
| [`BaseSandboxWeapon.Kick.cs`](../Code/Game/Weapon/BaseSandboxWeapon/BaseSandboxWeapon.Kick.cs) | 后坐力 / 视图冲击 / 震屏参数与 `KickView` |
| [`BaseSandboxWeapon.Reloading.cs`](../Code/Game/Weapon/BaseSandboxWeapon/BaseSandboxWeapon.Reloading.cs) | 装填取消窗口 |
| [`BaseSandboxWeapon.ViewModel.cs`](../Code/Game/Weapon/BaseSandboxWeapon/BaseSandboxWeapon.ViewModel.cs) / [`BaseSandboxWeapon.WorldModel.cs`](../Code/Game/Weapon/BaseSandboxWeapon/BaseSandboxWeapon.WorldModel.cs) | 视模说明、世界模型与掉落状态 |
| [`Code/Game/Weapon/BaseBulletWeapon/BaseBulletWeapon.cs`](../Code/Game/Weapon/BaseBulletWeapon/BaseBulletWeapon.cs) | 子弹枪基类 |
| [`Code/Game/Weapon/MeleeWeapon/MeleeWeapon.cs`](../Code/Game/Weapon/MeleeWeapon/MeleeWeapon.cs) | 近战基类 |
| [`Code/Game/Weapon/IronSightsWeapon.cs`](../Code/Game/Weapon/IronSightsWeapon.cs) | 机瞄基类 |
| [`Code/Game/Weapon/ViewModel/ViewModel.cs`](../Code/Game/Weapon/ViewModel/ViewModel.cs)、[`ViewModel.Throwables.cs`](../Code/Game/Weapon/ViewModel/ViewModel.Throwables.cs) | 第一人称视模 |
| [`Code/Game/Weapon/ProjectileEntity.cs`](../Code/Game/Weapon/ProjectileEntity.cs) | 可爆炸投掷物基类 |
| [`Code/Player/PlayerInventory.cs`](../Code/Player/PlayerInventory.cs)、[`PlayerInventory.Ammo.cs`](../Code/Player/PlayerInventory.Ammo.cs)、[`PlayerInventory.Seat.cs`](../Code/Player/PlayerInventory.Seat.cs) | 背包 |
| [`Code/Player/PlayerLoadout.cs`](../Code/Player/PlayerLoadout.cs) | 死亡装备保存与恢复 |
| [`Code/Game/Explosion.cs`](../Code/Game/Explosion.cs) | 爆炸事件与生成 |
| [`Code/Weapons/`](../Code/Weapons/) | 具体武器 |

---

## 2. 核心基类 BaseSandboxWeapon

### 2.1 类声明与 Owner

```csharp
// 引擎的 BaseCombatWeapon 已经提供了：DisplayName、DisplayIcon、Value、Slot、ViewModel、WorldModel、
// MuzzleGameObject、弹药、装填、FirePrimary/FireSecondary、弹道（Ballistics）等。
// 游戏层只补上"沙盒集成"所需的部分。
public partial class BaseSandboxWeapon : Sandbox.BaseCombatWeapon, IKillIcon, IPlayerControllable
{
    // 隐藏引擎的 PlayerController 版本 Owner，游戏代码统一用 Player
    public new Player Owner => GetComponentInParent<Player>( true );

    // 是否被玩家拿在手里（区别于"放在地上"或"装在座位上"）
    public bool HasOwner => Owner.IsValid();
}
```

要点：`HasOwner` 是后续所有分支的总开关——有主人走"预测 + 本地表现"，没有主人走"权威 / 物理"。

### 2.2 未持用瞄准与 Attacker

一把没被拿着的枪（比如工具枪摆出来的枪、装在载具座位上的枪）也要能开火，`UnheldAimRay` 就是它的瞄准来源：

```csharp
protected override Ray UnheldAimRay
{
    get
    {
        // 1. 座位/机械控制：如果武器支持"瞄准点控制"，直接用场景相机方向
        var seated = ControlContext.Player;
        if ( seated.IsValid() && IsTargetedAim && Scene.Camera.IsValid() )
            return Scene.Camera.Transform.World.ForwardRay;

        // 2. NPC 持枪：从枪口射向目标胸口（跟随蹲下），而不是沿枪管
        if ( NpcAimTarget.IsValid() )
        {
            var muzzle = GetMuzzleTransform().Position;
            return new Ray( muzzle, (GetChestPosition( NpcAimTarget ) - muzzle).Normal );
        }

        // 3. 其他：引擎默认（枪口方向）
        return base.UnheldAimRay;
    }
}

// 伤害归属：手持者 > 座位控制者 > 最近的 IKillSource > 武器自己
protected override GameObject Attacker
{
    get
    {
        if ( HasOwner ) return Owner.GameObject;

        var seatedPlayer = ControlContext.Player;
        if ( seatedPlayer.IsValid() ) return seatedPlayer.GameObject;

        var killSource = GetComponentInParent<IKillSource>( true );
        if ( killSource is Component c ) return c.GameObject;

        return GameObject;
    }
}
```

### 2.3 座位与机械控制

玩家坐上载具、或把枪接线进机械（`ManualLink`）时，武器的开火输入不再来自玩家按键，而是来自座位的输入绑定或机械信号。接口定义在 [`Code/Game/ControlSystem/IPlayerControllable.cs`](../Code/Game/ControlSystem/IPlayerControllable.cs)。

```csharp
// 引擎在玩家"控制某个座位/机械"时回调
void IPlayerControllable.OnControl()
{
    if ( HasOwner ) return;   // 被玩家拿在手里时不走这条路径
    OnContraptionControl();
}

protected virtual void OnContraptionControl()
{
    DispatchPrimary( ShootInput.Pressed(), ShootInput.Down(), ShootInput.Released() );
    DispatchSecondary( SecondaryInput.Pressed(), SecondaryInput.Down(), SecondaryInput.Released() );
}

private void DispatchPrimary( bool pressed, bool down, bool released )
{
    if ( pressed ) OnPrimaryPressed();
    if ( down ) OnPrimaryDown();       // 默认 => FirePrimary()
    if ( released ) OnPrimaryReleased();
}
```

`ShootInput` / `SecondaryInput` 是可在编辑器里配置的输入绑定；`[SignalInput]` 让机械信号（比如按钮）也能触发同一套流程：

```csharp
[SignalInput( Id = nameof( ShootInput ), Default = true )]
public void ShootSignal( SignalEvent input )
{
    if ( HasOwner || !Networking.IsHost ) return;
    DispatchPrimary( input.Pressed, input.Down, input.Released );
}
```

相关的三个门闩：

```csharp
// 接进机械（有 ManualLink）或已绑定座位输入的武器，不再能被捡起
protected override bool OnCanPickup( Sandbox.BaseInventoryComponent inventory )
{
    var isLinked = GameObject.Root.GetComponentsInChildren<ManualLink>( true ).Any();
    return !isLinked && !ShootInput.IsEnabled && !SecondaryInput.IsEnabled;
}

// 玩家坐在座位上且已收枪时，禁止切换武器
protected override bool OnCanSwitchTo()
{
    if ( Inventory is PlayerInventory { IsSeatHolstered: true } )
        return false;

    return base.OnCanSwitchTo();
}

// 玩家手里有武器时，不能控制座位武器
public virtual bool CanControl( Player player )
{
    var inventory = player.GetComponent<PlayerInventory>();
    return inventory is null || !inventory.ActiveWeapon.IsValid();
}
```

### 2.4 丢弃与收枪

```csharp
// 引擎要丢弃这把武器时：把"活的"武器对象扔到世界坐标，保留弹药和工具模式
protected override bool OnDrop()
{
    var player = Owner;

    if ( player.IsValid() )
    {
        var eye = player.EyeTransform;
        var velocity = (player.Controller.IsValid() ? player.Controller.Velocity : Vector3.Zero)
            + eye.Forward * 200f + Vector3.Up * 100f;

        DropIntoWorld( eye.Position + eye.Forward * 48f, velocity, player.Network.Owner );
    }
    else
    {
        DropIntoWorld( WorldPosition, Vector3.Up * 50f );
    }

    return true;
}
```

`DropIntoWorld` 只在主机执行（`Networking.IsHost`），核心动作：解除网络归属、脱父、恢复 prefab 缩放、放到指定位置、加上 `removable` 标签、把速度交给 `Rigidbody`。`IsDropped` 是 `[Sync(FromHost)]`，所有端会同步切换"掉落物理"（见 [`BaseSandboxWeapon.WorldModel.cs`](../Code/Game/Weapon/BaseSandboxWeapon/BaseSandboxWeapon.WorldModel.cs) 的 `ApplyDroppedState`）。

还有一个引擎行为的小坑，源码注释写得很清楚：

```csharp
protected override void OnHolstered()
{
    base.OnHolstered();
    _ = EnableIfDroppedAsync();
}

// 引擎会禁用"被收起来"的武器；丢弃时先在主机重新启用，但客户端只看到最终状态（enabled），
// 于是客户端手里的掉落武器会一直是禁用（不可见）的。这里轮询 1 秒做兜底修正。
private async Task EnableIfDroppedAsync()
{
    for ( var waited = 0f; waited < 1f; waited += 0.05f )
    {
        await GameTask.DelaySeconds( 0.05f );

        if ( !this.IsValid() ) return;

        if ( !Inventory.IsValid() )
        {
            GameObject.Enabled = true;
            return;
        }
    }
}
```

### 2.5 摄像机钩子

武器可以在相机合成阶段插一脚：

```csharp
public virtual void OnCameraSetup( Player player, Sandbox.CameraComponent camera ) { }
public virtual void OnCameraMove( Player player, ref Angles angles ) { }
```

- `OnCameraSetup`：在相机参数（FOV 等）确定后修改，典型用途是瞄准变焦（见 `SniperWeapon.ModifyCamera`）。
- `OnCameraMove`：直接改写玩家的视角角度，典型用途是工具枪 / 相机枪吸收鼠标。

视模的放置也发生在相机链里：

```csharp
CameraView _viewModelView;   // 相机骨骼偏移前的快照

protected override void ModifyCamera( CameraComponent camera, ref CameraView view )
{
    base.ModifyCamera( camera, ref view );

    _viewModelView = view;   // 先存快照

    // 让视模动画里的 camera 骨骼去移动相机（而不是移动枪）
    ViewModel?.GetComponentInChildren<ViewModel>()?.UpdateCameraBone( ref view );
}

protected override void PlaceViewModel( CameraComponent camera, in CameraView view )
{
    var model = ViewModel?.GetComponentInChildren<ViewModel>();
    if ( !model.IsValid() ) return;

    var placed = _viewModelView;                            // 用快照放置视模
    placed.Rotation = LookAtCursor( placed.Rotation );      // 检查菜单打开时转向鼠标

    model.Place( placed );
    model.FollowCameraEffects( camera, view );              // 再叠加纯渲染的震屏
}
```

### 2.6 HUD

```csharp
public virtual bool WantsHideHud => false;
public virtual HudElement HiddenHudElements => HudElement.All;

public override void DrawHud( Painter painter, Vector2 crosshair )
{
    if ( Hud.IsVisible( HudElement.Crosshair ) )
        DrawCrosshair( painter, crosshair );   // 具体武器重写这个
}
```

相机枪会 `WantsHideHud => true` 并隐藏除背包外的一切；狙击枪开镜时干脆不画准星。

### 2.7 partial 文件

| 文件 | 内容 |
|---|---|
| `BaseSandboxWeapon.cs` | 本文大部分内容 |
| `.Kick.cs` | `KickView` 与后坐力参数（第 4 节） |
| `.Reloading.cs` | 装填取消窗口（第 3 节） |
| `.ViewModel.cs` | 一句说明：视模生命周期由引擎管理 |
| `.WorldModel.cs` | 世界模型创建 / 销毁、`IsDropped` 同步 |

---

## 3. 背包、弹药与死亡掉落

### 3.1 PlayerInventory 概览

```csharp
public sealed partial class PlayerInventory : BaseInventoryComponent, Local.IPlayerEvents
{
    // 所有武器（按槽位排序），引擎 Items 的收窄版本
    public IEnumerable<BaseSandboxWeapon> Weapons => Items.OfType<BaseSandboxWeapon>();

    // 当前激活武器
    public BaseSandboxWeapon ActiveWeapon => ActiveItem as BaseSandboxWeapon;

    // 找某类武器中槽位最好的那个
    public T GetWeapon<T>() where T : BaseSandboxWeapon { ... }
}
```

引擎 `BaseInventoryComponent` 负责真正的槽位 / 切换 / 拾取逻辑，游戏层加的是：可取消的事件、弹药合并提示、自动切换、换槽事件。

### 3.2 事件化的拾取 / 丢弃 / 移除 / 换槽

```csharp
protected override bool OnAdding( Sandbox.BaseInventoryItem item, int slot )
{
    if ( item is not BaseSandboxWeapon weapon )
        return true;

    var pickupEvent = new PlayerPickupEvent { Player = Player, Weapon = weapon, Slot = slot };
    Local.IPlayerEvents.PostToGameObject( Player.GameObject, e => e.OnPickup( pickupEvent ) );
    Global.IPlayerEvents.Post( e => e.OnPlayerPickup( pickupEvent ) );

    return !pickupEvent.Cancelled;   // 返回 false 可以拒绝这次拾取
}
```

四个钩子对应四个可取消事件（定义在 [`Code/Player/PlayerEvent.cs`](../Code/Player/PlayerEvent.cs)）：

| 钩子 | 事件 | 触发时机 |
|---|---|---|
| `OnAdding` | `PlayerPickupEvent` | 引擎把武器加进背包之前 |
| `OnDropping` | `PlayerDropEvent` | 丢弃之前 |
| `OnRemoving` | `PlayerRemoveWeaponEvent` | 移除（销毁）之前 |
| `OnMovingSlot` | `PlayerMoveSlotEvent` | 换槽之前 |

另有 `CanSwitchTo` → `PlayerSwitchWeaponEvent`（切换武器前，见下文 `SwitchWeapon`）。`PlayerLoadout` 也监听这些事件来保存热键栏（见 3.5）。

### 3.3 自动切换

```csharp
private bool ShouldAutoswitchTo( BaseSandboxWeapon item, bool given = false )
{
    if ( !ActiveWeapon.IsValid() )
        return true;

    if ( !GamePreferences.AutoSwitch )
        return false;

    if ( ActiveWeapon.IsInUse() )     // 正在拉栓 / 烹饪手雷等，别打断
        return false;

    if ( given )
        return true;                  // 明确给予（生成菜单 / 给予）跳过价值比较

    if ( item.ShouldAvoid )           // 空的枪
        return false;

    return item.Value > ActiveWeapon.Value;   // 只捡"更好"的枪
}
```

`given` 来自 `Pickup(prefab, notice)` 这条"主动给予"路径，`SwitchWeapon` 则会先问 `CanSwitchTo`（含 `PlayerSwitchWeaponEvent`）再调用引擎的 `Switch`。

### 3.4 弹药合并与拾取提示

重复武器会被引擎转化为"弹药捐赠"。[`PlayerInventory.Ammo.cs`](../Code/Player/PlayerInventory.Ammo.cs) 在主机上监听弹药增量，并把提示发给拥有者：

```csharp
public new int GiveAmmo( BaseAmmoResource type, int amount )
{
    var added = base.GiveAmmo( type, amount );
    NotifyAmmoPickup( type, added );   // 只通知真正被接受的部分
    return added;
}

[Rpc.Owner]
private void ReceiveAmmoPickup( string resourcePath, int amount )
{
    // 本地玩家才显示；同类提示在 1.5 秒内合并数量，最多同时 4 条
}
```

### 3.5 PlayerLoadout：死亡不是终点

[`PlayerLoadout.cs`](../Code/Player/PlayerLoadout.cs) 挂在玩家身上，负责记住"玩家手里有哪些枪"，并在重生时恢复：

```csharp
public string SerializeLoadout()
{
    var entries = Inventory.Weapons
        .Where( w => !string.IsNullOrEmpty( w.GameObject.PrefabInstanceSource ) )
        .Select( w => new LoadoutEntry { PrefabPath = w.GameObject.PrefabInstanceSource } )
        .ToList();

    return entries.Count > 0 ? Json.Serialize( entries ) : null;
}
```

重生流程（主机）：

```csharp
void Global.IPlayerEvents.OnPlayerSpawned( Player player )
{
    if ( player != Player ) return;
    if ( !Networking.IsHost ) return;

    _ = RestoreOnSpawnAsync();
}

private async Task RestoreOnSpawnAsync()
{
    if ( !Player.IsLocalPlayer )
    {
        RequestClientLoadout();   // 主机向客户端要 hotbar
        return;
    }

    var json = GetSpawnLoadout();   // ServerSettings.ReEquipLast ? LocalData["hotbar"] : null
    if ( !string.IsNullOrEmpty( json ) )
    {
        await ReplaceLoadoutAsync( json );
        return;
    }

    GiveStarterItemsIfEmpty();      // 没有存档就发给初始物品
    var best = Inventory.GetBestWeapon();
    if ( best.IsValid() )
        Inventory.SwitchWeapon( best );
}
```

要点：

- 只有服务器打开 `ReEquipLast` 才会恢复上一次的热键栏，否则发给玩家预设的初始物品（`player.prefab` 里的 `StartingItems`）。
- 热键栏存在 **拥有者的本地**（`LocalData`），主机通过 `[Rpc.Owner]` 向客户端索取；主机自己的那份直接本地读。
- 死亡、拾取、丢弃、移除、换槽都会触发保存。
- 存档（`SaveSystem`）里以 `Loadout_{steamId}` 作为 metadata 存一份，读档后恢复。

### 3.6 座位收枪

[`PlayerInventory.Seat.cs`](../Code/Player/PlayerInventory.Seat.cs)：玩家坐上座位默认把武器收起来，起身再拿回来。

```csharp
protected override void OnFixedUpdate()
{
    base.OnFixedUpdate();

    if ( !Networking.IsHost ) return;

    var seat = GetComponentInParent<ISitTarget>( false );
    if ( seat == _seat ) return;

    var previous = _seat;
    _seat = seat;

    if ( previous is Component left && left.IsValid() )
        PostControlStopped( left );

    if ( seat is Component sat )
        PostControlStarted( sat );

    // 离开座位：恢复坐下前那把枪
    if ( IsSeatHolstered )
    {
        var weapon = _weaponBeforeSeat;
        _weaponBeforeSeat = null;
        IsSeatHolstered = false;

        if ( weapon.IsValid() && weapon.Inventory == this )
            Switch( weapon );
    }

    if ( seat is not null )
        HolsterForSeat( seat );
}
```

`HolsterForSeat` 会先发 `PlayerSeatHolsterEvent`（可取消），取消后玩家坐在座位上也能保留武器。`IsSeatHolstered` 是 `[Sync(FromHost)]`，也是 2.3 里 `OnCanSwitchTo` 的判断依据。

---

## 4. 后坐力与布娃娃冲击

### 4.1 KickView：三层后坐力

[`BaseSandboxWeapon.Kick.cs`](../Code/Game/Weapon/BaseSandboxWeapon/BaseSandboxWeapon.Kick.cs)：

```csharp
// 瞄准后坐（所有端都生效）：每次攻击随机抬高 / 偏移瞄准角
[Property, Group( "Recoil" )] public Vector2 RecoilPitch { get; set; } = new( -0.3f, -0.1f );
[Property, Group( "Recoil" )] public Vector2 RecoilYaw { get; set; } = new( -0.1f, 0.1f );

// 视图冲击（只有本地第一人称）：相机 punch + 震屏
[Property, Group( "Camera" )] public Vector2 PunchPitch { get; set; } = new( 0f, 0f );
[Property, Group( "Camera" )] public Vector2 PunchYaw { get; set; } = new( 0f, 0f );
[Property, Group( "Camera" )] public Vector2 PunchRoll { get; set; } = new( 0.5f, 0.5f );
[Property, Group( "Camera" )] public float PunchFrequency { get; set; } = 2f;
[Property, Group( "Camera" )] public float PunchDuration { get; set; } = 0.3f;
[Property, Group( "Camera" )] public float ShakeAmplitude { get; set; } = 0f;
[Property, Group( "Camera" )] public float ShakeFrequency { get; set; } = 40f;
[Property, Group( "Camera" )] public float ShakeDuration { get; set; } = 0.3f;

protected bool KickView()
{
    if ( !HasOwner ) return false;

    // 1. 真实的瞄准后坐：直接加到玩家 EyeAngles（会同步到主机）
    Owner.Controller.EyeAngles += new Angles(
        Random.Shared.Float( RecoilPitch.x, RecoilPitch.y ),
        Random.Shared.Float( RecoilYaw.x, RecoilYaw.y ), 0 );

    // 2. 只有本地玩家第一人称才需要纯视觉的 punch / shake
    if ( Owner.Controller.ThirdPerson || !Owner.IsLocalPlayer ) return false;

    var punch = new Angles(
        Random.Shared.Float( PunchPitch.x, PunchPitch.y ),
        Random.Shared.Float( PunchYaw.x, PunchYaw.y ),
        Random.Shared.Float( PunchRoll.x, PunchRoll.y ) );
    Scene.Camera?.AddPunch( punch, PunchFrequency, PunchDuration );

    if ( ShakeAmplitude > 0f )
        Scene.Camera?.AddShake( ShakeAmplitude, ShakeFrequency, ShakeDuration );

    return true;
}
```

注意 `KickView` 只在 **有主人** 时生效。独立放在地上的枪走另一条路，见 4.2。

### 4.2 独立武器的物理后坐

[`BaseBulletWeapon.DoRecoil`](../Code/Game/Weapon/BaseBulletWeapon/BaseBulletWeapon.cs)：

```csharp
protected virtual void DoRecoil()
{
    if ( !HasOwner )
    {
        // 没人拿的枪：用 Rigidbody 沿枪口反方向推一把
        if ( ShootForce > 0f && GetComponent<Rigidbody>( true ) is { } rb )
        {
            var muzzle = WeaponModel?.MuzzleGameObject?.WorldTransform ?? WorldTransform;
            rb.ApplyForce( muzzle.Rotation.Up * ShootForce );
        }

        return;
    }

    KickView();   // 有人拿：走视图后坐
}
```

### 4.3 枪声让 NPC 听见

```csharp
void EmitGunfireNoise( ShotEffect shot )
{
    // 只有主机广播；0.25 秒冷却避免全自动糊满 stimulus
    if ( !Networking.IsHost || shot.NoEvents || _timeSinceNoise < 0.25f ) return;

    var owner = Owner;
    if ( !owner.IsValid() ) return;

    _timeSinceNoise = 0;
    Scene.GetSystem<NpcStimulusSystem>()?.Broadcast(
        StimulusKind.Gunshot, owner.WorldPosition, owner.GameObject,
        radius: 3000f, lifetime: 1.5f );
}
```

### 4.4 布娃娃冲击

```csharp
void PushRagdoll( GameObject hitObject, Vector3 hitPosition, Vector3 origin )
{
    if ( !hitObject.IsValid() ) return;

    var struck = hitObject.Components.GetInAncestorsOrSelf<Rigidbody>();
    if ( !struck.IsValid() || struck.IsProxy ) return;

    var physics = hitObject.Components.GetInAncestorsOrSelf<ModelPhysics>();
    if ( !physics.IsValid() || physics.Bodies.Count < 2 ) return;

    // 命中骨骼吃满速度变化，其他骨骼分 RagdollBodyShare 的比例
    var speed = Ballistics.Force * RagdollImpactScale;
    if ( speed <= 0f ) return;

    var dir = (hitPosition - origin).Normal;

    foreach ( var body in physics.Bodies )
    {
        var rb = body.Component;
        if ( !rb.IsValid() ) continue;

        if ( rb == struck )
            rb.ApplyImpulseAt( hitPosition, dir * speed * rb.Mass );
        else if ( RagdollBodyShare > 0f )
            rb.ApplyImpulse( dir * speed * RagdollBodyShare * rb.Mass );
    }
}
```

关键设计：源码注释写明"每个端各自跑布娃娃"，所以这个冲击要在 **每个端** 执行（`OnShootEffects` 在所有端运行），而不是只主机。

---

## 5. 子弹枪 BaseBulletWeapon

```csharp
public partial class BaseBulletWeapon : BaseSandboxWeapon
{
    // 弹道（伤害、弹丸数、散布、射程）和开火本身在引擎 BaseCombatWeapon；
    // 这里只补：后坐力、枪声、布娃娃冲击、子弹碰撞规则。

    public override void PrimaryAttack()
    {
        base.PrimaryAttack();   // 引擎：消耗弹药、弹道、生成 ShotEffect
        DoRecoil();
    }

    protected override void OnShootEffects( ShotEffect shot )
    {
        base.OnShootEffects( shot );   // 枪口特效、弹壳、声音、命中效果

        EmitGunfireNoise( shot );      // 通知 NPC

        if ( shot.Hit )
            PushRagdoll( shot.HitObject, shot.HitPosition, shot.Origin ?? GetMuzzleTransform().Position );
    }
}
```

### 子弹碰撞规则

```csharp
protected override SceneTrace BulletTrace( Ray ray, float distance, float radius )
{
    return Scene.Trace.Ray( ray, distance )
        .IgnoreGameObjectHierarchy( AimIgnoreRoot )   // 忽略开枪者一伙
        .WithCollisionRules( "bullet" )               // 用沙盒的子弹碰撞层
        .WithoutTags( "playercontroller" )            // 玩家靠 hitbox 命中，不走控制器碰撞体
        .Radius( radius )
        .UseHitboxes();                               // 命中玩家的 hitbox（爆头判定）
}
```

`AimIgnoreRoot` 在持枪时是玩家的 GameObject，独立时是武器自己。命中 hitbox 会把命中分组（hitgroup）带给伤害系统——NPC 的 `OnDamage` 里爆头 ×2（见 [`Code/Npcs/Npc.cs`](../Code/Npcs/Npc.cs)）。

### 装填取消窗口

[`BaseSandboxWeapon.Reloading.cs`](../Code/Game/Weapon/BaseSandboxWeapon/BaseSandboxWeapon.Reloading.cs) 解决一个细节：装填开始后的一段时间内，扣扳机可以取消装填；过了这个窗口就不能了。

```csharp
[Property, Feature( "Reloading" ), ShowIf( nameof( CanCancelReload ), true )]
public float ReloadCancelWindow { get; set; } = 0f;

bool _inControl;

public bool ReloadCommitted => IsReloading && ReloadCancelWindow > 0f && _timeSinceReloadStart > ReloadCancelWindow;

public override void CancelReload()
{
    if ( _inControl && ReloadCommitted ) return;   // 引擎在 OnControl 里发起的取消也要被拦住

    base.CancelReload();
}
```

---

## 6. 近战 MeleeWeapon

```csharp
public class MeleeWeapon : BaseSandboxWeapon
{
    [Property] public float SwingDelay { get; set; } = 0.5f;       // 命中后的冷却
    [Property] public float MissSwingDelay { get; set; } = 0.75f;   // 挥空的冷却
    [Property] public float Damage { get; set; } = 12f;
    [Property] public float Range { get; set; } = 128f;
    [Property] public float SwingRadius { get; set; } = 10f;
    [Property] public float SwingForce { get; set; } = 1000f;
    [Property] public SoundEvent HitSound { get; set; }

    public MeleeWeapon()
    {
        UsesAmmo = false;   // 近战不用弹药
    }

    public bool CanAttack() => NextPrimaryFire <= 0;
    public override bool CanSecondaryAttack() => false;

    public override void PrimaryAttack()
    {
        var forward = AimRay.Forward;

        // 先细射线，没打中再用半径放宽一点
        var trace = Scene.Trace.Ray( AimRay with { Forward = forward }, Range )
                            .IgnoreGameObjectHierarchy( AimIgnoreRoot )
                            .WithoutTags( "playercontroller" )
                            .UseHitboxes();

        var tr = trace.Run();
        if ( !tr.Hit )
            tr = trace.Radius( SwingRadius ).Run();

        // 命中 / 挥空用不同冷却
        SetNextPrimaryFire( tr.GameObject.IsValid() ? SwingDelay : MissSwingDelay );

        // 挥击表现（本地预测 + 主机转发）
        SwingEffects( new ShotEffect( tr.HitPosition, tr.Hit, tr.Normal, tr.GameObject, tr.Surface ) );

        // 我们的 trace 决定命中；引擎把它"认领"给主机，由主机结算伤害
        ShootBullet( tr, Damage, SwingForce, hitboxTags: true );

        KickView();
    }
}
```

`CrowbarWeapon` 完全是空的（只有 `[Title]`），所有参数由 prefab 配置；`FistsWeapon` 则重写了攻击来支持左右手：

```csharp
public override bool CanSecondaryAttack() => NextSecondaryFire <= 0 && CanPrimaryAttack();

public override void PrimaryAttack() => Punch( false );
public override void SecondaryAttack() => Punch( true );

private void Punch( bool rightHand )
{
    _rightHand = rightHand;
    base.PrimaryAttack();
    SetNextSecondaryFire( NextPrimaryFire );   // 两只手共享冷却
}
```

拳头还有两个特殊点：`OnDrop() => false`（没有可掉落的实体），以及用 `Rpc.FilterExclude` 把拳头动画中继给其他玩家（详见 [`FistsWeapon.cs`](../Code/Weapons/FistsWeapon.cs)）。

---

## 7. 机瞄 IronSightsWeapon

```csharp
public abstract class IronSightsWeapon : BaseBulletWeapon
{
    [Property] public float IronSightsFireScale { get; set; } = 0.2f;    // 瞄准时散布 ×0.2
    [Property] public float AimFovScale { get; set; } = 0.8f;            // 瞄准时 FOV ×0.8
    [Property] public float AimFovTransitionTime { get; set; } = 0.15f;  // FOV 过渡时间

    // 瞄准状态是 owner-authoritative 的：同步给主机，主机的权威弹道也才算得准
    [Sync] public bool IsAiming { get; set; }

    protected override void OnControl()
    {
        base.OnControl();

        var wantsAim = Input.Down( "attack2" ) && !IsReloading;
        if ( wantsAim == IsAiming ) return;

        IsAiming = wantsAim;
        ViewModel?.RunEvent<ViewModel>( x =>
        {
            x.Renderer?.Set( "ironsights", IsAiming ? 1 : 0 );
            x.Renderer?.Set( "ironsights_fire_scale", IsAiming ? IronSightsFireScale : 1f );
        } );
    }

    protected override void ModifyCamera( CameraComponent camera, ref CameraView view )
    {
        if ( HasOwner && Owner.Controller.IsValid() && !Owner.Controller.ThirdPerson )
        {
            // 平滑混合 FOV
            _aimFovBlend = AimFovTransitionTime <= 0f
                ? (IsAiming ? 1f : 0f)
                : (_aimFovBlend + (IsAiming ? 1f : -1f) * Time.Delta / AimFovTransitionTime).Clamp( 0f, 1f );

            view.FieldOfView *= MathX.Lerp( 1f, AimFovScale.Clamp( 0.1f, 1f ), _aimFovBlend );
        }
        else
        {
            _aimFovBlend = 0f;
        }

        base.ModifyCamera( camera, ref view );
    }

    public override Vector2 CurrentSpread => base.CurrentSpread * (IsAiming ? IronSightsFireScale : 1f);
}
```

设计意图（源码注释）：如果 `IsAiming` 不同步，瞄准只会在拥有者的预测弹道里收窄，主机真正算伤害的那次射击仍是宽的。

---

## 8. 第一人称视模 ViewModel

`ViewModel` 继承引擎 `Sandbox.BaseWeaponModel`，生命周期（创建 / 销毁 / 放置）由引擎 `BaseCombatWeapon` 管理，游戏层只负责每帧的动画与摆动。

```csharp
public sealed partial class ViewModel : Sandbox.BaseWeaponModel
{
    [ConVar( "sandbox.hideviewmodel", ConVarFlags.Cheat )]
    private static bool HideViewModel { get; set; } = false;
}
```

### 8.1 放置链

```csharp
// 1. 相机合成时：让动画里的 "camera" 骨骼去移动相机
public void UpdateCameraBone( ref CameraView view )
{
    if ( !Renderer.IsValid() ) return;

    Renderer.Enabled = !HideViewModel;

    if ( !GamePreferences.ViewBobbing )
    {
        currentInertia = Vector2.Zero;
        isFirstUpdate = true;
        return;
    }

    ApplyInertia( view.Rotation );

    if ( Renderer.TryGetBoneTransformLocal( "camera", out var bone ) )
    {
        var scale = 0.5f;
        view.Position += view.Rotation * bone.Position * scale;
        view.Rotation *= bone.Rotation * scale;
    }
}

// 2. 放置视模：用骨骼偏移前的快照，这样相机骨骼是"相机绕着枪转"
public void Place( in CameraView view )
{
    var rotation = view.Rotation;
    var pivotOffset = Vector3.Zero;

    if ( LookInertia && GamePreferences.ViewBobbing )
        rotation = ApplyLookInertia( view.Rotation, out pivotOffset );
    else
    {
        _lookInertiaFirstUpdate = true;
        _lookVelocity = Vector2.Zero;
    }

    WorldPosition = view.Position + pivotOffset;
    WorldRotation = rotation;
}

// 3. 叠加纯渲染的震屏（不喂给 look inertia）
public void FollowCameraEffects( CameraComponent camera, in CameraView view )
{
    var effects = CameraEffectSystem.Get( Scene );
    if ( effects is null ) return;

    effects.QueryOffsets( camera, out var offset, out var angles, out _ );
    var renderTransform = new Transform( view.Position + view.Rotation * offset, view.Rotation * angles );
    WorldTransform = renderTransform.ToWorld( view.Transform.ToLocal( WorldTransform ) );
}
```

### 8.2 视图惯性 LookInertia

```csharp
[Property, Group( "Inertia" )] public bool LookInertia { get; set; } = false;
[Property, Group( "Inertia" )] public float LookInertiaFrequency { get; set; } = 3.0f;     // 弹簧硬度
[Property, Group( "Inertia" )] public float LookInertiaDamping { get; set; } = 0.35f;      // 阻尼（<1 会晃）
[Property, Group( "Inertia" )] public float LookInertiaPivotDistance { get; set; } = 10f; // 以眼前 10 单位为支点
[Property, Group( "Inertia" )] public float LookInertiaAmount { get; set; } = 1.0f;        // 整体强度
[Property, Title( "ADS Inertia Scale" ), Group( "Inertia" )] public float ADSInertiaScale { get; set; } = 0.3f;

Rotation ApplyLookInertia( Rotation rotation, out Vector3 pivotOffset )
{
    var angles = rotation.Angles();
    var target = new Vector2( angles.pitch, angles.yaw );

    if ( _lookInertiaFirstUpdate )
    {
        _lookAngles = target;
        _lookVelocity = Vector2.Zero;
        _lookInertiaFirstUpdate = false;
    }
    else
    {
        // 走最短路径过 -180/180 接缝，避免弹簧被拉飞
        target.y = _lookAngles.y + Angles.NormalizeAngle( target.y - _lookAngles.y );
    }

    _lookAngles = Vector2.SpringDamp( _lookAngles, target, ref _lookVelocity, Time.Delta, LookInertiaFrequency, LookInertiaDamping );

    var laggedFull = new Angles( _lookAngles.x, _lookAngles.y, angles.roll ).ToRotation();

    var amount = LookInertiaAmount;

    // 机瞄时进一步压低惯性
    var ironSights = GetComponentInParent<IronSightsWeapon>();
    if ( ironSights.IsValid() && ironSights.IsAiming )
        amount *= ADSInertiaScale;

    var delta = laggedFull * rotation.Inverse;
    var lagged = Rotation.Slerp( Rotation.Identity, delta, amount, clamp: false ) * rotation;

    // 以"眼前一段距离"为支点，让同样的角度差产生更大的可见摆动
    var pivotLocal = Vector3.Forward * LookInertiaPivotDistance;
    pivotOffset = (rotation * pivotLocal) - (lagged * pivotLocal);

    return lagged;
}
```

### 8.3 动画参数

```csharp
void UpdateAnimation()
{
    var playerController = GetComponentInParent<PlayerController>();
    if ( !playerController.IsValid() ) return;

    var rot = playerController.EyeAngles;   // 用眼睛角度，不是相机（相机此时还没合成完）

    Renderer.Set( "b_twohanded", true );
    Renderer.Set( "deploy_type", 1 );
    Renderer.Set( "reload_type", 1 );
    Renderer.Set( "b_grounded", playerController.IsOnGround );
    Renderer.Set( "move_bob", GamePreferences.ViewBobbing ? playerController.Velocity.Length.Remap( 0, playerController.RunSpeed * 2f ) : 0 );

    Renderer.Set( "aim_pitch", rot.pitch );
    Renderer.Set( "aim_pitch_inertia", GamePreferences.ViewBobbing ? currentInertia.x * InertiaScale.x : 0 );
    Renderer.Set( "aim_yaw", rot.yaw );
    Renderer.Set( "aim_yaw_inertia", GamePreferences.ViewBobbing ? currentInertia.y * InertiaScale.y : 0 );

    Renderer.Set( "attack_hold", IsAttacking ? AttackDuration.Relative.Clamp( 0f, 1f ) : 0f );

    // 移动方向 / 速度（move_direction / move_speed / move_groundspeed / move_x / move_y / move_z），
    // 供动画图混合，这里省略。
}
```

这些参数（`b_twohanded`、`move_bob`、`aim_pitch_inertia`…）都是动画图里读取的命名参数，具体含义以模型 / 动画图为准。

### 8.4 装填音效

```csharp
public record struct ReloadSoundEntry
{
    [KeyProperty] public float Time { get; set; }      // 装填开始后多少秒播放
    [KeyProperty] public SoundEvent Sound { get; set; }
}

[Property, Group( "Reload Sounds" )] public List<ReloadSoundEntry> ReloadSoundEvents { get; set; } = new();
[Property, Group( "Reload Sounds" )] public List<ReloadSoundEntry> IncrementalReloadSoundEvents { get; set; } = new();
[Property, Group( "Reload Sounds" )] public List<ReloadSoundEntry> IncrementalReloadStartSounds { get; set; } = new();
[Property, Group( "Reload Sounds" )] public List<ReloadSoundEntry> IncrementalReloadFinishSounds { get; set; } = new();
```

生命周期回调由引擎触发：`OnReloadStart` / `OnIncrementalReload` / `OnReloadFinish` / `OnReloadCancel`。音效用 `CancellationTokenSource` 按时间轴异步播放，装填取消时统一取消。

### 8.5 攻击与投掷

```csharp
public override void OnAttack( Vector3? hitPoint = null, Vector3? origin = null )
{
    base.OnAttack( hitPoint, origin );

    // 第一人称枪口闪光要跟随枪，否则后坐 / 摆动时会"脱钩"
    if ( MuzzleGameObject.IsValid() )
    {
        foreach ( var effect in MuzzleGameObject.GetComponentsInChildren<ParticleEffect>( true ) )
            effect.LocalSpace = 1.0f;
    }

    if ( IsThrowable )
    {
        Renderer?.Set( "b_throw", true );
        Invoke( 0.5f, () => { Renderer?.Set( "b_deploy_new", true ); Renderer?.Set( "b_pull", false ); } );
    }
}
```

`IsThrowable` / `ThrowableType` 在 [`ViewModel.Throwables.cs`](../Code/Game/Weapon/ViewModel/ViewModel.Throwables.cs) 里定义。

---

## 9. 投掷物与爆炸

### 9.1 ProjectileEntity

```csharp
public class ProjectileEntity : Component, Component.ICollisionListener
{
    [RequireComponent] public Rigidbody Rigidbody { get; set; }

    // 谁扔的（伤害归属）
    [Sync( SyncFlags.FromHost )] public Player Instigator { get; set; }

    void ICollisionListener.OnCollisionStart( Collision collision )
    {
        if ( IsProxy ) return;   // 只有权威端处理

        // 刚出手时无视自己
        var player = collision.Other.GameObject.GetComponentInParent<Player>();
        if ( Instigator.IsValid() && player.IsValid() && player == Instigator )
            return;

        OnHit( collision );
    }

    protected virtual void OnHit( Collision collision = default )
    {
        GameObject.Destroy();   // 默认：撞就消失
    }

    protected void Explode( float radius, float damage, float forceScale )
    {
        var explosion = Explosion.Spawn( new ExplosionEvent
        {
            Source = GameObject,
            Attacker = Instigator?.GameObject,
            Position = WorldPosition,
            Rotation = WorldRotation,
            Radius = radius,
            Damage = damage,
            ForceScale = forceScale
        } );

        if ( explosion.IsValid() )
            CreateScorchDecal();   // 在命中表面留焦痕

        GameObject.Destroy();
    }

    public void UpdateDirection( Vector3 direction, float speed )
    {
        WorldRotation = Rotation.LookAt( direction, Vector3.Up );
        Rigidbody.Velocity = WorldRotation.Forward * speed;
    }
}
```

### 9.2 爆炸管线

```csharp
public static class Explosion
{
    public const string DefaultPrefab = "/prefabs/engine/explosion_med.prefab";

    public static GameObject Spawn( ExplosionEvent e )
    {
        // 1. 先让监听者改参数 / 取消
        scene.RunEvent<Global.IExplosionEvents>( x => x.OnExplode( e ) );
        if ( e.Cancelled ) return null;

        // 2. 克隆爆炸 prefab，把半径 / 伤害 / 冲击力写进 RadiusDamage
        var go = GameObject.Clone( e.Prefab, new CloneConfig { Transform = new Transform( e.Position, e.Rotation ), StartEnabled = false } );
        ...
        go.RunEvent<RadiusDamage>( x =>
        {
            x.Radius = e.Radius;
            x.PhysicsForceScale = e.ForceScale;
            x.DamageAmount = e.Damage;
            x.Attacker = e.Attacker.IsValid() ? e.Attacker : go;
            x.DamageTags ??= new();
            x.DamageTags.Add( DamageTags.Explosion );
        }, FindMode.EverythingInSelfAndDescendants );

        // 3. 网络生成
        go.Enabled = true;
        go.NetworkSpawn( true, null );

        scene.RunEvent<Global.IExplosionEvents>( x => x.OnExploded( e, go ) );
        return go;
    }
}
```

爆炸 prefab 本身携带特效、声音和 `RadiusDamage`（引擎组件）。监听 `Global.IExplosionEvents` 可以做安全区、爆炸增幅等玩法。

[`TimedExplosive`](../Code/Weapons/HandGrenade/TimedExplosive.cs) 是"到时间就炸"的通用组件（手雷、地雷都用），支持被枪击 / 撞击引爆（忽略 `Crush` / `Impact` 标签）。

### 9.3 手雷 HandGrenadeWeapon

手雷是一个状态机：按住开火 → 拉环开始烹饪；松开 → 扔出去；煮太久 → 在手里爆炸。

```csharp
[Sync] TimeSince TimeSinceCooked { get; set; }
[Sync] bool IsCooking { get; set; }
[Sync] bool IsThrowing { get; set; }
[Sync] TimeUntil TimeUntilThrown { get; set; }

protected override void OnControl()
{
    var player = Owner;

    // 投掷动画期间：等动画结束再发下一颗
    if ( IsThrowing )
    {
        if ( TimeUntilThrown )
        {
            IsThrowing = false;

            if ( !HasPrimaryAmmo() )
            {
                Inventory?.SwitchToBest();
                DestroyGameObject();
                return;
            }

            // 部署下一颗
            if ( DeploySound is not null )
                GameObject.PlaySound( DeploySound );

            WeaponModel?.Renderer?.Set( "b_reload", true );
        }

        return;
    }

    // 按下攻击键：开始烹饪
    if ( !IsCooking && CanPrimaryAttack() && (Input.Pressed( "Attack1" ) || Input.Pressed( "Attack2" )) )
    {
        IsCooking = true;
        TimeSinceCooked = 0;

        if ( PinPullSound is not null )
            GameObject.PlaySound( PinPullSound );

        WeaponModel?.Renderer?.Set( "b_charge", true );
        WeaponModel?.Renderer?.Set( "charge_type", 0 );
    }

    if ( !IsCooking )
        return;

    UpdateThrowType();   // 远近投掷混合

    // 煮太久：在手里爆炸
    if ( TimeSinceCooked > Lifetime )
    {
        IsCooking = false;
        ExplodeInHand();
        return;
    }

    // 两个键都松开：扔出去
    if ( !Input.Down( "Attack1" ) && !Input.Down( "Attack2" ) )
    {
        Throw( player );
    }
}
```

三个主机 RPC 值得注意：

```csharp
[Rpc.Host]
void SpawnProjectile( Player player, Vector3 startPos, Vector3 direction, float powerScale )
{
    TakePrimaryAmmo( 1 );   // 弹药由主机扣（拥有者只是预测了投掷动作）

    var go = Prefab.Clone( startPos );

    var explosive = go.GetOrAddComponent<TimedExplosive>();
    if ( explosive.IsValid() )
    {
        explosive.Lifetime = MathF.Max( 0.1f, Lifetime - TimeSinceCooked );  // 剩余引信时间
        explosive.Radius = Radius;
        explosive.Damage = MaxDamage;
        explosive.Force = Force;
        explosive.Attacker = player.GameObject;
    }

    var rb = go.GetComponent<Rigidbody>();
    if ( rb.IsValid() )
    {
        var baseVelocity = player.GetComponent<PlayerController>().Velocity;
        rb.Velocity = baseVelocity + direction * (ThrowPower * powerScale) + Vector3.Up * 100f;
        rb.AngularVelocity = go.WorldRotation.Right * 10f;
    }

    // 别和扔出它的武器相撞
    var filter = go.AddComponent<PhysicsFilter>();
    filter.Body = GameObject;

    go.NetworkSpawn();
}
```

- `ExplodeInHand`：直接生成爆炸，扣一颗弹药，只有最后一颗才销毁武器。
- `DropGrenade`：死亡时把未扔的手雷原地丢下（`OnPlayerDeath`）。
- `Throw` 里 `CurrentThrowType` 决定近投 / 远投：近投更低、力度减半、方向略向上。

### 9.4 火箭筒 RpgWeapon

特点：右键切换"制导瞄准"，火箭持续追踪准星。

```csharp
[Property, Sync( SyncFlags.FromHost ), ClientEditable]
public bool IsTrackedAim { get; set; } = false;   // 主机权威

public override bool IsTargetedAim => IsTrackedAim;

[Sync( SyncFlags.FromHost )] RpgProjectile Projectile { get; set; }

public bool IsGuiding => IsTrackedAim && Projectile.IsValid();

public override void PrimaryAttack()
{
    if ( HasOwner && !TakePrimaryAmmo( 1 ) )
    {
        DryFire();
        return;
    }

    TimeSinceShoot = 0;
    SetNextFire( PrimaryDelay );

    ShootEffects();   // 本地预测

    var ray = AimRay;
    var muzzlePos = GetMuzzleTransform().Position;
    var spawnPos = muzzlePos + ray.Forward * 64f;

    if ( HasOwner )
    {
        spawnPos = CheckThrowPosition( Owner, muzzlePos, spawnPos );
        if ( KickView() )
            _hasFired = true;
    }

    CreateProjectile( spawnPos, ray.Forward, ProjectileSpeed );  // [Rpc.Host]
}

[Rpc.Host]
void CreateProjectile( Vector3 start, Vector3 direction, float speed )
{
    var go = ProjectilePrefab?.Clone( start );

    var projectile = go.GetComponent<RpgProjectile>();
    Assert.True( projectile.IsValid(), "RpgProjectile not on projectile prefab" );

    if ( Owner.IsValid() )
        projectile.Instigator = Owner;
    else if ( ControlContext.Player.IsValid() )
        projectile.Instigator = ControlContext.Player;

    go.NetworkSpawn();

    Projectile = projectile;
    projectile.UpdateDirection( direction, speed );
}

private Vector3 GetAimTarget()
{
    var ray = AimRay;
    var tr = Scene.Trace.Ray( ray, 16384f )
        .IgnoreGameObjectHierarchy( AimIgnoreRoot )
        .WithoutTags( "trigger", "projectile" )
        .Run();

    return tr.Hit ? tr.HitPosition : ray.Position + ray.Forward * 16384f;
}
```

制导在 `OnControl` 里每帧更新：`Projectile.UpdateWithTarget( GetAimTarget(), ProjectileSpeed )`。射击后松开开火键自动装填；制导中则等火箭消失后再装填。

### 9.5 其他值得一看的实现

- **Mp5 榴弹发射器**：`SecondaryAttack` 扣 `SecondaryAmmo` 并 `LaunchGrenade`（`[Rpc.Host]`），榴弹是 `ImpactGrenade`（撞击爆炸）；`OnAdded` 首次进背包时给 6 颗备弹。
- **SniperWeapon**：开镜状态机；开镜后延迟 0.2 秒隐藏视模（`Scene.Camera.RenderExcludeTags.Add( "sniper_scoped" )`），用 `SniperScopeEffect` 做镜片遮罩和鼠标 / 移动模糊；`ModifyCamera` 直接改 `view.FieldOfView`；`OnCameraMove` 用 `ScopeSensitivity` 缩小鼠标输入。
- **ScreenWeapon**：把视模的屏幕材质替换成一张可绘制的 `RenderTarget`，子类重写 `DrawScreenContent( Rect, Painter )` 画内容；工具枪用它显示模式 / 参数。
- **CameraWeapon**：`OnControl` 里 `attack1` 拍照（`Game.TakeScreenshot()`），`attack2` 调整变焦 / 翻滚，用 `DepthOfField` 做景深。

---

## 10. 具体武器与新增实践

### 10.1 武器清单

| 武器 | 类 | 基类 | 特点 |
|---|---|---|---|
| UP-45 | `GlockWeapon` | `IronSightsWeapon` | 自动手枪 |
| Heritage .45 | `Colt1911Weapon` | `IronSightsWeapon` | 半自动手枪 |
| Ironwood .357 | `RevolverWeapon` | `IronSightsWeapon` | 左轮 |
| AR-4 | `M4a1Weapon` | `IronSightsWeapon` | 突击步枪 |
| Spaghelli M4 | `ShotgunWeapon` | `IronSightsWeapon` | 霰弹枪，扩散环准星 |
| SG-5 | `Mp5Weapon` | `BaseBulletWeapon` | 冲锋枪 + 榴弹发射器 |
| M700 | `SniperWeapon` | `BaseBulletWeapon` | 开镜狙击 |
| Ashford R-7 | `RpgWeapon` | `BaseSandboxWeapon` | 制导火箭 |
| 手雷 | `HandGrenadeWeapon` | `BaseSandboxWeapon` | 烹饪手雷，可近投 |
| 撬棍 | `CrowbarWeapon` | `MeleeWeapon` | 空白近战模板 |
| 拳头 | `FistsWeapon` | `MeleeWeapon` | 左右手轮流挥拳 |
| 相机 | `CameraWeapon` | `BaseSandboxWeapon` | 拍照、变焦、景深 |
| 工具枪 | `Toolgun` | `ScreenWeapon` | 30+ 工具模式 |
| 物理枪 | `Physgun` | `ScreenWeapon` | 抓取 / 推拉 / 投掷物体 |

### 10.2 新增一把枪的最小步骤

1. **选基类**：普通枪 `BaseBulletWeapon`；能机瞄 `IronSightsWeapon`；近战 `MeleeWeapon`；特殊玩法直接 `BaseSandboxWeapon`。
2. **写 prefab**：武器 GameObject + 视模 / 世界模型 + 引擎 `Ballistics`（伤害、弹丸、散布、射程）+ 弹药配置。初始物品配在 [`Assets/prefabs/engine/player.prefab`](../Assets/prefabs/engine/player.prefab) 的 `StartingItems`。
3. **实现输入**：

   ```csharp
   protected override bool WantsPrimaryAttack() => Input.Pressed( "attack1" );
   ```

   自动武器用 `Input.Down`，半自动用 `Input.Pressed`。
4. **按需重写**：
   - `PrimaryAttack` / `SecondaryAttack`：自定义开火流程（参考 `RpgWeapon`）
   - `OnShootEffects`：自定义命中 / 特效（参考 `BaseBulletWeapon`）
   - `ModifyCamera` / `OnCameraMove` / `OnCameraSetup`：瞄准与镜头效果
   - `DrawCrosshair` / `DrawHud`：准星
   - `OnControl`：持续状态（参考 `HandGrenadeWeapon`、`SniperWeapon`）
5. **加元数据**：`[Title]`、`[Icon]`；需要本地化就用 `#key` 并往 [`Localization/`](../Localization/) 加文本。
6. **验证**：主机 / 客户端两边都试；注意预测表现与权威结算的边界（弹药、爆炸、投射物都以主机为准）。

---

## 11. 总结与阅读建议

### 心智模型

- **引擎负责"枪"**：弹道、弹药、装填、射击、视模 / 世界模型生命周期。
- **游戏层负责"沙盒"**：谁拿着、谁控制、掉到哪、死了怎么掉、视模怎么摆、枪声怎么被 NPC 听见。
- **主机权威**：弹药扣除、投射物生成、爆炸、掉落物生成都在主机；拥有者只做预测表现。
- **分层重写**：`BaseBulletWeapon` 只加"手感"，`IronSightsWeapon` 只加"瞄准"，`MeleeWeapon` 换成"挥砍"，`ScreenWeapon` 只加"屏幕"——具体武器基本只做配置和少量重写。

### 推荐阅读顺序

1. [`BaseSandboxWeapon.cs`](../Code/Game/Weapon/BaseSandboxWeapon/BaseSandboxWeapon.cs)：Owner / Attacker / 座位控制 / 丢弃
2. [`PlayerInventory.cs`](../Code/Player/PlayerInventory.cs) + [`PlayerLoadout.cs`](../Code/Player/PlayerLoadout.cs)：背包与死亡
3. [`BaseBulletWeapon.cs`](../Code/Game/Weapon/BaseBulletWeapon/BaseBulletWeapon.cs) + [`BaseSandboxWeapon.Kick.cs`](../Code/Game/Weapon/BaseSandboxWeapon/BaseSandboxWeapon.Kick.cs)：手感
4. [`ViewModel.cs`](../Code/Game/Weapon/ViewModel/ViewModel.cs)：视模
5. 挑一把具体武器（推荐 `GlockWeapon` → `HandGrenadeWeapon` → `RpgWeapon`）

### 术语速查

| 术语 | 含义 |
|---|---|
| `BaseCombatWeapon` | 引擎武器基类，负责机制 |
| `ShotEffect` | 一次射击的结果（命中点、命中对象、表面…） |
| `Ballistics` | 引擎弹道配置（伤害、弹丸、散布、力） |
| `AimRay` / `UnheldAimRay` | 瞄准射线；未被持有时的替代射线 |
| `KickView` | 持枪者的后坐力（瞄准 + 视图） |
| `IsAiming` | 机瞄状态，owner-authoritative |
| `LookInertia` | 视模随视角的弹簧滞后 |
| `TimedExplosive` | 定时爆炸物组件 |
| `ExplosionEvent` | 爆炸参数 + 可取消事件 |
| `IPlayerControllable` | 座位 / 机械控制武器的接口 |
| `PlayerLoadout` | 热键栏保存与恢复 |
