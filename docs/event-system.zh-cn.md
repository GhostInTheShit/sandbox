# 游戏开发基础 · 第 3 课：事件系统

> 本课只讲**一个知识点**：**事件 = 广播一件事，谁关心谁监听**。
> 这是"游戏功能逻辑设计"里最常用的工具，也是插件和 Mod 能插手官方功能的原因。
> 不需要网络知识，约 8 分钟读完。
> 前置：[第 1 课](game-loop-and-lifecycle.zh-cn.md)、[第 2 课](input-system.zh-cn.md)。

---

## 1. 为什么需要事件

设想"玩家捡到一把枪"。这一瞬间，可能同时需要：

- 把枪放进背包
- 播放拾取音效
- 在屏幕提示"按 X 切换到这把枪"
- 保存 hotbar（下次进游戏还记得）
- 更新 UI 上的武器图标
- 让服务器插件判断"这玩家能不能捡这把枪"

如果把这些全写进一个 `Pickup()` 函数，会变成几百行的"意大利面"，而且**别的功能想插手时只能改这个函数**。

事件的思路是：**在关键节点广播一件事，谁关心谁来听。**

```
玩家捡到枪
   ↓
广播："有人捡到枪了！"（PlayerPickupEvent）
   ↓
├─ 背包：收到 → 放进槽位
├─ 音效：收到 → 播音效
├─ UI：收到 → 显示提示
├─ 存档：收到 → 保存 hotbar
└─ 插件：收到 → 如果禁止，设置 Cancelled = true
```

广播的人**不需要知道有谁在听**；监听的人**也不需要知道是谁广播的**。两边互不依赖，这就是"解耦"。

---

## 2. 项目里的四种事件范围

项目里的事件接口都长一个样：`interface IXxxEvents : ISceneEvent<IXxxEvents>`。区别在**广播范围**：

| 类型 | 范围 | 怎么发 |
|---|---|---|
| `Global.IPlayerEvents` | **整个场景**，任何人都能收到 | `Global.IPlayerEvents.Post( x => x.OnPlayerPickup( e ) )` |
| `Local.IPlayerEvents` | **只发给某个 GameObject 及其子级** | `Local.IPlayerEvents.PostToGameObject( go, x => x.OnPickup( e ) )` |
| `Scene.RunEvent<T>` | **整个场景**（另一种写法） | `Scene.RunEvent<Global.IDamageEvents>( x => x.OnDamaging( e ) )` |
| `IEvent.PostToGameObject` | 发给某个 GameObject | `IEvent.PostToGameObject( WorldModel, x => x.OnCreateWorldModel() )` |

> `Local` 和 `Global` 的区别：`Local` 是"只通知这个玩家的机器"，适合音效、屏幕提示这类本地表现；`Global` 是"通知全场景"，适合别的系统也要知道的事。
> 注意 [`PlayerEvent.cs`](../Code/Player/PlayerEvent.cs) 里的注释：`OnMove` 和几个相机事件虽然写在 `Local` 里，但会**发到整个场景**，好让 UI 也能监听。

---

## 3. 事件怎么定义

事件接口就是一组**空方法**：

```csharp
public static partial class Global
{
    public interface IPlayerEvents : ISceneEvent<IPlayerEvents>
    {
        void OnPlayerSpawned( Player player ) { }
        void OnPlayerDied( Player player, PlayerDiedParams args ) { }
        void OnPlayerPickup( PlayerPickupEvent e ) { }
        void OnPlayerDrop( PlayerDropEvent e ) { }
        ...
    }
}
```

- 方法体是空的 `{ }`：监听者**只重写自己关心的那几行**，其他不用管。
- `ISceneEvent<T>`：引擎要求的标记接口，这样 `Scene` 才能找到并调用它。

事件的数据放在**事件类**里：

```csharp
public class PlayerPickupEvent
{
    public Player Player { get; init; }          // 谁捡的
    public BaseSandboxWeapon Weapon { get; init; } // 捡了什么
    public int Slot { get; init; }               // 放到哪个槽
    public bool Cancelled { get; set; }          // 关键：可以取消
}
```

> `init` 表示"创建时赋值，之后不能改"，防止监听者篡改数据。
> `Cancelled` 是这套事件系统的灵魂，第 6 节专门讲。

---

## 4. 怎么发事件

最典型的例子是捡武器（[`PlayerInventory.cs`](../Code/Player/PlayerInventory.cs)）：

```csharp
protected override bool OnAdding( Sandbox.BaseInventoryItem item, int slot )
{
    if ( item is not BaseSandboxWeapon weapon )
        return true;

    // 1. 造一个事件数据
    var pickupEvent = new PlayerPickupEvent { Player = Player, Weapon = weapon, Slot = slot };

    // 2. 本地广播（这个玩家的机器）
    Local.IPlayerEvents.PostToGameObject( Player.GameObject, e => e.OnPickup( pickupEvent ) );

    // 3. 全局广播（整个场景）
    Global.IPlayerEvents.Post( e => e.OnPlayerPickup( pickupEvent ) );

    // 4. 检查有没有人取消
    return !pickupEvent.Cancelled;
}
```

