# Agent Brief: UE5.5 MetaHuman 口型同步插件（Lip‑Sync）

> 面向“代码智能体（Codex）”的项目工作说明书，用于在本仓库内持续产出插件代码、测试与文档。**目标**：在 UE 5.5 中实现一个跨平台（Win/macOS）可用、既支持离线高精度、又支持近实时的 MetaHuman 口型同步插件。

---

## 0. 智能体角色与总体目标

**你的角色**：项目内的“资深 UE 插件工程师 + 构建与测试机器人”。

**你要达成的事**：

1. 创建名为 **MetaLipSync** 的 UE 插件（运行时模块 `MetaLipSync` + 编辑器模块 `MetaLipSyncEditor`）。
2. 支持两条流程：

   * **离线精确流程**（导入音频 → 语音强制对齐/音素识别 → 生成曲线与关键帧 → Sequencer 轨道）
   * **近实时流程**（播放时做轻量音素/双唇闭合检测 → 驱动 ARKit/Curve 名称映射 → 驱动 MetaHuman）
3. 输出的动画数据可直接驱动 **MetaHuman 面部系统**：

   * 支持 **ARKit 52 blendshape 曲线命名**约定（与 Live Link Face 一致）。
   * 支持 **Control Rig/Face AnimBP** 接口：把曲线注入 `Pose`/`Curve` 或通过 `EvaluateGraphExposedInputs` 写入。
4. 在 **Sequencer** 中提供 **“LipSync Track”**：能绑定音频、可视化音素、允许手工微调关键帧与强度。
5. 提供 **Blueprint 与 C++ API**，以及可选的 **命令行批处理**。

---

## 1. 非目标（Out of Scope）

* 不实现真人实时摄像头的人脸捕捉（已有 Live Link Face）。
* 不造新的人脸骨骼/重拓扑，仅复用 MetaHuman 标准。
* 不写云端服务，所有推理在本地或可选的 ONNX Runtime。

---

## 2. 用户故事（优先级从高到低）

* **US1**（动画师）：我导入一段 WAV，点击“分析”，得到可编辑的音素与嘴形曲线，拖到 Sequencer 里直接驱动 MetaHuman。
* **US2**（原画/previz）：我把音频挂在角色上，运行时即可看到“基本准确”的嘴型跟随（< 120ms 延迟）。
* **US3**（技术美术）：我能打开“映射表”，把音素 → ARKit 曲线的权重自定义并保存为 Preset。
* **US4**（工程）：我能用命令行批量把一堆音频转成动画资产，进 CI。

---

## 3. 架构概览

### 3.1 模块

* **Runtime：`MetaLipSync`**

  * `ULipSyncComponent`：挂到角色/MetaHuman 上，运行时驱动曲线。
  * `ULipSyncSubsystem`（Engine Subsystem）：模型加载、音素识别、Preset 管理。
  * `ULipSyncAsset`（UObject/DataAsset）：存储离线分析得到的**时间帧 → 曲线名/数值**。
  * `FPhonemeMapper`：音素 → ARKit 曲线权重混合。
  * `FLiveDriver`：把曲线写入 Anim Instance / Control Rig。
* **Editor：`MetaLipSyncEditor`**

  * 导入器与面板（音频→LipSyncAsset）、曲线可视化、Sequencer Track、Preset 编辑器。

\$1

### 3.3 可插拔后端接口 `IMetaLipSyncBackend`

* 通过 **Modular Features** 暴露后端：`IMetaLipSyncBackend`，支持 *Rhubarb*、*ONNX* 以及未来自研后端热插拔。
* Editor 提供“后端选择”与“语言包选择”。
* 运行时可在不重启的情况下切换后端（必要时做冷启动预热）。

