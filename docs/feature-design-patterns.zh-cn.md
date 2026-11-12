# 游戏开发基础 · 第 4 课：游戏功能设计实战

> 本课只讲**一个知识点**：一个游戏功能 = **组件 + 配置属性 + 接口约定 + 自动发现**。
> 用工具枪的 30+ 种工具模式做例子，看完你就明白项目里大部分功能是怎么搭起来的。
> 网络部分只标一句"有这回事"，细节放最后一课。约 10 分钟读完。
> 前置：[第 1~3 课](game-loop-and-lifecycle.zh-cn.md)、[学习指南](learning-guide.zh-cn.md)。

---

## 1. 先看一个真实的功能

工具枪（Toolgun）有 30 多种工具：重力、焊接、气球、灯光、弹簧、推进器……

- 每种工具是一个独立的 `ToolMode` 组件（[`GravityTool.cs`](../Code/Weapons/ToolGun/Modes/GravityTool.cs) 等）。
- **工具枪自己不知道有哪些工具**。它用一行代码把所有工具都找出来：

```csharp
foreach ( var mode in Game.TypeLibrary.GetTypes<ToolMode>() ) { ... }
```

- 所以你新增一个工具，**不需要改工具枪**，只要新写一个 `ToolMode` 子类。

> 这就是"对扩展开放、对修改关闭"：加功能 = 加文件，不是改老代码。

---

## 2. 模式一：基类定义"接口约定"

[`ToolMode.cs`](../Code/Weapons/ToolGun/ToolMode.cs)：

```csharp
public abstract partial class ToolMode : Component, IToolInfo
{
    public bool IsValidState { get; protected set; } = true;

    /// 为 true 时，工具枪会"吃掉"鼠标输入，工具自己读 Input.AnalogLook 来用鼠标
    public virtual bool AbsorbMouseInput => false;

    /// 工具显示名，默认取 TypeDescription 的 Title
    public virtual string Name => Game.Language.GetPhrase( (TypeDescription?.Title ?? GetType().Name).TrimStart( '#' ) );

    /// 工具说明
    public virtual string Description => string.Empty;

    /// 主/副动作的提示文字（自动从注册的 action 里取）
    public virtual string PrimaryAction => GetActionName( ToolInput.Primary );
    public virtual string SecondaryAction => GetActionName( ToolInput.Secondary );
}
```

要点：

- `abstract`：不能直接创建，必须继承。
- `virtual`：给一个默认实现，子类**按需重写**。
- [`IToolInfo`](../Code/UI/ToolInfo/IToolInfo.cs)：额外的标记接口，让工具枪和其他系统识别"这是个工具"。

> 基类只负责"约定"（必须有什么），不负责"实现"。子类写自己特有的行为。

---

## 3. 模式二：属性 = 数据 + 配置

看 [`GravityTool.cs`](../Code/Weapons/ToolGun/Modes/GravityTool.cs) 的头部：

```csharp
[Icon( "🪂" )]                                    // 工具菜单里的图标
[Title( "#tool.name.gravity" )]                  // 显示名（走本地化）
[ClassName( "gravity" )]                         // 切换工具时用的名字
[Group( "#tool.group.tools" )]                   // 在工具菜单里归到哪组
public sealed class GravityTool : ToolMode
{
    // 同步 + 属性面板可编辑 + 显示名 + 范围 + 步长
    [Sync, Property, Title( "#tool.setting.gravity_scale" ), Range( -1, 4 ), Step( 0.05f )]
    public float Value { get; set; } = 1.0f;     // 重力倍率
}
```

一行 `[...]` 干了很多事：

| Attribute | 作用 |
|---|---|
| `[Icon]` / `[Title]` / `[Group]` | 让工具出现在工具菜单里 |
| `[ClassName]` | 切换工具时按名字找它 |
| `[Property]` | 在编辑器/工具面板里显示这个属性 |
| `[Range]` / `[Step]` | 面板里给个滑块，限制取值 |
| `[Sync]` | 属性值在网络间同步（网络，先知道有这回事） |

> **设计心得**：把"玩家能调的参数"做成属性，比写死在代码里好得多——策划能在游戏里调，不用重新编译。

---

## 4. 模式三：注册动作，基类统一分发

工具不用自己写 `if (Input.Pressed("attack1"))`，而是**注册**动作，基类每帧检查：

```csharp
protected override void OnStart()
{
    base.OnStart();

    // 注册三个动作：左键设置、右键复制、装填键重置
    RegisterAction( ToolInput.Primary,   () => "#tool.hint.gravity.set",   OnSetGravity   );
    RegisterAction( ToolInput.Secondary, () => "#tool.hint.gravity.copy",  OnCopyGravity  );
    RegisterAction( ToolInput.Reload,    () => "#tool.hint.gravity.reset", OnResetGravity );
}
```

`RegisterAction` 的签名（[`ToolMode.cs`](../Code/Weapons/ToolGun/ToolMode.cs)）：

```csharp
protected void RegisterAction( ToolInput input, Func<string> name, Action callback, InputMode mode = InputMode.Pressed )
{
    if ( IsProxy ) return;      // 网络：只有主机注册（先知道有这回事）
    _actions.Add( new ToolActionEntry( input, name, callback, mode ) );
}
```

基类在每帧的 `OnControl` 里自动分发（[`ToolMode.cs`](../Code/Weapons/ToolGun/ToolMode.cs)）：

```csharp
private void DispatchActions()
{
    foreach ( var action in _actions )
    {
        bool active = action.Mode == InputMode.Down
            ? Input.Down( inputName )
            : Input.Pressed( inputName );

        if ( active )
            action.Callback?.Invoke();
    }
}
```

好处：

