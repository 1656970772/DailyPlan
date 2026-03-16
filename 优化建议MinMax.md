# UE5 GAS Demo 计划优化建议

> 基于对现有文档的深入分析，结合 Lyra Starter Game、Action Game 等知名 UE5 项目的架构实践，提供以下优化与细化建议。

---

## 一、Phase 0 工程环境搭建 - 优化建议

### 1.1 模块依赖细化

**现状**：文档中统一添加了完整的模块列表。

**优化建议**：

| 模块 | 建议添加原因 | 参考项目 |
|------|-------------|----------|
| `ReplicationGraph` | 为 Phase 4 大规模多人验证做准备 | Lyra、Fortnite |
| `PhysicsCore` | 未来扩展攀爬/物理交互的基础 | - |
| `AnimationLocomotionLibrary` | UE5.4 新增的动画 locomotion 工具库 | UE5.4+ 官方 |

```csharp
// 建议增加的模块
PublicDependencyModuleNames.AddRange(new string[]
{
    // ... 现有模块 ...
    "PhysicsCore",
    "AnimationLocomotionLibrary",
});

// 可选：如果计划做高级复制优化
#if WITH_REPLICATION
    "ReplicationGraph",
#endif
```

### 1.2 GameInstance 初始化增强

**现状**：只调用了 `InitGlobalData()`。

**优化建议**：参考 Lyra 的初始化模式，增加更完整的全局初始化链：

```cpp
// 建议扩展
void UZZCGameInstance::Init()
{
    Super::Init();

    // 1. GAS 全局数据初始化
    UAbilitySystemGlobals::Get().InitGlobalData();

    // 2. 全局 GameplayTag 加载（可选）
    // UGameplayTagManager::Get().LoadGlobalTags();

    // 3. 订阅网络诊断事件（为 Phase 4 准备）
    // FOnNetworkFailureListener::Bind(&UZZCGameInstance::HandleNetworkFailure);
}
```

### 1.3 目录结构优化

**现状**：目录按类型划分。

**优化建议**：增加 `Demo/` 层，与 `Internal/` 区分：

```text
Source/ZZCDemo/
├── Core/                    // 核心框架
├── Character/              // 角色相关
├── GAS/                    // GAS 系统
│   ├── Abilities/         // 技能实现
│   ├── Attributes/        // 属性集
│   ├── Effects/          // 效果
│   ├── Components/        // 组件
│   └── Config/           // ★ 新增：配置数据（DataAsset）
├── UI/                     // UI 系统
├── Network/               // 网络相关
├── Demo/                  // ★ 新增：Demo 特定逻辑
│   ├── Components/        // Demo 专用组件
│   └── Systems/           // Demo 专用系统
└── Editor/                // ★ 新增：编辑器插件
```

---

## 二、Phase 1 3C 系统 - 优化建议

### 2.1 输入系统增强

**现状**：使用基础的 Enhanced Input 映射。

**优化建议**：参考 Lyra 的输入系统，增加输入修饰符和组合输入：

```cpp
// 建议增加的输入能力
enum class EZZCInputModifier
{
    SensitivityScale,      // 敏感度缩放
    DeadZone,              // 死区处理
    Curve,                 // 曲线映射
};

// 组合输入示例：跑动中按下跳跃 = 闪避
struct FComboInputConfig
{
    EZZCInputType FirstInput;     // 第一个输入
    EZZCInputType SecondInput;    // 第二个输入（窗口期内）
    float ComboWindow;            // 组合窗口时间（秒）
};
```

### 2.2 Sprint 预测优化

**现状**：通过 SavedMove 保存 Sprint 状态。

**优化建议**：增加更多的预测状态标志：

```cpp
// 扩展 FZZCSavedMove 建议
struct FZZCSavedMove : public FSavedMove_Character
{
    // 现有
    uint8 bWantsToSprint : 1;

    // ★ 建议新增
    uint8 bWantsToCrouch : 1;      // 蹲伏预测
    uint8 bWantsToAim : 1;        // 瞄准预测
    float AimYawOffset;           // 瞄准偏转（用于预测回弹）

    // Sprint 相关
    float SprintStartTime;         // Sprint 开始时间（用于计算体力消耗）
};
```

