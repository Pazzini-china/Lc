---
tags:
  - PLC
  - 状态机
  - SCL
  - ISA-88
created: 2026-10-23
---

# 状态模型实践：从 17 个裁到 5 个

> 不必每个项目都用 17 个状态。标准状态模型的价值是**统一词汇**，不是**统一数量**。按项目裁剪，但裁剪结果要写进项目规范。

---

## 1 先决定：这个项目需要几个状态

一个状态只有在满足下面**任一条**时才保留：

1. HMI 上要显示不同内容
2. 需要不同的输出映射
3. 出错/停止时的恢复动作不同

三条都不满足 → 合并掉。

### 三档状态集

| 档 | 状态 | 适用场景 | 状态数 |
|---|---|---|---|
| **A 最小** | 待机 / 启动中 / 运行 / 停止中 / 故障 | 单工位、简单机构（气缸、单轴、泵、风机） | 5 |
| **B 标准** | A + 收尾中 / 已完成 / 已停止 | 有循环或批次结束概念的工位、EM 层 | 8 |
| **C 完整** | B + 保持组 + 暂停组 + 中止组 | 过程单元、上下游联动、接 MES | 17 |

**建议分工**：
- CM 层用 **A**
- EM 层用 **A 或 B**
- Unit 层按需上 **C**

### 与标准状态的对应

| 本标准名 | PackML 对应 | 是否常裁剪 |
|---|---|---|
| 待机 | Idle | 保留 |
| 启动中 | Starting | 保留（很重要，见第 8 节） |
| 运行 | Execute | 保留 |
| 停止中 | Stopping | 保留 |
| 故障 | Aborted / Fault | 保留 |
| 收尾中 / 已完成 | Completing / Complete | 按需 |
| 已停止 / 复位 | Stopped / Resetting | 按需，简单设备可直接回待机 |
| 保持组 | Holding / Held / Unholding | 有暂停需求才加 |
| 暂停组 | Suspending / Suspended / Unsuspending | 上下游联动才加 |
| 中止组 | Aborting / Aborted / Clearing | 需要作废当前批次才加 |

---

## 2 状态号约定

全项目一套，禁止各块自定义。十位留空便于插入新状态。

| 号 | 状态 | 语义 |
|---|---|---|
| 0 | 待机 Idle | 输出全复位，可接受启动 |
| 10 | 启动中 Starting | 命令已发，等待条件成立 |
| 20 | 运行 Execute | 正常执行功能 |
| 30 | 停止中 Stopping | 停止已发，等待回到安全位 |
| 90 | 故障 Fault | 有故障，输出复位，等待复位 |

B 档可加：40 收尾中、50 已完成、60 已停止。
建议把状态号做成**常量**（如 `cStateIdle := 0`）或全局常量 DB，不要在代码里散落魔法数字。

---

## 3 程序骨架：四段式

> 说明：这个"四段式"是**工程写法归纳**，不是标准原文里的术语。ISA-88 / PackML 只规定状态语义，不规定程序怎么分区；西门子 LAD、PCS 7 APL 等库内部有类似的"输入 → 联锁/模式 → 输出 → 诊断"分区，但名称和切分方式因库而异。要严格对齐某个库，以该库块的 region 划分为准。

每个 CM 的 FB 固定按这个顺序写，好处是**只有一处写状态、只有一处写输出**，不会出现竞争。

```
0 输入处理   边沿检测、定时器调用
1 故障汇总   所有故障条件集中在这里，置 fault + faultCode
2 状态迁移   CASE 里只改 state，不写输出
3 输出映射   用 state 推导 Q，只在这里写 Q
4 状态回送   拼 Stat，供 EM / HMI 使用
```

四条硬规则：

- **只有一个地方写 `state`**（第 2 段）
- **只有一个地方写 `Q`**（第 3 段）
- **故障只有一个入口**（第 1 段汇总，统一跳 90）
- **中间状态必须有超时**（启动中、停止中）

---

## 4 完整示例：气缸伸出 CM（SCL）

```scl
FUNCTION_BLOCK "FB_Cyl_Extend"
VAR_INPUT
  Cmd : "Cmd_t";      // Enable / Start / Stop / Reset / Mode
  Par : "Par_t";      // T_Start / T_Stop
END_VAR
VAR_OUTPUT
  Stat : "Stat_t";
  Alm  : "Alm_t";
END_VAR
VAR
  state     : Int := 0;
  rtStart   : R_TRIG;
  rtReset   : R_TRIG;
  tStart    : TON;
  tStop     : TON;
  tNow      : DTL;
  fault     : Bool;
  faultCode : Word;
  sensExt   : Bool;
  sensRet   : Bool;
END_VAR
BEGIN
  // ---------- 0 输入处理 ----------
  rtStart(CLK := Cmd.Start);
  rtReset(CLK := Cmd.Reset);
  sensExt := "I_SensExt";
  sensRet := "I_SensRet";

  // 定时器必须每周期调用，用 IN 控制起停，不能只在某分支里调用
  tStart(IN := (state = 10), PT := Par.T_Start);
  tStop (IN := (state = 30), PT := Par.T_Stop);

  // ---------- 1 故障汇总 ----------
  fault := FALSE;
  faultCode := 16#0000;
  IF sensExt AND sensRet THEN
    fault := TRUE;  faultCode := 16#2001;   // 到位信号矛盾
  END_IF;
  IF tStart.Q THEN
    fault := TRUE;  faultCode := 16#2002;   // 伸出超时
  END_IF;
  IF tStop.Q THEN
    fault := TRUE;  faultCode := 16#2003;   // 缩回超时
  END_IF;

  // ---------- 2 状态迁移（唯一写 state 的地方） ----------
  CASE state OF
    0:   // Idle
      IF rtStart.Q AND (Cmd.Mode = 2) THEN   // 2 = 自动
        state := 10;
      END_IF;

    10:  // Starting
      IF sensExt THEN
        state := 20;
      END_IF;

    20:  // Execute
      IF Cmd.Stop OR NOT Cmd.Enable THEN
        state := 30;
      END_IF;

    30:  // Stopping
      IF sensRet THEN
        state := 0;
      END_IF;

    90:  // Fault
      IF rtReset.Q AND NOT fault THEN
        state := 0;
        Alm.Active := FALSE;
      END_IF;
  END_CASE;

  // 故障统一跳转（唯一入口）
  IF fault AND state <> 90 THEN
    state := 90;
    RD_SYS_T(OUT => tNow);
    Alm.Active    := TRUE;
    Alm.Code      := faultCode;
    Alm.TimeStamp := tNow;
  END_IF;

  // ---------- 3 输出映射（唯一写 Q 的地方） ----------
  "Q_ValveExtend" := (state = 10) OR (state = 20);

  // ---------- 4 状态回送 ----------
  Stat.State     := state;
  Stat.Busy      := (state = 10) OR (state = 30);
  Stat.Done      := (state = 20);
  Stat.Error     := (state = 90);
  Stat.ErrorCode := Alm.Code;
  Stat.Actual    := 0.0;
END_FUNCTION_BLOCK
```

