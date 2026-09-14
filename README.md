# Practice-in-StateMachine · 有限状态机实践

> 基于 Unity 的 2D 动作游戏有限状态机（FSM）架构练习项目。
>
> 实现了玩家移动与攻击两套并行状态机，涵盖跳跃缓冲、土狼时间、可变跳跃高度、冲刺、蓄力攻击、连击输入等动作游戏核心机制。

[![Unity](https://img.shields.io/badge/Unity-2022.3.62f1c1-57B9E7?logo=unity)](https://unity.com/)
[![URP](https://img.shields.io/badge/Render%20Pipeline-URP-8B5CF6)](https://unity.com/srp/universal-render-pipeline)
[![Input](https://img.shields.io/badge/Input%20System-1.14.2-34D399)]()
[![FSM](https://img.shields.io/badge/Pattern-Finite%20State%20Machine-FF6B6B)]()

---

## 📖 项目简介

本项目是一个有限状态机（Finite State Machine）的学习与实践工程，目标是搭建一套可扩展、低耦合的角色状态管理框架，并在此基础上实现 2D 横版动作游戏的核心玩法。

项目采用**双状态机并行**设计：
- **移动状态机**：管理 Idle / Walk / Dash / Jump / Fall 等移动状态
- **攻击状态机**：管理 NormalAttack / ChargeUp / ChargeAttack 等战斗状态

两套状态机通过动画事件桥接协同工作——攻击时移动状态机进入 Null 状态锁定移动，攻击结束后根据输入自动恢复到 Idle 或 Walk。

---

## 🏗️ 状态机架构

### 核心接口

```csharp
public interface IState
{
    void Enter();                       // 进入状态
    void Exit();                        // 退出状态
    void HandInput();                   // 处理输入
    void Update();                      // 每帧更新
    void OnAnimationTranslateEvent(IState state);  // 动画转场事件
    void OnAnimationExitEvent();        // 动画退出事件
}
```

### 状态机基类

```csharp
public class StateMachine
{
    public BindableProperty<IState> currentState = new BindableProperty<IState>();

    public void ChangeState(IState nextState)
    {
        currentState.Value?.Exit();
        currentState.Value = nextState;
        currentState.Value?.Enter();
    }
    // HandInput / Update / 动画事件均委托给当前状态
}
```

**设计要点：**
- 状态实例在状态机构造时一次性创建，运行时切换不产生 GC
- `currentState` 使用 `BindableProperty` 包装，可响应状态变化
- 状态切换遵循严格的 `Exit → Enter` 顺序

### 移动状态机

```
                    ┌──────────┐
            ┌──────▶│   Idle   │◀──────────┐
            │       └────┬─────┘           │
            │            │ 有方向输入       │ 速度归零
            │            ▼                 │
            │       ┌──────────┐           │
            │       │   Walk   │───────────┘
            │       └────┬─────┘
            │ 冲刺键      │ 离开地面 & 下落
            │            ▼
            │       ┌──────────┐
            │       │   Dash   │
            │       └────┬─────┘
            │            │ 动画结束
            │            ▼
            │    (回到 Idle/Walk/Fall)
            │
     跳跃键 │       ┌──────────┐
            └──────▶│   Jump   │
                    └────┬─────┘
                         │ 速度≤0
                         ▼
                    ┌──────────┐
                    │   Fall   │──落地──▶ Idle/Walk
                    └──────────┘
```

| 状态 | 进入条件 | 退出条件 | 关键行为 |
|------|----------|----------|----------|
| **Idle** | 落地且无输入 | 有移动输入 / 冲刺 / 跳跃 | 注册输入回调 |
| **Walk** | 有方向输入且在地面 | 输入归零 / 离地 / 冲刺 | 平滑加减速、角色转向 |
| **Dash** | 按下冲刺键 | 冲刺动画结束 | 瞬间施加冲刺速度、禁止跳跃 |
| **Jump** | 跳跃缓冲 + 土狼时间判定通过 | 上升速度≤0 | 短按/长按不同跳跃力度 |
| **Fall** | 离地且速度向下 | 落地 | 空中控制 |
| **Null** | 进入攻击状态 | 攻击结束 | 锁定移动输入 |

### 攻击状态机

```
              ┌──────────────┐
       ┌─────▶│     Null     │◀────────────┐
       │      └──────┬───────┘             │
       │             │ 攻击键              │ 动画结束 & 无新指令
       │             ▼                     │
       │      ┌──────────────┐             │
       │      │ NormalAttack │─────────────┘
       │      └──────┬───────┘
       │             │ 长按攻击键
       │             ▼
       │      ┌──────────────┐
       │      │  ChargeUp    │
       │      └──────┬───────┘
       │             │ 蓄力完成(≥阈值)
       │             ▼
       │      ┌──────────────┐
       │      │ ChargeAttack │─────────────┘
       │      └──────────────┘
       │
  空中+上+攻击 ──▶ UpAttack（通过 Animator CrossFade 直接切换）
```

| 状态 | 进入条件 | 退出条件 | 关键行为 |
|------|----------|----------|----------|
| **Null** | 默认 / 攻击结束 | 攻击输入 | 不做任何事 |
| **NormalAttack** | 短按攻击键 | 动画结束 | 触发 Attack 动画、记录连击指令 |
| **ChargeUp** | 长按攻击键≥阈值 | 蓄力完成 | 进入 ChargUp 动画、累计蓄力时间 |
| **ChargeAttack** | 蓄力完成后松开 | 动画结束 | 触发 ChargeAttack 动画、高伤害攻击 |

---

## 🎮 已实现机制

### 移动系统

| 机制 | 说明 | 参数 |
|------|------|------|
| **平滑加减速** | 速度向目标值线性插值，区分加速度/减速度 | acceleration=20, deceleration=20 |
| **跳跃缓冲** | 落地前提前按跳跃，落地后自动起跳 | jumpBufferTime=0.2s |
| **土狼时间** | 离开平台后短时间内仍可跳跃 | coyoteTime=0.15s |
| **可变跳跃高度** | 短按小跳、长按大跳；松开跳跃键立即削减上升速度 | shortJump=12, longJump=18, threshold=0.3s |
| **冲刺** | 按当前速度方向或朝向施加冲刺速度，冲刺中禁止跳跃 | DashSpeed=80 |
| **角色转向** | 根据水平速度方向翻转 Scale，带阈值防抖 | facingThreshold=0.1 |

### 战斗系统

| 机制 | 说明 |
|------|------|
| **普通攻击** | 点击攻击键触发，支持连击输入缓冲 |
| **蓄力攻击** | 长按攻击键进入蓄力，达到阈值后松开放出重击 |
| **上挑攻击** | 空中按上方向 + 攻击触发 UpAttack |
| **敌人检测** | 攻击时在玩家前方圆形区域检测 Tag 为 "Enemy" 的对象 |
| **攻击可视化** | Scene 视图中用 Gizmos 绘制检测范围（可开关） |

### 动画驱动

- 使用 `StateMachineBehaviour`（`OnAnimationTranslation`）挂载在动画状态上
- 动画进入时回调 `Player.OnAnimationTranslateEvent()`，由 Player 分发到对应状态机
- 动画退出时回调 `OnAnimationExitEvent()`，状态自行决定转移目标
- 实现了**动画与逻辑解耦**——状态切换时机由动画时间线控制，而非硬编码计时

---

## 📂 项目结构

```
Practice-in-StateMachine/
├── Assets/
│   ├── Scripts/
│   │   ├── FSM/
│   │   │   ├── StateMachine/
│   │   │   │   ├── IState.cs              # 状态接口
│   │   │   │   └── StateMachine.cs        # 状态机基类
│   │   │   └── Charactors/Player/
│   │   │       ├── Player.cs              # 玩家主控制器（双状态机协调）
│   │   │       ├── Data/
│   │   │       │   ├── PlayerReusableData.cs       # 玩家全局数据
│   │   │       │   └── States/
│   │   │       │       ├── PlayerMoveReusableData.cs   # 移动配置数据
│   │   │       │       └── PlayerComboReusableData.cs  # 攻击配置数据
│   │   │       └── StateMachine/
│   │   │           ├── OnAnimationTranslation.cs   # 动画事件桥接（StateMachineBehaviour）
│   │   │           ├── OnAnimationExitEvent.cs     # 动画退出事件
│   │   │           ├── MoveMent/
│   │   │           │   ├── PlayerMoveMentStateMachine.cs
│   │   │           │   ├── PlayerMoveMentState.cs       # 移动状态基类
│   │   │           │   ├── PlayerNullState.cs
│   │   │           │   └── States/
│   │   │           │       ├── GroundedStates/
│   │   │           │       │   ├── PlayerIdleingState.cs
│   │   │           │       │   └── Moving/
│   │   │           │       │       ├── PlayerWalkingState.cs
│   │   │           │       │       └── PlayerDashState.cs
│   │   │           │       └── AirborneStates/
│   │   │           │           ├── PlayerJumpingState.cs
│   │   │           │           └── PlayerFallingState.cs
│   │   │           └── Combo/
│   │   │               ├── PlayerComboStateMachine.cs
│   │   │               ├── PlayerAttackStateBase.cs      # 攻击状态基类
│   │   │               ├── CharactorComboBase.cs         # 连击逻辑 & 敌人检测
│   │   │               └── States/
│   │   │                   ├── PlayerNormalAttack.cs
│   │   │                   ├── PlayerChargeUpState.cs
│   │   │                   ├── PlayerChargeAttackState.cs
│   │   │                   ├── playerUpAttack.cs
│   │   │                   └── PlayerComboNullState.cs
│   │   ├── Character/Base/
│   │   │   └── CharacterControllerBase.cs   # 角色基类（地面检测、Rigidbody2D、Animator）
│   │   ├── Input/
│   │   │   ├── CharacterInputs.cs           # Input Action 资产生成类
│   │   │   └── CharactorInputSystem.cs      # 输入系统单例（Move/Jump/Dash/Attack）
│   │   ├── Managers/
│   │   │   ├── GameBlackboard.cs            # 游戏黑板（敌人列表、玩家引用）
│   │   │   └── EventManager/                # 事件管理器
│   │   ├── Animator/
│   │   │   └── AnimatorID.cs                # Animator 参数哈希缓存
│   │   ├── Tool/BindableProperty/
│   │   │   └── BindableProperty.cs          # 可绑定属性（值变化回调）
│   │   ├── Utils/
│   │   │   └── AnimationStateChecker.cs     # 动画状态查询工具
│   │   └── common/patterns/
│   │       ├── Singleton/                   # 单例基类（Mono & 非Mono）
│   │       └── DevelopmentTool/             # 调试日志工具
│   ├── Scenes/
│   │   └── SampleScene.unity                # 主测试场景
│   ├── Animation/                           # Animator Controller & 动画剪辑
│   ├── Textures/                            # 角色与场景贴图
│   ├── Ilumisoft/Health System/             # 第三方血量系统插件
│   └── Settings/                            # URP 渲染设置
├── Packages/manifest.json
└── ProjectSettings/
```

---

## 🔧 技术要点

### 1. 双状态机协同

```csharp
// Player.cs 中每帧同时驱动两套状态机
movemenStateMachine.HandInput();
movemenStateMachine.Update();
combomenStateMachine.HandInput();
combomenStateMachine.Update();
```

攻击开始时，移动状态机切换到 `moveNullState`（锁定移动）；攻击结束时，攻击状态机的 `OnAnimationExitEvent` 主动将移动状态机切回 `idlingState` 或 `walkingState`。

### 2. 动画事件驱动状态切换

```
Animator 状态进入 → OnAnimationTranslation.OnStateEnter()
  → Player.OnAnimationTranslateEvent(stateEnum)
    → 移动状态机.ChangeState(对应状态)
    → 攻击状态机.ChangeState(对应状态)
```

状态切换时机完全由动画时间线控制，避免了代码中硬编码 `yield return new WaitForSeconds()`。

### 3. 可复用数据模式

将状态运行时数据从状态类中抽离到 `PlayerMoveReusableData` / `PlayerComboReusableData`，在 Inspector 中可配置，状态之间共享同一份数据，避免了状态切换时的数据丢失。

### 4. 输入系统

使用 Unity 新 Input System（`InputSystem` 包），通过 `CharacterInputs` 动作映射支持键盘/手柄。输入查询封装在 `CharactorInputSystem` 单例中，提供 `WasPressedThisFrame` / `IsPressed` / `WasReleasedThisFrame` 等细粒度查询。

---


## 📈 开发日志

| 日期 | 提交 | 内容 |
|------|------|------|
| 2025-09-24 | 框架和状态机搭建 | FSM 核心框架、IState、StateMachine 基类 |
| 2025-09-25 | MoveStateMachine | 移动状态机（Idle/Walk/Jump/Fall） |
| 2025-09-26 | Test | 集成测试 |
| 2025-09-28 | 冲刺Trigger无法重置的bug | 修复冲刺动画 Trigger 不重置的问题 |
| 2025-09-28 | PlayerMove | 完善玩家移动逻辑 |
| 2025-09-28 | EnemyDetect | 敌人检测系统 |

---

## 👤 开发者

- **zrcheng** — 架构设计与程序开发