### 2.3 相机系统增强

**现状**：探索态固定参数。

**优化建议**：增加相机行为状态机（参考 Fortnite/Valorant）：

```cpp
// 建议增加的相机模式
enum class EZZCCameraMode
{
    Explore,              // 探索模式（当前）
    Combat,                // 战斗模式（锁定目标）
    Sprint,                // 冲刺模式（FOV 变化）
    Cover,                 // ★ 掩体模式（增强项）
    Zipline,               // ★ 滑索模式（增强项）
};

// 相机参数结构
struct FZZCCameraSettings
{
    float TargetArmLength;
    FRotator SocketOffset;
    float FieldOfView;
    float CameraLagSpeed;
    float InterpSpeed;
};
```

### 2.4 添加相机碰撞检测

**现状**：SpringArm 默认碰撞可能不够完善。

**优化建议**：增加自定义碰撞处理：

```cpp
// 建议实现
void UZZCSpringArmComponent::TickComponent(float DeltaTime)
{
    // 1. 基础 SpringArm 行为
    Super::TickComponent(DeltaTime);

    // 2. 添加自定义碰撞检测（防止相机穿墙）
    if (bDoCollisionTest)
    {
        FVector TraceStart = GetComponentLocation();
        FVector TraceEnd = GetTargetLocation();

        // 使用 Trace 处理相机位置
        // ...
    }
}
```

---

## 三、Phase 2 GAS 核心 - 优化建议

### 3.1 AttributeSet 复制优化

**现状**：使用默认复制方式。

**优化建议**：参考 Lyra 的复制策略，优化网络传输：

```cpp
// 建议的 AttributeSet 复制策略
UCLASS()
class UZZCBaseAttributeSet : public UAttributeSet
{
    GENERATED_BODY()

public:
    // 使用 REP_NOTIFY 优化复制
    UPROPERTY(EditAnywhere, BlueprintReadWrite, ReplicatedUsing=OnRep_Health, Category="Attributes")
    float Health = 100.0f;

    UFUNCTION()
    virtual void OnRep_Health(const FGameplayAttributeData& OldValue);

    // ★ 建议增加：属性变更回调（用于通知系统）
    FOnAttributeChanged OnHealthChanged;
};
```

### 3.2 伤害系统增强

**现状**：使用基础的 ExecCalc。

**优化建议**：增加完整的伤害管线（参考 Lyra/Paragon）：

```cpp
// 建议的伤害处理结构
struct FZZCDamageEvaluations
{
    // 基础伤害
    float BaseDamage;

    // 伤害修正
    float DamageMultiplier;     // 伤害乘数
    float CriticalMultiplier;   // 暴击乘数
    float ArmorReduction;      // 护甲减伤

    // 特殊效果
    bool bIsCritical;
    FGameplayTagContainer DamageTags;

    // 伤害来源信息
    FVector HitLocation;
    FVector HitDirection;
};

// 建议增加：伤害类型枚举
enum class EZZCDamageType
{
    Physical,
    Magic,
    True,          // 真实伤害（无视护甲）
    Heal,          // 治疗
};
```

### 3.3 ASC 初始化优化

**现状**：直接在 PlayerState 创建 ASC。

**优化建议**：增加更完整的初始化流程：

```cpp
// 建议的 ASC 初始化顺序
void UZZCAbilitySystemComponent::InitializeWithPlayerState(APlayerState* InPlayerState)
{
    // 1. 基础初始化
    InitAbilityActorInfo(InPlayerState, InPlayerState->GetPawn());

    // 2. 绑定属性变更回调
    BindAttributeChangeCallbacks();

    // 3. 应用默认效果（被动Buff等）
    ApplyDefaultEffects();

    // 4. 初始化能力（从 DataAsset 加载）
    GrantDefaultAbilities();

    // 5. 激活被动能力
    ActivatePassiveAbilities();
}
```