- 工具只写"做什么"（`OnSetGravity`），不写"什么时候做"。
- 提示文字（`Func<string>`）能随状态变化，比如"按住以蓄力"。
- 基类还能统一处理 `InputMode.Down`（按住）和 `Pressed`（按下）。

---

## 5. 模式四：主机权威 + 事件把关

工具动作最终要改变世界（比如改重力）。这一步分两层：

```csharp
[Rpc.Host( NetFlags.OwnerOnly )]          // ← 网络：只有主机真正执行（先知道有这回事）
private void SetGravity( GameObject target, float scale, ToolInput input )
{
    if ( !FireToolAction( input ) ) return;   // ← 动作前事件：插件可以取消

    if ( !target.IsValid() || target.IsProxy ) return;

    scale = scale.Clamp( -4f, 4f );

    var props = target.GetOrAddComponent<PhysicalProperties>();
    props.GravityScale = scale;
    props.Apply();

    FirePostToolAction( input );              // ← 动作后事件：通知"已生效"
}
```

- `[Rpc.Host]`：客户端调用这个方法时，请求会被送到主机执行。**世界状态只有主机改**，这是防作弊的基础。
- `FireToolAction` / `FirePostToolAction`：动作**前**发一个可取消事件，动作**后**发一个成功事件（[`ToolMode.cs`](../Code/Weapons/ToolGun/ToolMode.cs)）：

```csharp
protected bool FireToolAction( ToolInput input )
{
    var data = new IToolActionEvents.ActionData { Tool = this, Input = input, Player = Player?.Network.Owner };
    Scene.RunEvent<IToolActionEvents>( x => x.OnToolAction( data ) );
    return !data.Cancelled;      // 有人取消 → 不执行
}
```

> 这就是第 3 课讲的事件系统：**工具动作也能被插件插手**。比如服务器可以禁止玩家用重力工具改某些物体。

---

## 6. 自动发现：工具枪怎么"认识"所有工具

[`Toolgun.cs`](../Code/Weapons/ToolGun/Toolgun.cs)：

```csharp
public void CreateToolComponents()
{
    if ( !Networking.IsHost ) { ... return; }

    bool enabled = true;      // 保证最多只有一个工具是"激活"的

    foreach ( var mode in Game.TypeLibrary.GetTypes<ToolMode>() )
    {
        if ( mode.IsAbstract ) continue;      // 跳过抽象基类

        // 如果工具枪的预制体里已经有这个组件，就复用它（保留它存的参数）
        var component = Components.Get( mode.TargetType, FindMode.EverythingInSelf )
                       ?? Components.Create( mode, enabled );

        enabled &= !component.Enabled;
    }

    Network.Refresh( GameObject );
}
```

- `Game.TypeLibrary.GetTypes<ToolMode>()`：反射找出所有 `ToolMode` 子类。
- `enabled` 这个变量保证**最多只有一个工具是激活的**：一旦某个工具处于启用状态，后面创建的就都是禁用。
- 切换工具就是改 `Enabled`（[`Toolgun.cs`](../Code/Weapons/ToolGun/Toolgun.cs)）：

```csharp
[Rpc.Host]
public void SetToolMode( string name )
{
    ...
    currentMode.Enabled = false;   // 关掉旧的
    newMode.Enabled = true;        // 打开新的
    ...
}
```

> **新增一个工具 = 新写一个 `ToolMode` 子类，啥都不用改。** 这就是"自动发现"的威力。

---

## 7. 这套模式的通用性

同样的套路在项目里随处可见，只是换了名字：

| 功能 | 基类 / 接口 | 发现方式 | 配置 |
|---|---|---|---|
| 工具模式 | `ToolMode` | `TypeLibrary.GetTypes<ToolMode>()` | `[Icon]` `[Title]` `[Group]` |
| 武器 | `BaseSandboxWeapon` | 生成菜单 / 背包 | `[Property]` 弹药、后坐力 |
| 座位 | `ISitTarget` | `GetComponentInParent<ISitTarget>()` | 座位位置、视角 |
| 控制模式 | `IPlayerControllable` | 座位/机械查找 | 输入映射 |
| 实体 | `Component` | 生成菜单 | `[Title]` `[Icon]` `[Category]` |

**共同点**：功能被拆成小组件，用接口/基类约定行为，用属性暴露配置，用查找或反射实现"自动发现"。

---

## 8. 如果你要加一个新功能

按这个顺序想：

1. **约定**：定义一个接口或抽象基类，写清楚"这个功能必须能做什么"。
2. **实现**：写一个组件，只实现自己特有的部分，其余继承基类。
3. **配置**：把可调参数做成 `[Property]` + Attribute，让玩家/策划能调。
4. **扩展**：在关键节点发可取消事件，让别人能插手。
5. **发现**：用接口查找（`GetComponent<T>`）或 `TypeLibrary.GetTypes<T>()` 自动找到你的实现。
6. **网络**：决定谁是权威（一般主机），用 `[Rpc.Host]` / `[Sync]` 标注（最后一课）。

> 这 6 步就是"游戏功能逻辑设计"的骨架。武器、工具、座位、载具、NPC 行为，基本都是这个模板。

---

## 9. 小结

- 一个功能 = **组件 + 配置 + 接口 + 发现**。
- 基类定约定，子类写行为；属性做配置；事件做扩展。
- 自动发现让"加功能 = 加文件"，不改老代码。
- 主机权威 + 可取消事件，让功能既安全又开放。

**本课跳过**：`[Rpc.Host]`、`[Sync]`、`IsProxy` 的具体机制（网络，最后一课讲）。

---

## 下一课

- [Player.cs 第 1 课](player-walkthrough.zh-cn.md) — 把这些模式用到 Player 上
- [武器系统讲解](weapon-system-walkthrough.zh-cn.md) — 看一个更复杂的功能怎么用这套模板