```cpp
class IMetaLipSyncBackend : public IModularFeature {
public:
  virtual FName GetBackendName() const = 0;                 // "Rhubarb" / "ONNX" / "Builtin"
  virtual bool SupportsLanguage(FName Lang) const = 0;       // zh-CN / en-US ...

  // 离线：音频(+可选台本) → viseme 时序
  virtual bool Analyze(const struct FLipSyncAnalyzeRequest& In,
                       struct FLipSyncAnalyzeResult& Out) = 0;

  // 近实时：特征帧 → 曲线
  virtual void InferRealtime(const struct FLipSyncRealtimeInput& In,
                             struct FLipSyncCurves& Out) = 0;
};
```

> 默认提供 `MetaLipSyncBackend_Rhubarb` 与 `MetaLipSyncBackend_Onnx` 两个实现模块，自研后端命名为 `MetaLipSyncBackend_Builtin`（留空壳待接）。

---

## 4. 与 MetaHuman 的对接

* **曲线命名**遵循 ARKit：例如 `jawOpen`, `mouthClose`, `mouthFunnel`, `mouthPucker`, `mouthSmile_L/R`, `mouthDimple_L/R`, `mouthFrown_L/R`, `mouthRollLower`, `mouthLowerDown_L/R`, `mouthUpperUp_L/R`, `tongueOut`…（完整 52 项在代码中常量化）。
* **默认嘴形集合（Viseme）**：`REST / AI / E / U / O / FV / L / MBP / WQ / CHJ / TH / RR`
* **示例映射（可在 Preset 中调整）**：

  * `MBP` → `mouthClose:1.0`, `jawOpen:0.05`
  * `FV` → `mouthFunnel:0.7`, `mouthPucker:0.3`
  * `AI` → `jawOpen:0.6`, `mouthSmile_L/R:0.2`
  * `L` → `jawOpen:0.4`, `tongueOut:0.4`
  * 平滑：`attack=35ms`, `release=65ms`, `lookAhead=20ms`（可配）。

---

## 5. 公开 API 设计

### 5.1 C++（Runtime）

```cpp
// MetaLipSync/Public/LipSyncComponent.h
UCLASS(ClassGroup=(Audio), meta=(BlueprintSpawnableComponent))
class METALIPSYNC_API ULipSyncComponent : public UActorComponent {
  GENERATED_BODY()
public:
  UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="LipSync")
  TObjectPtr<USkeletalMeshComponent> TargetSkeletalMesh;

  UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="LipSync")
  TObjectPtr<class ULipSyncPreset> Preset;

  UFUNCTION(BlueprintCallable, Category="LipSync")
  void PlayFromAsset(ULipSyncAsset* Asset, float StartTime = 0.f, bool bLoop=false);

  UFUNCTION(BlueprintCallable, Category="LipSync")
  void Stop();

  UFUNCTION(BlueprintCallable, Category="LipSync")
  void SetRealtimeEnabled(bool bEnable);
};
```

```cpp
// MetaLipSync/Public/LipSyncSubsystem.h
UCLASS()
class METALIPSYNC_API ULipSyncSubsystem : public UEngineSubsystem {
  GENERATED_BODY()
public:
  UFUNCTION(BlueprintCallable, Category="LipSync")
  ULipSyncAsset* AnalyzeAudio(const FString& AudioFilePath, const ULipSyncPreset* Preset);

  UFUNCTION(BlueprintCallable, Category="LipSync")
  void SetModelBackend(ELipSyncBackend Backend); // Rhubarb / ONNX
};
```

### 5.2 Blueprint 节点

* `Analyze Audio To LipSync Asset`
* `Play LipSync Asset`
* `Enable Realtime LipSync`

### 5.3 资产结构

```cpp
USTRUCT(BlueprintType)
struct FLipSyncFrame {
  GENERATED_BODY();
  UPROPERTY(EditAnywhere) float Time; // 秒
  UPROPERTY(EditAnywhere) TMap<FName, float> Curves; // 曲线名 → 归一化值 0..1
};

UCLASS(BlueprintType)
class ULipSyncAsset : public UDataAsset {
  GENERATED_BODY();
public:
  UPROPERTY(EditAnywhere) TArray<FLipSyncFrame> Frames;
  UPROPERTY(EditAnywhere) float Duration;
};
```