### 3.4 增加 GameplayTag 层级设计

**现状**：文档提到使用 GameplayTags，但未详细设计。

**优化建议**：建立完整的 Tag 层级：

```text
// 建议的 Tag 层级结构
ZZC.                     // 根命名空间
├── Character            // 角色状态
│   ├── State            // 状态标签
│   │   ├── Alive
│   │   ├── Dead
│   │   ├── Stunned
│   │   └── Frozen
│   └── Movement         // 移动标签
│       ├── Sprinting
│       ├── Crouching
│       └── Airborne
├── Ability              // 技能标签
│   ├── Type             // 技能类型
│   │   ├── Melee
│   │   ├── Ranged
│   │   └── Utility
│   ├── Element          // 元素类型
│   │   ├── Fire
│   │   ├── Ice
│   │   └── Lightning
│   └── Combat           // 战斗标签
│       ├── Offensive
│       └── Defensive
└── Damage               // 伤害标签
    ├── Type
    └── Critical
```

---

## 四、Phase 3 技能系统 - 优化建议

### 4.1 Ability 激活预测增强

**现状**：基础 Ability 激活。

**优化建议**：增加更完整的预测链路：

```cpp
// 建议的 Ability 预测接口
class UZZCGameplayAbility : public UGameplayAbility
{
public:
    // ★ 新增：预测激活
    virtual bool CanActivatePrediction() const;

    // ★ 新增：预测执行（客户端立即执行）
    virtual void ExecutePrediction(FGameplayAbilitySpecHandle Handle);

    // ★ 新增：预测确认（服务端确认）
    virtual void ConfirmPrediction();

    // ★ 新增：预测取消（回滚）
    virtual void CancelPrediction(const FGameplayAbilitySpecHandle Handle);

protected:
    // 预测相关数据
    FPredictionData CurrentPredictionData;
};
```

### 4.2 Ability Queue 增强

**现状**：深度 1 的简单缓存。

**优化建议**：参考 Fighting Game 的连招系统，增加更多队列特性：

```cpp
// 建议的队列数据结构
struct FZZCAbilityQueueEntry
{
    FGameplayAbilitySpecHandle AbilityHandle;
    FGameplayTagContainer RequiredTags;
    FGameplayTagContainer BlockedTags;
    float InputHoldDuration;      // 输入持续时间
    float QueueWindow;             // 队列窗口
    EZZCQueuePriority Priority;    // 队列优先级
};

class UZZCAbilityQueueComponent : public UActorComponent
{
public:
    // 队列操作
    bool QueueAbility(FGameplayTag InputTag, float HoldDuration = 0.0f);
    void ProcessQueue();           // 处理队列
    void ClearQueue();

    // 队列配置
    float GlobalQueueWindow;       // 全局队列窗口（默认 0.2s）
    int32 MaxQueueDepth;           // 最大队列深度（建议 2-3）

private:
    TArray<FZZCAbilityQueueEntry> QueuedAbilities;
};
```

### 4.3 DetectionPipeline 增强

**现状**：基础的命中检测。

**优化建议**：增加更复杂的检测选项（参考 Paragon/Lyra）：

```cpp
// 建议的检测配置
enum class EZZCDetectionShape
{
    Sphere,
    Box,
    Capsule,
    Cone,              // ★ 新增：锥形检测
    SingleLine,        // ★ 新增：单线检测
};

struct FZZCDetectionConfig
{
    EZZCDetectionShape Shape;

    // 形状参数
    float Radius;
    FVector BoxExtent;
    float ConeAngle;
    float ConeLength;

    // 过滤条件
    FGameplayTagQuery TargetTagQuery;
    FGameplayTagQuery IgnoreTagQuery;
    ETeamAttitude::Type TeamFilter;

    // 命中处理
    bool bIgnoreSelf;
    bool bAllowMultipleHits;
    float DamageCooldown;          // 伤害冷却（防止单次检测多次命中）
};

// 建议增加：追踪系统（用于弹道类技能）
class UZZCProjectileTrackingComponent : public UActorComponent
{
    FZZCDetectionConfig DetectionConfig;
    TArray<AActor*> HitActors;    // 已命中列表（用于去重）
};
```

