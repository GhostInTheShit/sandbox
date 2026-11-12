# GameManager.cs 代码讲解（中文注释版）

> 本文逐段讲解 [`Code/GameLoop/GameManager.cs`](../Code/GameLoop/GameManager.cs)。
> `GameManager` 是整个沙盒游戏的"总管"：负责开房间、生成玩家、处理死亡、同步属性、清理世界。
> 建议对照原文阅读，本文中的代码片段带有中文注释。

---

## 目录

1. [这个类是什么](#1-这个类是什么)
2. [类声明与构造](#2-类声明与构造)
3. [开局：创建大厅](#3-开局创建大厅)
4. [通知系统](#4-通知系统)
5. [玩家连接与断开](#5-玩家连接与断开)
6. [生成玩家（核心）](#6-生成玩家核心)
7. [读档后补生成玩家](#7-读档后补生成玩家)
8. [请求复活](#8-请求复活)
9. [寻找出生点](#9-寻找出生点)
10. [死亡处理](#10-死亡处理)
11. [属性修改 RPC 工具方法](#11-属性修改-rpc-工具方法)
12. [物理与清理](#12-物理与清理)
13. [总结与阅读建议](#13-总结与阅读建议)

---

## 1. 这个类是什么

`GameManager` 是一个**场景级单例系统**（`GameObjectSystem<GameManager>`），意味着：

- 每个场景只有一个实例，用 `GameManager.Current` 访问；
- 它不属于某个 GameObject，而是"场景的一部分"；
- 它同时实现了多个引擎/游戏接口，由引擎在合适的时机回调。

它还是 `partial class`，完整功能分散在多个文件里：

| 文件 | 内容 |
|---|---|
| `GameManager.cs` | 大厅、玩家生成、死亡、属性 RPC、清理（本文） |
| `GameManager.Spawn.cs` | `Spawn` 生成道具/实体 |
| `GameManager.Gravity.cs` | 全局重力 |
| `GameManager.Water.cs` | 水效果 |
| `GameManager.MenuBar.cs` | 菜单栏 |
| `GameManager.ContextMenu.cs` | 右键菜单 |
| `GameManager.Achievements.cs` | 成就 |
| `GameManager.PropImpactShake.cs` | 道具撞击震屏 |
| `GameManager.Util.cs` | 工具方法 |

---

## 2. 类声明与构造

```csharp
// 使用 Sandbox.Npcs（NPC 系统）和 Sandbox.UI（通知等）
using Sandbox.Npcs;
using Sandbox.UI;

// sealed：不允许被继承
// partial：类被拆成多个文件（见上表）
// GameObjectSystem<GameManager>：场景级单例系统，每个场景一个实例
// 后面一串接口：引擎会在合适的时机回调它们
public sealed partial class GameManager : GameObjectSystem<GameManager>,
    Component.INetworkListener,   // 监听玩家连接/断开
    ISceneStartup,                // 场景启动（主机初始化）
    IScenePhysicsEvents,          // 物理事件（如物体飞出世界）
    ICleanupEvents,               // 清理事件
    Global.ISaveEvents            // 存档事件
{
    // 构造函数：必须调用 base(scene) 把场景传给基类
    public GameManager( Scene scene ) : base( scene )
    {
    }
}
```

**要点**：

- 这些接口大多是**显式接口实现**（方法名前面带接口名，如 `void ISceneStartup.OnHostInitialize()`），这样方法不会污染类的公开 API；
- 构造函数只有一行，说明初始化逻辑都放在各个接口的回调里。

---

## 3. 开局：创建大厅

```csharp
// 显式实现 ISceneStartup：场景启动时在主机上调用
void ISceneStartup.OnHostInitialize()
{
    // 如果还没有网络会话（比如玩家直接单机启动），就创建一个大厅
    if ( !Networking.IsActive )
    {
        Networking.CreateLobby( new Sandbox.Network.LobbyConfig()
        {
            Privacy     = Sandbox.Network.LobbyPrivacy.Public,  // 公开大厅
            MaxPlayers  = 32,                                   // 最多 32 人
            Name        = "Sandbox",                            // 大厅名字
            DestroyWhenHostLeaves = false                       // 房主退出后不销毁
        } );
    }
}
```

**要点**：

- 这是整个游戏"开始"的地方：先有大厅，玩家才能连进来；
- `MaxPlayers = 32` 是大厅人数，而 `sandbox.sbproj` 里的 `MaxPlayers = 64` 是游戏支持的上限，两者不冲突；
- 如果是在线加入别人的游戏，`Networking.IsActive` 为真，就不用自己开大厅。

---

## 4. 通知系统

```csharp
// 向所有人发送一条普通文本通知（只有主机能调用）
internal void Notify( string text )
{
    // 断言：只有主机可以发通知，否则说明代码写错了
    Assert.True( Networking.IsHost, "Only the host can send notifications" );
    NotifyRpc( text );   // 调用下面的 RPC
}

// [Rpc.Broadcast] 表示这个函数会被广播到所有客户端执行
// NetFlags.HostOnly 表示只有主机才能发起这个广播
[Rpc.Broadcast( NetFlags.HostOnly )]
private void NotifyRpc( string text )
{
    // 把文本加到聊天栏
    Sandbox.Platform.Chat.AddText( text );
}
```

```csharp
// 发送一条"本地化"通知：只广播一个短语 key 和参数，由每个客户端自己翻译成自己的语言
internal void NotifyLocalized( string phrase, Dictionary<string, string> tokens )
{
    Assert.True( Networking.IsHost, "Only the host can send notifications" );

    // 把参数字典序列化成 JSON，随 RPC 一起发送（RPC 不能直接传 Dictionary）
    NotifyLocalizedRpc( phrase, Sandbox.Json.Serialize( tokens ) );
}

[Rpc.Broadcast( NetFlags.HostOnly )]
private void NotifyLocalizedRpc( string phrase, string tokensJson )
{
    // 客户端收到后反序列化，再用本地语言解析短语
    var tokens = Sandbox.Json.Deserialize<Dictionary<string, string>>( tokensJson );
    Sandbox.Platform.Chat.AddText( LocalizedText.Resolve( phrase, tokens ) );
}
```

**要点**：

- 这是典型的"主机权威"模式：**主机决定发什么，客户端只负责显示**；
- `NotifyLocalized` 的设计很聪明：不同语言的客户端各自翻译，主机不用关心玩家用什么语言；
- `Rpc.Broadcast(NetFlags.HostOnly)` = "主机发起，所有人执行"。

---

## 5. 玩家连接与断开

```csharp
// 显式实现 INetworkListener：有连接请求时调用
// 返回 false 表示拒绝连接，reason 会作为拒绝理由
bool Component.INetworkListener.AcceptConnection( Connection channel, ref string reason )
{
    // 构造一个"正在连接"事件，让监听者（如封禁系统）有机会拒绝
    var e = new ConnectingEvent { Client = channel, Reason = reason };
    Scene.RunEvent<Global.IConnectionEvents>( x => x.OnConnecting( e ) );

    reason = e.Reason;          // 监听者可能修改了理由
    return !e.Cancelled;        // 被取消 = 拒绝连接
}
```

```csharp
// 连接正式激活（玩家真正进入服务器）时调用
void Component.INetworkListener.OnActive( Connection channel )
{
    // 客户端不能自己生成网络对象，只能生成纯本地的
    channel.CanSpawnObjects = false;

    // 1. 为该连接创建 PlayerData（保存击杀/死亡等数据）
    var playerData = CreatePlayerInfo( channel );

    // 2. 广播"玩家加入"事件（UI、统计等会响应）
    Scene.RunEvent<Global.IConnectionEvents>( x => x.OnPlayerJoined( playerData ) );

    // 3. 生成玩家对象
    SpawnPlayer( playerData );

    // 4. 检查成就、好友在线统计
    CheckConnectionAchievement( channel );
    CheckFriendsOnlineStat();
}
```

```csharp
// 有玩家离开时调用（只在主机上）
void Component.INetworkListener.OnDisconnected( Connection channel )
{
    // 找到该连接对应的 PlayerData
    var pd = PlayerData.For( channel );
    if ( pd is not null )
    {
        // 广播"玩家离开"事件
        Scene.RunEvent<Global.IConnectionEvents>( x => x.OnPlayerLeft( pd ) );

        // 销毁该玩家的数据对象
        pd.GameObject.Destroy();
    }

    // 清理该玩家的撤销栈，防止内存泄漏
    UndoSystem.Current?.RemovePlayer( channel.SteamId );
}
```

```csharp
// 为一个连接创建（或找回）PlayerData
private PlayerData CreatePlayerInfo( Connection channel )
{
    // 如果已经有数据（比如断线重连），直接复用
    var existingPlayerInfo = PlayerData.For( channel );
    if ( existingPlayerInfo.IsValid() )
        return existingPlayerInfo;

    // 新建一个 GameObject 挂 PlayerData 组件
    var go = new GameObject( true, $"PlayerInfo - {channel.DisplayName}" );
    var data = go.AddComponent<PlayerData>();

    // 生成到网络上，并归属给这个连接（这样该客户端能收到自己的数据）
    go.NetworkSpawn( channel );

    // 所有权固定，不转移
    go.Network.SetOwnerTransfer( OwnerTransfer.Fixed );

    return data;
}
```

**要点**：

- `PlayerData` 是**网络对象**，每个玩家一份，用来存击杀/死亡/上帝模式等持久数据；
- `channel.CanSpawnObjects = false` 很重要：防止客户端自己生成网络对象（安全）；
- 加入/离开都通过 `IConnectionEvents` 广播，具体响应由别的系统负责。

---

## 6. 生成玩家（核心）

这是全文件最重要的部分。有两个入口：

- `SpawnPlayer`：常规生成（会询问监听者是否允许）；
- `Respawn`：强制重生（用于自定义回合制的插件）。

```csharp
/// 为该连接生成一个玩家；如果它还没有的话。只在主机执行。
/// 会先触发 OnPlayerRespawning，监听者可以移动出生点或取消生成。
/// force=true 时跳过该事件（用于插件自己控制生成时机）。
public Player SpawnPlayer( Connection connection, bool force = false )
    => SpawnPlayer( PlayerData.For( connection ), force );

public Player SpawnPlayer( PlayerData playerData, bool force = false )
{
    Assert.NotNull( playerData, "PlayerData is null" );
    // 只有主机能生成玩家
    Assert.True( Networking.IsHost, $"Client tried to SpawnPlayer: {playerData.Network.Owner?.DisplayName}" );

    // 这个连接已经有玩家了吗？有就直接返回，避免重复生成
    if ( Scene.GetAll<Player>().Any( x => x.Network.Owner == playerData.Network.Owner ) )
        return null;

    // 1. 找一个出生点
    var startLocation = FindSpawnLocation().WithScale( 1 );

    // 2. 非强制时，触发"即将重生"事件
    //    监听者可以修改出生点位置，或者取消（比如回合制游戏还没到复活时间）
    if ( !force )
    {
        var respawnEvent = new PlayerRespawnEvent { PlayerData = playerData, SpawnLocation = startLocation };
        Global.IPlayerEvents.Post( x => x.OnPlayerRespawning( respawnEvent ) );

        if ( respawnEvent.Cancelled )
            return null;   // 被取消，不生成

        startLocation = respawnEvent.SpawnLocation;   // 监听者可能移动了出生点
    }

    // 3. 克隆玩家预制体到世界里（先禁用，设置好归属再启用）
    var playerGo = GameObject.Clone( "/prefabs/engine/player.prefab",
        new CloneConfig
        {
            Name = playerData.Network.Owner?.DisplayName,
            StartEnabled = false,          // 先禁用，避免生成过程中被看到
            Transform = startLocation
        } );

    // 4. 取出预制体上的 Player 组件，关联 PlayerData
    var player = playerGo.Components.Get<Player>( true );
    player.PlayerData = playerData;

    // 5. 生成到网络，归属给这个连接
    var owner = playerData.Network.Owner;
    playerGo.NetworkSpawn( owner );

    // 6. 广播生成事件
    Local.IPlayerEvents.PostToGameObject( player.GameObject, x => x.OnSpawned() );   // 只发给本地相关
    Global.IPlayerEvents.Post( x => x.OnPlayerSpawned( player ) );                    // 广播给所有人

    return player;
}
```

**要点**：

- `StartEnabled = false` 是常见技巧：先克隆、设置好所有数据，再启用，避免"半成品"被同步出去；
- `player.prefab` 是核心预制体，包含 `Player`、`PlayerController`、`PlayerInventory`、`Rigidbody`、各种 `MoveMode` 等；
- 生成流程是"**主机克隆 + 网络生成 + 归属给该玩家**"，这是 s&box 多人生成的标准做法。

```csharp
/// 强制重生：不管死活都重生。用于插件自己的回合流程。
public Player Respawn( Connection connection )
{
    Assert.True( Networking.IsHost, "Only the host can respawn players" );

    var playerData = PlayerData.For( connection );
    if ( !playerData.IsValid() )
        return null;

    // 如果已有玩家对象，直接销毁（不走死亡流程）
    var existing = Player.FindForConnection( connection );
    if ( existing.IsValid() )
    {
        existing.GameObject.Destroy();
    }

    // 清掉死亡观察者
    DestroyObservers( connection );

    // 强制生成（跳过 OnPlayerRespawning 事件）
    return SpawnPlayer( playerData, force: true );
}
```

```csharp
/// 清理该连接遗留的死亡观察者（尸体旁的观战相机）
private void DestroyObservers( Connection connection )
{
    foreach ( var observer in Scene.GetAllComponents<PlayerObserver>()
                                      .Where( x => x.Network.Owner == connection )
                                      .ToArray() )
    {
        observer.GameObject.Destroy();
    }
}
```

---

## 7. 读档后补生成玩家

```csharp
// 显式实现 ISaveEvents：读档完成后调用
void Global.ISaveEvents.AfterLoad( string filename )
{
    if ( !Networking.IsHost ) return;

    // 读档可能只保存了世界，没保存所有玩家对象
    // 所以给每个已连接的连接补生成一个玩家
    foreach ( var connection in Connection.All )
    {
        var playerData = CreatePlayerInfo( connection );
        SpawnPlayer( playerData );
    }
}
```

**要点**：读档 = 重建世界 + 重新生成玩家。这就是为什么读档后玩家会"刷新"一下。

---

## 8. 请求复活

```csharp
/// 客户端（通过死亡观察者）请求复活时调用
[Rpc.Host]   // 客户端调用，主机执行
internal void RequestRespawn()
{
    var connection = Rpc.Caller;   // 谁发起的

    // 主机尝试生成（如果被事件取消，就继续保持观战）
    var player = SpawnPlayer( connection );
    if ( !player.IsValid() )
        return;

    // 生成成功，销毁观察者
    DestroyObservers( connection );
}
```

**要点**：

- `Rpc.Caller` 表示"发起这次 RPC 的连接"；
- 如果 `OnPlayerRespawning` 被插件取消（比如复活冷却），玩家会继续留在观战状态。

---

## 9. 寻找出生点

```csharp
/// 站立玩家的近似包围盒，用于检查出生点是否被占据、是否落地
private static readonly BBox PlayerHull = new(
    new Vector3( -16f, -16f, 0f ),
    new Vector3( 16f, 16f, 72f ) );

/// 上次生成玩家的位置，避免重复用同一个点
private Vector3? _lastSpawnPosition;

/// 找一个重生位置：落地良好、没人站、且尽量避开上次的位置
Transform FindSpawnLocation()
{
    // 收集场景里所有 SpawnPoint 组件
    var spawnPoints = Scene.GetAllComponents<SpawnPoint>().ToArray();

    // 没有出生点就退回世界原点，并警告（地图需要放 SpawnPoint）
    if ( spawnPoints.Length == 0 )
    {
        Log.Warning( "No SpawnPoint components in the scene — spawning at the world origin. Add a SpawnPoint to the map to fix this." );
        return Transform.Zero;
    }

    // 打乱顺序，逐个尝试
    var transform = SpawnPlacement.FindSpawnPosition(
        spawnPoints.OrderBy( _ => Random.Shared.Next() ).Select( x => x.Transform.World ),
        PlayerHull,                        // 用玩家包围盒检查是否被挡住
        avoid: _lastSpawnPosition );       // 避开上次的位置

    _lastSpawnPosition = transform.Position;
    return transform;
}
```

**要点**：

- 地图通过放置 `SpawnPoint` 组件来告诉游戏"这里可以出生"；
- `SpawnPlacement.FindSpawnPosition` 负责检查地面和遮挡（工具类在 [`Code/Utility/SpawnPlacement.cs`](../Code/Utility/SpawnPlacement.cs)）。

---

## 10. 死亡处理

```csharp
/// 主机上玩家被杀时调用
internal void OnDeath( Player player, DamageInfo dmg )
{
    Assert.True( Networking.IsHost );
    Assert.True( player.IsValid(), "Player invalid" );
    Assert.True( player.PlayerData.IsValid(), $"{player.GameObject.Name}'s PlayerData invalid" );

    // 1. 找出"谁该为这次伤害负责"
    //    DamageBlame 会追踪：谁扔的道具、谁在开车等
    var source = DamageBlame.Find( dmg ) as IKillSource
                 ?? dmg.Attacker?.GetComponentInParent<IKillSource>( true );
    if ( source == null ) return;   // 找不到来源就不记击杀

    // 2. 判断是否自杀
    var isSuicide = source is Player p && p == player;

    // 3. 非自杀时，通知来源"你击杀了一个人"（来源可能是玩家、车辆等）
    if ( !isSuicide )
        source.OnKill( player.GameObject );

    // 4. 如果来源是玩家，给它记击杀事件
    if ( !isSuicide && source is Player killer )
    {
        var killEvent = new PlayerKillEvent
        {
            Player = killer,
            Victim = player.GameObject,
            DamageInfo = dmg
        };
        Local.IPlayerEvents.PostToGameObject( killer.GameObject, x => x.OnKill( killEvent ) );
        Global.IPlayerEvents.Post( x => x.OnPlayerKill( killEvent ) );
    }

    // 5. 死亡数 +1
    player.PlayerData.Deaths++;

    // 6. 准备击杀信息（武器图标、伤害标签等），发到击杀播报
    var weapon = dmg.Weapon;
    var w = weapon.IsValid() ? weapon.GetComponentInChildren<IKillIcon>() : null;
    var damageTags = dmg.Tags.ToString() + ( isSuicide ? " suicide" : "" );
    var attackerTags = isSuicide ? "" : source.Tags;
    var attackerName = isSuicide ? null : source.DisplayName;
    var attackerSteamId = isSuicide ? 0L : source.SteamId;
    var playerName = player.Network.Owner?.DisplayName ?? "Unknown";
    var playerSteamId = (long)(player.Network.Owner?.SteamId ?? default);

    // 通知击杀播报 UI
    Scene.RunEvent<Feed>( x => x.NotifyKill( playerName, playerSteamId, attackerName,
                                             attackerSteamId, damageTags, attackerTags, "", w?.DisplayIcon ) );

    // 7. 同时在控制台/聊天里输出一行文字
    if ( string.IsNullOrEmpty( attackerName ) )
    {
        NotifyConsole( $"{playerName} died (tags: {dmg.Tags})" );
    }
    else if ( weapon.IsValid() )
    {
        NotifyConsole( $"{attackerName} killed {(isSuicide ? "self" : playerName)} with {weapon.Name} (tags: {dmg.Tags})" );
    }
    else
    {
        NotifyConsole( $"{attackerName} killed {(isSuicide ? "self" : playerName)} (tags: {dmg.Tags})" );
    }
}
```

```csharp
/// 主机上 NPC 被杀时调用，逻辑类似，但没有自杀概念
internal void OnNpcDeath( Npc victim, DamageInfo dmg )
{
    Assert.True( Networking.IsHost );
    Assert.True( victim.IsValid(), "NPC invalid" );

    var source = DamageBlame.Find( dmg ) as IKillSource ?? dmg.Attacker?.GetComponent<IKillSource>();
    source?.OnKill( victim.GameObject );

    if ( source is Player killer )
    {
        var killEvent = new PlayerKillEvent { Player = killer, Victim = victim.GameObject, DamageInfo = dmg };
        Local.IPlayerEvents.PostToGameObject( killer.GameObject, x => x.OnKill( killEvent ) );
        Global.IPlayerEvents.Post( x => x.OnPlayerKill( killEvent ) );
    }

    // 击杀播报里 NPC 的 SteamId 是 0，类型标记为 "npc"
    var w = dmg.Weapon.IsValid() ? dmg.Weapon.GetComponentInChildren<IKillIcon>() : null;
    var attackerName = source?.DisplayName;
    var attackerSteamId = source?.SteamId ?? 0L;
    var attackerTags = source?.Tags ?? "";

    Scene.RunEvent<Feed>( x => x.NotifyKill( victim.DisplayName, 0, attackerName,
                                             attackerSteamId, dmg.Tags.ToString(), attackerTags, "npc", w?.DisplayIcon ) );
}
```

**要点**：

- `DamageBlame` 是"伤害归因"系统：被车撞、被自己扔的道具砸，都能算到正确的人头上（见 [`Code/Player/DamageBlame.cs`](../Code/Player/DamageBlame.cs)）；
- `IKillSource` 接口让"任何能击杀的东西"（玩家、车辆、NPC）都能被记击杀；
- 死亡本身在 `Player.Kill()` 里执行（创建布娃娃、观察者），`GameManager.OnDeath` 只负责"记功和播报"。

---

## 11. 属性修改 RPC 工具方法

这一组方法是给**工具/UI**用的：客户端请求修改世界里的某个属性，主机执行后同步给所有人。

```csharp
/// 远程修改一个组件属性（客户端调用，主机执行并广播）
[Rpc.Host]
internal static void ChangeProperty( Component c, string propertyName, object value )
{
    if ( !c.IsValid() ) return;

    // 通过类型库找到这个属性的定义
    var tl = TypeLibrary.GetType( c.GetType() );
    if ( tl is null ) return;

    var prop = tl.GetProperty( propertyName );
    if ( prop is null ) return;

    // 设置值
    prop.SetValue( c, value );

    // 让这个组件所在的 GameObject 刷新同步，把变化发给所有客户端
    // （注释里提到 Refresh(c) 理论上更优但有问题，所以整对象刷新）
    c.GameObject.Network?.Refresh();
}
```

```csharp
/// 批量应用morph（表情/脸部变形）变化，只改传入的那几个
[Rpc.Host]
internal static void ApplyMorphBatch( SkinnedModelRenderer smr, string morphsJson )
{
    if ( !smr.IsValid() ) return;

    // MorphState 是一个附加组件，专门负责把 morph 变化同步给其他客户端
    smr.GameObject.GetOrAddComponent<MorphState>().ApplyBatch( morphsJson );
}
```

```csharp
/// 应用一整套脸部预设（JSON），由 MorphState 负责同步
[Rpc.Host]
internal static void ApplyFacePosePreset( SkinnedModelRenderer smr, string morphsJson )
{
    if ( !smr.IsValid() ) return;

    smr.GameObject.GetOrAddComponent<MorphState>().ApplyPreset( morphsJson );
}
```

```csharp
/// 修改某个 renderer 指定槽位的材质覆盖（比如染色工具）
[Rpc.Host]
internal static async void ChangeMaterialOverride( ModelRenderer renderer, int materialIndex, string materialPath )
{
    if ( !renderer.IsValid() ) return;

    Material material = null;

    if ( !string.IsNullOrEmpty( materialPath ) )
    {
        // 先尝试本地加载，失败再从云端下载
        material = Material.Load( materialPath );
        material ??= await Cloud.Load<Material>( materialPath );
    }

    // await 之后对象可能已被销毁，必须重新检查
    if ( !renderer.IsValid() ) return;

    renderer.Materials.SetOverride( materialIndex, material );

    // 同步给所有人
    renderer.GameObject.Network?.Refresh();
}
```

```csharp
/// 设置替换所有槽位的材质（整体换材质），传 null/空串清除
[Rpc.Host]
internal static async void ChangeWholeMaterialOverride( ModelRenderer renderer, string materialPath )
{
    if ( !renderer.IsValid() ) return;

    Material material = null;

    if ( !string.IsNullOrEmpty( materialPath ) )
    {
        material = Material.Load( materialPath );
        material ??= await Cloud.Load<Material>( materialPath );
    }

    if ( !renderer.IsValid() ) return;

    renderer.MaterialOverride = material;

    renderer.GameObject.Network?.Refresh();
}
```

```csharp
/// 修改材质组（模型上的材质分组）
[Rpc.Host]
internal static void ChangeMaterialGroup( ModelRenderer renderer, string group )
{
    if ( !renderer.IsValid() ) return;

    renderer.MaterialGroup = group;

    // 如果是道具，同步更新 Prop 上的记录
    var prop = renderer.GetComponent<Prop>();
    if ( prop.IsValid() )
        prop.MaterialGroup = group;

    renderer.GameObject.Network?.Refresh();
}
```

```csharp
/// 修改模型的 bodygroup（模型部件选择，比如换脸/换零件）
[Rpc.Host]
internal static void ChangeBodyGroup( ModelRenderer renderer, string partName, int choice )
{
    if ( !renderer.IsValid() ) return;

    // 多个对象可能被同时选中，跳过那些模型没有这个选项的
    var part = renderer.Model?.Parts.Get( partName );
    if ( part is null || choice < 0 || choice >= part.Choices.Count ) return;

    renderer.SetBodyGroup( part.Index, choice );

    // 同步到 Prop 上
    var prop = renderer.GetComponent<Prop>();
    if ( prop.IsValid() )
        prop.BodyGroups = renderer.BodyGroups;

    renderer.GameObject.Network?.Refresh();
}
```

**要点**：

- 这一组方法体现了"**主机权威**"：客户端只能"请求"，真正改世界的是主机；
- `Network.Refresh()` 是把对象变化同步给所有人的关键；
- `async` 方法里 `await` 之后必须重新检查对象有效性（await 期间对象可能被销毁）。

---

## 12. 物理与清理

```csharp
// 物理事件：刚体飞出世界边界时调用
void IScenePhysicsEvents.OnOutOfBounds( Rigidbody body )
{
    // 直接销毁，防止物体掉出地图
    body.DestroyGameObject();
}
```

```csharp
// 清理事件：清理系统完成一次清理后调用
void ICleanupEvents.OnCleanup( int removedObjects, int restoredObjects )
{
    // 玩家尸体不是网络对象，主机的清理会漏掉其他人的
    // 所以在这里把所有死亡观察目标（尸体）都销毁
    foreach ( var corpse in Scene.GetAllComponents<DeathCameraTarget>().ToArray() )
    {
        corpse.GameObject.Destroy();
    }

    // 根据是"清理"还是"恢复"选择一条本地化提示
    var phrase = restoredObjects > 0 ? "cleanup.restored" : "cleanup.removed";
    var message = Game.Language.GetPhrase( phrase, new()
    {
        { "removed", removedObjects },
        { "restored", restoredObjects }
    } );

    // 在屏幕角落显示通知
    Notices.AddNotice( "cleaning_services", Color.Green, message );
}
```

**要点**：

- 清理系统会移除"不属于任何人的遗留物"，并恢复被误删的基线对象；
- `Notices.AddNotice` 是全局通知 API，参数是 id、颜色、文本。

---

## 13. 总结与阅读建议

### 13.1 这个文件的设计模式

| 模式 | 体现 |
|---|---|
| **场景单例系统** | `GameObjectSystem<GameManager>`，全场景唯一 |
| **显式接口实现** | 把引擎回调藏起来，不污染公开 API |
| **主机权威** | 所有改变世界的操作都先检查 `Networking.IsHost` |
| **事件广播** | `Global.X.Post` / `Local.X.PostToGameObject` / `Scene.RunEvent<T>` |
| **RPC 工具方法** | `[Rpc.Host]` 让客户端请求主机改属性 |
| **partial 拆分** | 按功能把大类拆成多个文件 |

### 13.2 记住这条主线

```
开大厅 → 玩家连接 → 创建 PlayerData → 生成玩家 → 游戏
                                                          ↓
                                   受伤/死亡 → OnDeath 记功 → 布娃娃+观战
                                                          ↓
                                   请求复活 → RequestRespawn → 重新生成
```

### 13.3 继续阅读

1. [`Code/GameLoop/GameManager.Spawn.cs`](../Code/GameLoop/GameManager.Spawn.cs) —— 道具/实体是怎么生成的；
2. [`Code/Player/Player.cs`](../Code/Player/Player.cs) —— 玩家受伤、死亡、布娃娃的细节；
3. [`Code/Player/PlayerData.cs`](../Code/Player/PlayerData.cs) —— 玩家持久数据；
4. [`Code/Player/DamageBlame.cs`](../Code/Player/DamageBlame.cs) —— 伤害归因系统；
5. [`Code/Cleanup/CleanupSystem.cs`](../Code/Cleanup/CleanupSystem.cs) —— 清理系统如何工作。

> 配套的入门文档见 [`docs/learning-guide.zh-cn.md`](./learning-guide.zh-cn.md)。