---

### 5.4 语言包与曲线映射资产

* **ULipSyncLanguagePack**：描述语言的音素/viseme 集、G2P（可引用外部字典）、共发音（Coarticulation）权重表。
* **UCurveRemapAsset**：ARKit 52 曲线 ↔ 项目曲线命名/强度重映射，便于兼容不同 Face AnimBP。

```cpp
USTRUCT(BlueprintType)
struct FVisemeDef {
  GENERATED_BODY();
  UPROPERTY(EditAnywhere) FName Name;               // AI / E / MBP ...
  UPROPERTY(EditAnywhere) TArray<FName> Phonemes;   // ARPAbet 或拼音
  UPROPERTY(EditAnywhere) TMap<FName,float> Curves; // ARKit曲线→权重
};

UCLASS(BlueprintType)
class ULipSyncLanguagePack : public UDataAsset {
  GENERATED_BODY();
public:
  UPROPERTY(EditAnywhere) FName Language = TEXT("zh-CN");
  UPROPERTY(EditAnywhere) TArray<FVisemeDef> Visemes;
  UPROPERTY(EditAnywhere) TMap<FName,float> Coarticulation; // bigram 权重表，可选
};

UCLASS(BlueprintType)
class UCurveRemapAsset : public UDataAsset {
  GENERATED_BODY();
public:
  UPROPERTY(EditAnywhere) TMap<FName,FName> NameRemap; // ARKit→Project
  UPROPERTY(EditAnywhere) TMap<FName,float> Gain;      // 每曲线增益
};
```

## 6. 编辑器与 Sequencer 集成

* **导入器**：右键音频 → `Create LipSync Asset` → 选择 Preset → 生成 `ULipSyncAsset`。
* **曲线面板**：显示音素区间（彩条）与曲线强度（折线），可框选批量偏移/缩放。
* **Sequencer 轨道**：`LipSync Track` 支持：

  * 绑定 `ULipSyncAsset` 与声音 Cue
  * 关键帧编辑、吸附到音素边界
  * “重算选中区段”按钮

---

## 7. 插件骨架与构建

### 7.1 `.uplugin`

```json
{
  "FileVersion": 3,
  "Version": 1,
  "VersionName": "0.1.0",
  "FriendlyName": "MetaLipSync",
  "Description": "Lip-sync for MetaHuman with offline and realtime pipelines.",
  "Category": "Animation",
  "Modules": [
    { "Name": "MetaLipSync", "Type": "Runtime", "LoadingPhase": "Default" },
    { "Name": "MetaLipSyncEditor", "Type": "Editor", "LoadingPhase": "Default" }
  ],
  "SupportedTargetPlatforms": ["Win64", "Mac"],
  "CanContainContent": true,
  "EnabledByDefault": true
}
```

### 7.2 `Build.cs` 依赖（示例）

```csharp
PublicDependencyModuleNames.AddRange(new[] {
  "Core", "CoreUObject", "Engine", "Projects", "AnimGraphRuntime",
  "LiveLinkInterface", "AudioMixer", "SignalProcessing", "DeveloperSettings"
});

PrivateDependencyModuleNames.AddRange(new[] {
  "Slate", "SlateCore", "UnrealEd", "Sequencer", "MovieScene"
});
```

---

## 8. 第三方后端与许可证

* **Rhubarb Lip Sync**（MIT）：默认离线音素识别，支持中/英（以音素为主）。
* **ONNX Runtime**（MIT）：可选近实时小模型。
* 外部可执行/模型放置在 `ThirdParty/`，首次构建自动下载（校验 hash）。

---

## 9. 预置（Preset）与映射表

`ULipSyncPreset` 保存：