### 4.4 技能冷却系统增强

**现状**：未详细展开。

**优化建议**：增加完整的冷却系统：

```cpp
// 建议的冷却数据结构
struct FZZCCooldownInfo
{
    FGameplayTag CooldownTag;
    float RemainingTime;
    float TotalDuration;
    bool bIsInfinite;              // 无限冷却（如某些大招）
};

class UZZCCooldownManager : public UObject
{
public:
    // 开始冷却
    void StartCooldown(FGameplayTag AbilityTag, float Duration);

    // 查询冷却
    float GetCooldownRemaining(FGameplayTag AbilityTag) const;
    bool IsOnCooldown(FGameplayTag AbilityTag) const;

    // 冷却减少（天赋/装备）
    void ReduceCooldown(FGameplayTag AbilityTag, float Reduction);

private:
    TMap<FGameplayTag, FZZCCooldownInfo> ActiveCooldowns;
};
```

---

## 五、Phase 4 网络预测 - 优化建议

### 5.1 诊断工具增强

**现状**：基础的诊断显示。

**优化建议**：参考 Fortnite 的网络诊断系统，增加更完善的工具：

```cpp
// 建议的诊断数据结构
struct FZZCNetworkMetrics
{
    // 时序数据
    double InputTimestamp;
    double PredictionTimestamp;
    double ServerConfirmTimestamp;
    double RollbackTimestamp;

    // 状态数据
    int32 PredictionKey;
    float RTT;                    // 往返时间
    float Jitter;                 // 抖动
    int32 PacketsLost;            // 丢包数

    // 预测数据
    bool bWasRolledBack;
    int32 RollbackCount;
    FVector PredictedPosition;
    FVector CorrectedPosition;
    float PositionError;          // 位置误差
};

// 建议的诊断显示
class UZZCNetworkDiagnostics : public UObject
{
public:
    // 记录指标
    void RecordInput(FGameplayAbilitySpecHandle Handle, const FVector& Position);
    void RecordPrediction(uint32 PredictionKey, const FVector& Position);
    void RecordServerConfirm(uint32 PredictionKey, const FVector& ServerPosition);
    void RecordRollback(uint32 PredictionKey);

    // 获取统计数据
    FZZCNetworkMetrics GetMetrics(uint32 PredictionKey) const;
    float GetAveragePositionError() const;
    int32 GetTotalRollbacksThisSession() const;

    // 可视化
    void DrawDebugHUD(UCanvas* Canvas);
    void ExportToCSV(const FString& FilePath);

private:
    TMap<uint32, FZZCNetworkMetrics> MetricsHistory;
};
```

### 5.2 网络模拟测试自动化

**现状**：手动设置网络模拟。

**优化建议**：增加自动化测试能力：

```cpp
// 建议的网络自动化测试
class UZZCNetworkTestRunner : public UObject
{
public:
    // 预设测试场景
    enum class ETestScenario
    {
        LowLatency_50ms,
        MediumLatency_100ms,
        HighLatency_200ms,
        HighLatencyWithJitter,
        PacketLoss_5Percent,
    };

    // 运行测试
    void RunScenario(ETestScenario Scenario, int32 DurationSeconds);

    // 生成报告
    FString GenerateReport() const;

private:
    struct FTestResult
    {
        float AverageLatency;
        float MaxLatency;
        int32 RollbackCount;
        float PositionErrorAverage;
        float PositionErrorMax;
    };

    TArray<FTestResult> Results;
};
```

### 5.3 预测回滚可视化

**现状**：文字描述。