要点解释：

- `rtStart` 用上升沿：否则从停止回到待机后，如果 Start 一直为 1，会立刻重启。
- 定时器无条件调用：TON 只在被调用时计时，写在分支里会漏计时。
- 故障跳转放在 CASE **之后**：保证同一周期内不会既执行状态分支又抱着故障继续跑。
- `Cmd.Mode = 2` 硬编码不推荐，正式项目用常量。

---

## 5 EM 层怎么用

EM 只做三件事：**发命令、读状态、编排步序**。EM 不写 CM 的 `state`，也不直接写 Q。

```scl
// EM 步序：命令/状态分离
CASE step OF
  10:  // 夹紧
    cmdCyl.Start := TRUE;
    IF statCyl.Done THEN
      cmdCyl.Start := FALSE;
      step := 20;
    ELSIF statCyl.Error THEN
      step := 900;
    END_IF;

  20:  // 进给
    cmdAxis.Start := TRUE;
    IF statAxis.Done THEN
      cmdAxis.Start := FALSE;
      step := 30;
    END_IF;

  900: // 异常收尾：全部下停止，等状态回到 Idle
    cmdCyl.Stop  := TRUE;
    cmdAxis.Stop := TRUE;
    IF (statCyl.State = 0) AND (statAxis.State = 0) THEN
      cmdCyl.Stop  := FALSE;
      cmdAxis.Stop := FALSE;
      step := 0;
    END_IF;
END_CASE;
```

EM 自己的状态机也用同一套状态号，只是它管的是"整段流程跑到哪"，不是"机构在哪"。

---

## 6 ModeMgr 怎么接入

1. ModeMgr 每周期算出一个**当前模式**，写进共享数据（或作为输入传给每个 CM）。
2. 每个 CM 在**入口**检查模式，不允许的命令直接忽略。
3. 切换模式前，ModeMgr 先让所有 CM / EM 回 Idle，再切；不允许在运行时直接切。
4. 模式切换要留记录（时间、操作员、原模式、新模式），便于追溯。

约定：**CM 不判断"该不该在这个模式"，只判断"当前模式允不允许我这个命令"。** 模式本身由 ModeMgr 独裁。

---

## 7 HMI 与报警

- PLC 只给**状态号**，HMI 维护"状态号 → 文本 / 颜色"映射表，不要在每个画面里写死逻辑。
- 报警统一通过 `Alm_t` 输出（Active / Class / Code / TimeStamp），HMI 只做展示与确认。
- 错误码分段：`16#2001`~ 机构类、`16#3001`~ 驱动类、`16#4001`~ 通讯类，便于快速定位。

---

## 8 常见坑

| 坑 | 后果 | 对策 |
|---|---|---|
| 命令用电平不用边沿 | 停止后自动重启 | 所有启动类命令做 R_TRIG |
| 多处写 `state` | 状态竞争、偶发卡死 | 只允许第 2 段写 |
| 多处写 `Q` | 输出抖动、故障不断电 | 只允许第 3 段写 |
| 故障复位只看 Reset | 故障未消失又立刻进故障 | 复位条件 = Reset 上升沿 + 故障条件消失 |
| 没有"启动中/停止中" | 命令已发就当作已完成，误判 | 保留中间状态 |
| `Done` 语义漂移 | 有的块表示完成，有的表示运行中 | 写进接口规范并评审 |
| 状态号各块自定义 | 无法统一显示与诊断 | 全局常量表 |
| 定时器写在分支里 | 计时不准、超时不报 | 每周期调用，IN 控制起停 |

---

## 9 落地检查清单

- [ ] 本项目状态集已裁剪并写入规范（A / B / C 哪一档）
- [ ] 状态号全项目统一，无常量散落
- [ ] 每个 FB 只有一个地方写 `state`
- [ ] 每个 FB 只有一个地方写 `Q`
- [ ] 所有中间状态都有超时监控
- [ ] 故障统一入口 + 错误码表
- [ ] `Busy` / `Done` / `Error` 语义写进接口文档并评审
- [ ] HMI 状态映射表已建立
- [ ] ModeMgr 作用域已明确，切换流程已定义