* 语言/发音集（`en-US`, `zh-CN` 等）
* Viseme → ARKit 曲线权重（0..1）
* 平滑参数：`attack`, `release`, `lookAhead`, `curveSmoothing`
* 口型强度全局缩放：`GlobalMouthGain`

**语言包**：`ULipSyncLanguagePack`，为每种语言定义 viseme 与共发音表；路径：`/Content/MetaLipSync/Languages/`。

**曲线重映射**：`UCurveRemapAsset`，适配不同项目的曲线命名与力度；路径：`/Content/MetaLipSync/Remaps/`。

文件：`/Content/MetaLipSync/Presets/Default_ARKIT.asset`

---

## 10. 质量门槛与验收

* **对齐误差**：离线口型边界误差 **P50 ≤ 35ms / P90 ≤ 70ms**（带台本对齐时）。
* **实时延迟**：端到端 **≤ 120ms**（含平滑）；抖动 ≤ ±20ms。
* **共发音平滑度**：相邻 viseme 过渡的最大二阶导数阈值（无可见跳变）；人工抽检 ≥ 95% 片段无“弹跳”。
* **性能**：1080p PIE 中 **CPU < 2ms**（平均），内存 < 200MB（含模型）。
* **资产体积**：Cook 后较原曲线直存 **≥ 50%** 体积下降。
* **稳定性**：播放/暂停/Seek/循环无崩溃，无资源泄露（`-trace=memleak` 通过）。
* **回归测试**：Golden 音频 10 段，曲线哈希一致；CI 报告误差/性能曲线。

---

## 11. 目录结构

```
/Plugins/MetaLipSync/
  MetaLipSync.uplugin
  Source/
    MetaLipSync/
      MetaLipSync.Build.cs
      Public/ ...
      Private/ ...
    MetaLipSyncEditor/
      MetaLipSyncEditor.Build.cs
  Content/
    Presets/
  ThirdParty/
    Rhubarb/
    OnnxRuntime/
  Tests/
    Golden/
```

---

## 12. 代码风格与提交规范

* C++17，UE 风格（成员 `PascalCase`，局部 `camelCase`，指针 `TObjectPtr`）。
* 每次提交带 **变更日志**与 **测试结果**摘要；PR 需含动图/截图。
* 关键模块必须含 **`checkf`** 与 **早返回**。

---

## 13. 命令行与 CI 钩子

* **命令行**

  * `UECmd.exe -MetaLipSyncAnalyze -Input="X.wav" -Preset="Default_ARKIT" -Out="X_LipSync.uasset" \\  --language-pack zh-CN --remap ProjectFace --backend Rhubarb --transcript X.txt --export-json X.json`
* **GitHub Actions**：

  * 缓存 `ThirdParty/`
  * 构建 Win64/Mac Editor
  * 运行单元与回归测试，产出报告（误差/CPU/内存）与样例资产

---

## 14. 示例用法

### 14.1 Blueprint

* 角色蓝图中添加 `LipSyncComponent` → `Analyze Audio To LipSync Asset` → `Play LipSync Asset`。

### 14.2 C++

```cpp
auto* Subsystem = GEngine->GetEngineSubsystem<ULipSyncSubsystem>();
ULipSyncAsset* Asset = Subsystem->AnalyzeAudio(TEXT("/Game/VO/line_01.wav"), Preset);
LipSyncComponent->PlayFromAsset(Asset);
```

---

## 15. 风险与对策

* **多语言音素集差异** → 通过 Preset 抽象 Viseme 层；允许每语言自定义映射。
* **模型体积/授权** → 仅内置 MIT/Apache 许可后端；商业模型需用户自带。
* **MetaHuman 版本兼容** → 曲线名常量集中定义 + 适配层（`FArkitCurveNames`）。

---

## 16. 立即任务清单（给 Codex）

**阶段一（Batch A）交付为主**：

* [ ] 搭建 `IMetaLipSyncBackend` 接口与后端选择 UI（Rhubarb/ONNX 两实现）。
* [ ] 语言包 \`ULipSyncLanguagePack