**优化建议**：增加调试绘制能力：

```cpp
// 建议的调试绘制
void UZZCNetworkDiagnostics::DrawDebugInfo()
{
    // 绘制预测轨迹
    for (const auto& Prediction : PredictedPositions)
    {
        DrawDebugSphere(Prediction.Position, 5.0f, FColor::Green);
    }

    // 绘制服务端确认位置
    DrawDebugSphere(ServerConfirmedPosition, 5.0f, FColor::Red);

    // 绘制回滚轨迹
    if (bWasRolledBack)
    {
        DrawDebugLine(PredictedPosition, CorrectedPosition, FColor::Yellow);
    }

    // 绘制时间轴
    DrawTimeline();
}
```

---

## 六、编辑器增强建议

### 6.1 技能编辑器功能增强

**现状**：基础的 Tag 选择和 Details 编辑。

**优化建议**：增加以下功能：

| 功能 | 说明 | 复杂度 |
|------|------|--------|
| Timeline 可视化编辑 | 拖拽式 Timeline 编辑 | 中 |
| Preview 角色 | 在编辑器中预览技能效果 | 高 |
| 技能流程图 | 可视化技能执行流程 | 中 |
| 数值模拟器 | 在编辑器中模拟伤害/治疗 | 低 |
| 版本对比 | 比较不同版本的技能配置 | 低 |

### 6.2 数据验证增强

**现状**：基础的 Validate + Normalize。

**优化建议**：增加更完整的验证系统：

```cpp
// 建议的验证系统
struct FZZCSkillValidationResult
{
    bool bIsValid;
    TArray<FString> Errors;
    TArray<FString> Warnings;
    TArray<FString> Suggestions;
};

class UZZCSkillValidator
{
public:
    // 验证配置
    FZZCSkillValidationResult ValidateSkill(const UZZCSkillDataAsset* Asset);

    // 验证项
    bool ValidateDamageValues(const FZZCDamageConfig& Config);
    bool ValidateCooldown(float Cooldown);
    bool ValidateTimeline(const FZZCAbilityTimeline& Timeline);
    bool ValidateTagRequirements(const FGameplayTagContainer& Tags);

    // 修复建议
    void AutoFix(FZZCSkillDataAsset* Asset);
};
```

---

## 七、整体架构优化建议

### 7.1 消息系统

**现状**：各系统间耦合较紧。

**优化建议**：增加事件驱动架构：

```cpp
// 建议的事件系统
UCLASS()
class UZZCEventManager : public UObject
{
    DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnHealthChanged, float, NewHealth);
    DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnAbilityActivated, FGameplayAbilitySpecHandle, Handle);
    DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnDamageTaken, float, Damage);

public:
    // 订阅事件
    void Subscribe(AActor* Listener, FOnHealthChanged::FDelegate&& Delegate);

    // 发布事件
    void Publish(AActor* Source, const FOnHealthChanged& Event);

private:
    TMap<FString, TMulticastDelegate*> EventMap;
};
```

### 7.2 配置管理层

**现状**：各系统分散读取配置。

**优化建议**：建立统一的配置管理：

```cpp
// 建议的配置管理器
class UZZCConfigManager : public UGameInstanceSubsystem
{
public:
    // 加载配置
    void LoadAllConfigs();

    // 获取配置
    const UZZCGameplayTagsConfig* GetTagsConfig() const;
    const UZZCDamageTable* GetDamageTable() const;
    const UZZCAbilityDefaults* GetAbilityDefaults() const;

    // 热重载（开发用）
    void ReloadConfig(UClass* ConfigClass);
};
```

### 7.3 日志与调试系统

**现状**：分散的日志输出。

**优化建议**：建立统一的日志系统：