> 这里的 `OnAdding` 是**引擎的钩子**（在引擎把物品加进背包之前调用）。你返回 `false`，引擎就认为"添加失败"，会丢掉这个物品。这就是"用事件取消一个动作"的底层机制。

---

## 5. 怎么监听事件

只要让你的组件**实现事件接口**，引擎就会自动调用你：

```csharp
public sealed class PlayerLoadout : Component,
    Local.IPlayerEvents,      // 监听本地事件
    Global.IPlayerEvents,     // 监听全局事件
    Global.ISaveEvents        // 监听存档事件
{
    void Global.ISaveEvents.BeforeSave( string filename )
    {
        // 保存前把 hotbar 写进本地数据
        ...
    }

    void Local.IPlayerEvents.OnDied( PlayerDiedParams args )
    {
        // 玩家死亡时保存一次 hotbar（下次重生可能用得上）
        SaveLoadout();
    }
}
```

> 注意接口名前面写了 `Global.` / `Local.`：因为两个接口都有 `IPlayerEvents`，必须写全名区分。

### 真实例子：禁止用物理枪抓玩家

[`Player.Physgun.cs`](../Code/Player/Player.Physgun.cs)：

```csharp
void IPhysgunEvent.OnPhysgunGrab( IPhysgunEvent.GrabEvent e )
{
    if ( !CanPhysgunPlayers( e.Grabber ) )
        e.Cancelled = true;      // 没权限 → 取消这次抓取
}
```

一行 `Cancelled = true`，就让整个服务器（或某个玩家）的物理枪抓不到人。**这就是事件系统的威力**：功能之间不用互相引用。

---

## 6. `Cancelled`：可取消事件

很多事件类都带一个 `Cancelled` 字段，形成固定套路：

```
发事件  →  检查 Cancelled  →  继续 or 放弃
```

项目里的可取消事件（节选）：

| 事件 | 什么时候 | 取消会怎样 |
|---|---|---|
| `PlayerPickupEvent` | 捡武器前 | 武器不被捡起 |
| `PlayerDropEvent` | 丢武器前 | 武器留在手上 |
| `PlayerSwitchWeaponEvent` | 切枪前 | 不切 |
| `PlayerSeatHolsterEvent` | 在座位上收枪前 | 不收 |
| `DamageEvent` | 玩家 / NPC 受伤前 | 不受伤害 |
| `DeathEvent` | 死亡前 | 不死 |
| `ExplosionEvent` | 爆炸前 | 不爆炸 |
| `IToolActionEvents.ActionData` | 工具枪动作前 | 动作不执行 |
| `ICarryEvent` | 搬起物体前 | 搬不动 |

**设计心得**：如果你写一个新功能，希望"别人能插手"，就发一个可取消事件。这是 s&box 里最标准的扩展方式。

---

## 7. 项目事件速查表

| 事件接口 | 内容 | 定义位置 |
|---|---|---|
| `Global.IPlayerEvents` | 玩家出生 / 死亡 / 受伤 / 跳跃 / 落地 / 切枪 / 丢枪 / 搬东西 / 击杀 | [PlayerEvent.cs](../Code/Player/PlayerEvent.cs) |
| `Local.IPlayerEvents` | 同上，但只发给玩家自己；`OnMove`、相机事件例外 | [PlayerEvent.cs](../Code/Player/PlayerEvent.cs) |
| `Global.IDamageEvents` | `OnDamaging` / `OnDamaged`，玩家和 NPC 通用 | [IDamageEvents.cs](../Code/Game/IDamageEvents.cs) |
| `Global.IDeathEvents` | `OnDying` / `OnDied`，可取消 | [IDeathEvents.cs](../Code/Game/IDeathEvents.cs) |
| `Global.IExplosionEvents` | `OnExplode` / `OnExploded` | [Explosion.cs](../Code/Game/Explosion.cs) |
| `Global.ISaveEvents` | `BeforeSave` / `AfterLoad` | [ISaveEvents.cs](../Code/Save/ISaveEvents.cs) |
| `Global.IControlEvents` | `OnControlStarted` / `OnControlStopped`（座位 / 机械） | [IControlEvents.cs](../Code/Game/ControlSystem/IControlEvents.cs) |
| `IToolActionEvents` | 工具枪动作前 / 后 | [IToolActionEvents.cs](../Code/Weapons/ToolGun/IToolActionEvents.cs) |

> 想看一个功能"能被哪些插件插手"，最快的办法就是搜它发的事件名。

---

## 8. 小结

- 事件 = **广播一件事，谁关心谁监听**，两边互不依赖。
- 四种范围：`Global`（全场景）、`Local`（某个 GameObject）、`Scene.RunEvent`（全场景）、`IEvent.PostToGameObject`（某个 GameObject）。
- 事件类带数据 + `Cancelled`；`Cancelled = true` 就能取消整个动作。
- 想扩展一个功能，**发一个可取消事件**是最标准的做法。
- 这也是 s&box 插件系统的基础：插件几乎不改官方代码，只监听事件。

**本课跳过**：事件的网络行为（谁在主机上广播、谁收到），最后一课讲。

---

## 下一课

- [第 4 课：功能设计实战](feature-design-patterns.zh-cn.md) — 组件 + 配置 + 自动发现
- [Player.cs 第 1 课](player-walkthrough.zh-cn.md) — 把事件和 Player 串起来看