```cpp
// 建议的日志系统
enum class EZZCLogCategory
{
    Default,
    Ability,
    Movement,
    Network,
    Animation,
    UI,
};

class FZZCLog
{
public:
    // 分级日志
    static void Verbose(EZZCLogCategory Category, const FString& Message);
    static void Log(EZZCLogCategory Category, const FString& Message);
    static void Warning(EZZCLogCategory Category, const FString& Message);
    static void Error(EZZCLogCategory Category, const FString& Message);

    // 专用日志
    static void LogAbilityActivation(const FString& AbilityName, bool bSuccess);
    static void LogNetworkMetrics(const FZZCNetworkMetrics& Metrics);
};
```

---

## 八、测试与验证建议

### 8.1 单元测试框架

**建议增加**：使用 UE5 的自动化测试框架：

```cpp
// 建议的测试用例示例
IMPLEMENT_SIMPLE_AUTOMATION_TEST(FZZCAttributeTest, "ZZC.GAS.Attribute", EAutomationTestFlags::ApplicationContextMask | EAutomationTestFlags::ProductFilter)

bool FZZCAttributeTest::RunTest(const FString& Parameters)
{
    // 测试属性设置
    // 测试属性边界
    // 测试属性变化回调
    // 测试复制一致性

    return true;
}
```

### 8.2 集成测试场景

**建议增加**：预置的测试场景清单：

| 测试名称 | 测试内容 | 预期结果 |
|----------|----------|----------|
| Sprint_ZeroLatency | 本地 Sprint 响应 | 即时加速 |
| Sprint_100msLatency | 100ms 延迟 Sprint | 轻微回拉 |
| Sprint_200msLatency | 200ms 延迟 Sprint | 可接受回拉 |
| Ability_Activation | 技能激活预测 | 即时反馈 |
| Damage_Sync | 伤害同步 | 双端一致 |

---

## 九、文档与演示增强建议

### 9.1 代码示例完整性

**建议**：为每个核心类增加完整代码示例：

- `ZZCCharacterMovementComponent` 的完整实现
- `ZZCAbilitySystemComponent` 的初始化流程
- 自定义 `GameplayEffectExecutionCalculation` 的写法

### 9.2 常见问题扩展

**建议增加**：

| 问题 | 原因分析 | 解决方案 |
|------|----------|----------|
| ASC 初始化时机过早 | Pawn 还未完全创建 | 延迟到 PostInitializeComponents |
| 属性复制不同步 | ReplicatedUsing 未正确设置 | 检查 OnRep 回调 |
| 技能无法激活 | 标签不满足 | 检查 AbilityTags 和 ActivationBlockedTags |
| 预测回滚过多 | 客户端/服务端计算不一致 | 统一计算逻辑 |

### 9.3 面试表达材料

**建议增加**：专门用于面试的表达提纲：

```markdown
# 面试表达提纲

## 1. 3C 系统设计
- 如何处理 Sprint 预测
- 相机与角色分离的设计思路

## 2. GAS 架构
- ASC 放在 PlayerState 的原因
- AttributeSet 拆分策略
- 伤害计算管线

## 3. 网络预测
- 预测-确认-回滚机制
- 如何调试预测问题

## 4. 扩展性设计
- 子类化 vs 改源码的选择
- 编辑器扩展思路
```

---

## 十、优先级建议

### 高优先级（建议在当前计划中实现）

1. **GameplayTag 层级设计** - 影响后续所有系统
2. **AttributeSet 复制优化** - 影响联机体验
3. **诊断工具增强** - Phase 4 验证的关键
4. **输入系统增强** - 提升手感

### 中优先级（可在后续迭代中实现）

1. **消息系统** - 降低系统耦合
2. **配置管理层** - 提升可维护性
3. **日志系统** - 提升调试效率
4. **技能编辑器 Preview** - 提升编辑体验

### 低优先级（增强项）

1. **网络自动化测试**
2. **单元测试框架**
3. **版本对比功能**

---

## 参考资料

- Lyra Starter Game (Epic Games)
- Action Game Sample (Epic Games)
- GASDocumentation (tranek)
- Unreal Engine 5.4 Official Documentation
- Fortnite Networked Movement Guide (公开分享)
