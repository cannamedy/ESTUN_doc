# Estun Codroid 远程 API 交接文档

> **文档用途**：本文档供 AI Agent 或开发者快速理解并对接埃斯顿酷卓（Estun Codroid）协作机器人远程控制 API，整合自官方文档 V2.3 及最新版（适配固件 ≥ 2.3.2.6b）。
>
> **原始文档来源**：
> - `Estun Codroid 远程API文档+V2.3.pdf`（基础版，55页）
> - `Estun Codroid 远程模式API手册.pdf`（增强版，含最新接口）
> - Python SDK 示例：https://github.com/Guybod/CodroidApi.git

---

## 目录

1. [通信基础](#1-通信基础)
2. [重要注意事项](#2-重要注意事项)
3. [工程控制接口](#3-工程控制接口)
4. [全局变量接口](#4-全局变量接口)
5. [工程变量接口](#5-工程变量接口)
6. [机器人控制命令](#6-机器人控制命令)
7. [机器人运动控制](#7-机器人运动控制)
8. [机器人计算接口](#8-机器人计算接口)
9. [IO 接口](#9-io-接口)
10. [寄存器接口](#10-寄存器接口)
11. [主题订阅 / 推送接口](#11-主题订阅--推送接口)
12. [远程脚本模式（端口9002）](#12-远程脚本模式端口9002)
13. [CRI 实时控制接口](#13-cri-实时控制接口)
14. [末端 RS485 接口](#14-末端-rs485-接口)
15. [ModbusTCP 主站接口](#15-modbustcp-主站接口)
16. [拖动功能接口](#16-拖动功能接口)
17. [传送带接口](#17-传送带接口)
18. [接口速查表](#18-接口速查表)

---

## 1. 通信基础

### 1.1 连接参数

| 参数 | 值 |
|------|----|
| 协议 | TCP/IP |
| 服务端 | 机器人控制器 |
| 主接口端口 | **9001** |
| 远程脚本模式端口 | **9002** |
| CRI 实时控制（UDP） | 自定义（见第13节） |
| 默认 IP | `192.168.1.136` |
| 数据格式 | JSON 字符串，UTF-8 编码 |

### 1.2 通用请求格式

```json
{
  "id": "请求ID",    // 用户自定义，数字或字符串，响应时原值返回
  "ty": "请求类型",  // 接口名称，如 "Robot/switchOn"
  "db": {}           // 请求数据，无数据时可省略
}
```

### 1.3 通用响应格式

```json
{
  "id": "请求ID",    // 与请求一致
  "ty": "请求类型",  // 与请求一致
  "db": {},          // 返回数据
  "err": {           // 仅失败时存在
    "code": 错误码,
    "msg": "错误描述"
  }
}
```

> **无返回数据的接口**，成功时仅返回 `{"id": ..., "ty": ...}`，不含 `db` 字段。

---

## 2. 重要注意事项

1. **心跳机制**：`Robot/jog`（点动）和 `Robot/moveTo`（RunTo）运动期间，必须每 **500ms** 发送一次心跳请求，否则运动将自动停止。
2. **模式约束**：
   - 手动模式 → 远程模式：需先切换到自动模式，再切换到远程模式（不能直接跳转）。
   - 远程模式 → 手动模式：同上，不能直接跳转。
3. **movC / movCircle**：必须传笛卡尔坐标点，不能使用关节角度点。
4. **Robot/move 的 bug**：如果传入 `"coor": [], "tool": []`（两个空数组），后端会崩溃，请不要传入这两个字段（使用默认值）。
5. **工程ID获取方式**：导出当前脚本，解压 `Project.zip`，`projectlua` 文件夹下的文件夹名即为工程ID。
6. **全局变量**：掉电保存，工程结束后保留运行时的值，不会恢复初始值。
7. **版本要求**：部分接口（模式切换、拖拽模式）仅 **≥ 2.3.2.6** 版本可用，已在各接口中标注。

---

## 3. 工程控制接口

### 3.1 运行脚本

```
接口：project/runScript
功能：客户端直接发送 Lua 脚本并运行
```

```json
// 请求
{
  "id": 1,
  "ty": "project/runScript",
  "db": {
    "scripts": {
      "main": "主程序代码（Lua字符串）",
      "subThreads": {"线程名": "线程代码"},      // 可选
      "subPrograms": {"程序名": "子程序代码"},   // 可选
      "interrupts": {"中断名": "中断程序代码"}   // 可选
    },
    "vars": {                                    // 可选，脚本共用变量
      "v1": 1,
      "v2": "hello"
    }
  }
}
// 响应
{"id": 1, "ty": "project/runScript"}
```

### 3.2 进入远程脚本模式

```
接口：project/enterRemoteScriptMode
功能：进入远程脚本模式（使用9002端口通信）
```

```json
// 请求
{"id": 1, "ty": "project/enterRemoteScriptMode"}
// 响应
{"id": 1, "ty": "project/enterRemoteScriptMode"}
```

### 3.3 按工程ID运行工程（内部接口）

```
接口：project/run
注意：工程ID不对时 web 端弹警告但接口不报错
```

```json
{"id": 1, "ty": "project/run", "db": {"id": "mhv9ubqz0pr69d5f"}}
```

### 3.4 按索引号运行工程

```
接口：project/runByIndex
功能：通过工程映射索引号运行已绑定的工程
```

```json
{"id": 1, "ty": "project/runByIndex", "db": 1}  // db 为 int，非字符串
```

### 3.5 工程运行控制

| 功能 | 接口类型 |
|------|---------|
| 暂停工程 | `project/pause` |
| 恢复工程 | `project/resume` |
| 停止工程 | `project/stop` |

```json
// 以暂停为例（pause/resume/stop 格式相同）
{"id": 1, "ty": "project/pause"}
```

### 3.6 单步运行（内部接口）

```
接口：project/runStep
说明：无工程运行时需传工程ID；工程暂停时不需传ID
     只有主任务单步执行，线程和中断正常运行
```

```json
{"id": 1, "ty": "project/runStep", "db": {"id": "mhv9ubqz0pr69d5f"}}
```

### 3.7 设置启动行

```
接口：project/setStartLine
说明：只对主任务有效，仅生效一次，下次运行后自动清除
```

```json
{"id": 1, "ty": "project/setStartLine", "db": 3}  // db 为行号
```

### 3.8 清除启动行

```json
{"id": 1, "ty": "project/clearStartLine"}
```

> ⚠️ 断点相关接口（`setBreakpoint` / `addBreakpoint` / `removeBreakpoint` / `clearBreakpoint`）**暂时无法使用**。

---

## 4. 全局变量接口

### 4.1 变量类型说明

| 类型 | 格式示例 |
|------|---------|
| 数字 | `100` 或 `2.1` |
| 字符串 | `"\"abcas\""` （需转义双引号）|
| 数组 | `"[2, 4.1, \"abc\"]"` |
| Map | `"{\"jp\": [1,2,3]}"` |

> 变量名须符合 Lua 规范：以字母或下划线开头，禁止使用双下划线开头，禁止使用系统保留关键字（完整列表见原文档3.2节）。

### 4.2 获取全局变量

```
接口：globalVar/getVars
功能：获取所有全局变量
```

```json
// 请求
{"id": 1, "ty": "globalVar/getVars"}

// 响应
{
  "id": 1,
  "ty": "globalVar/getVars",
  "db": {
    "变量名": {
      "val": "变量值（JSON字符串形式）",
      "nm": "变量备注"
    }
  }
}
```

### 4.3 保存（写入/更新）全局变量

```
接口：globalVar/saveVars
功能：增量保存，若变量已存在则更新
```

```json
{
  "id": 1,
  "ty": "globalVar/saveVars",
  "db": {
    "v991": {"val": "100",          "nm": "整数示例"},
    "v992": {"val": "90.4",         "nm": "浮点数示例"},
    "v993": {"val": "[1, 2, 3]",    "nm": "数组示例"},
    "v994": {"val": "{\"a\": 100}", "nm": "Map示例"}
  }
}
```

### 4.4 删除全局变量

```
接口：globalVar/removeVars
说明：删除不存在的变量不报错
```

```json
{"id": 1, "ty": "globalVar/removeVars", "db": ["v991", "v992"]}
```

---

## 5. 工程变量接口

### 5.1 获取当前工程变量

```
接口：globalVar/GetProjectVarUpdate
说明：仅在工程运行状态下有效；不能获取全局变量；不返回注释
```

```json
// 请求
{"id": 1, "ty": "globalVar/GetProjectVarUpdate"}

// 响应
{
  "id": 1,
  "ty": "globalVar/GetProjectVarUpdate",
  "db": {
    "project/point/p1": {          // 点位变量
      "coord": 0,
      "cp": [216.2, -485.345, 436.12, -175.22, 1.657, -68.624],  // 笛卡尔坐标
      "ep": [],
      "rj": [-88.342, 0.882, 89.009, -3.283, 86.059, -109.757],
      "tool": 0
    },
    "project/point/p2": {          // 关节角度点位
      "coord": 0,
      "ep": [],
      "jp": [-88.342, 14.56, 121.328, 42.713, 86.059, -109.757],
      "tool": 0
    },
    "project/var/var1": "0"        // 普通变量
  }
}
```

---

## 6. 机器人控制命令

### 6.1 使能控制

| 功能 | 接口类型 |
|------|---------|
| 上使能 | `Robot/switchOn` |
| 下使能 | `Robot/switchOff` |

```json
{"id": 1, "ty": "Robot/switchOn", "db": ""}
// 响应 db 为 null
```

### 6.2 模式切换（需固件 ≥ 2.3.2.6）

| 目标模式 | 接口类型 | 限制 |
|---------|---------|------|
| 手动模式 | `Robot/toManual` | 不能从远程直接跳转 |
| 自动模式 | `Robot/toAuto` | 无限制 |
| 远程模式 | `Robot/toRemote` | 不能从手动直接跳转 |
| 仿真模式 | `Robot/toSimulation` | 无限制 |
| 实机模式 | `Robot/toActual` | 无限制 |

```json
// 所有模式切换接口格式相同
{"id": 1, "ty": "Robot/toRemote", "db": ""}
```

### 6.3 特殊控制

| 功能 | 接口类型 | 说明 |
|------|---------|------|
| 清除错误 | `System/clearError` | — |
| 救援模式 | `Robot/switchOnRescue` | 异常时使用 |
| 开启拖拽 | `Robot/startDrag` | 仅远程/手动模式，需 ≥2.3.2.6 |
| 关闭拖拽 | `Robot/stopDrag` | 仅远程/手动模式，需 ≥2.3.2.6 |

```json
{"id": 1, "ty": "System/clearError", "db": ""}
```

---

## 7. 机器人运动控制

### 7.1 点动（Jog）

```
接口：Robot/jog
⚠️ 需要每 500ms 发送一次 Robot/jogHeartbeat 心跳
```

```json
{
  "id": 1,
  "ty": "Robot/jog",
  "db": {
    "mode": 2,       // 1=关节点动，2=直线点动
    "speed": -0.1,   // 速度 -1~1（正负代表方向）
    "index": 3,      // 关节点动：1~6表示轴1~6；直线点动：依次为x/y/z/a/b/c
    "coorType": 0,   // 0=用户坐标系，1=工具坐标系
    "coorId": 1      // 用户坐标系ID
  }
}
```

### 7.2 点动心跳

```json
{"id": 1, "ty": "Robot/jogHeartbeat", "db": ""}
```

### 7.3 停止点动

```json
{"id": 1, "ty": "Robot/stopJog", "db": ""}
```

### 7.4 RunTo（运动到指定位置）

```
接口：Robot/moveTo
⚠️ 需要每 500ms 发送一次 Robot/moveToHeartbeat 心跳
```

```json
{
  "id": 1,
  "ty": "Robot/moveTo",
  "db": {
    "type": 4,          // 0=Home位, 1=安全位, 2=蜡烛位, 3=打包位,
                        // 4=关节规划到指定位置, 5=直线规划到指定位置, 6=程序恢复点
    "target": {         // type=4或5时有效
      "jp": [0, 0.1, 0.2, 0.3, 0.4, 0.5],  // 关节位置
      "cp": [x, y, z, a, b, c],              // 末端位置
      "ep": []                               // 附加轴
    }
  }
}
```

### 7.5 RunTo 心跳

```json
{"id": 1, "ty": "Robot/moveToHeartbeat"}
```

### 7.6 运动指令（Robot/move）

```
接口：Robot/move
功能：发送运动指令列表，一次发多条可保证指令间过渡连续
说明：指令接收后接口即返回，运动状态需通过订阅实时查看
⚠️ 不要传入空的 coor 和 tool 字段（后端会崩溃）
```

```json
{
  "id": 1,
  "ty": "Robot/move",
  "db": [
    {
      "type": "movJ",     // movJ=关节运动, movL=直线运动,
                          // movC=圆弧运动, movCircle=圆周运动
      "speed": 60,        // 关节运动: °/s；笛卡尔运动: mm/s
      "acc": 150,         // 加速度，单位与speed对应
      "blend": 20,        // 过渡半径，单位 mm
      "circleNum": 2,     // 仅 movCircle 有效，圆周数，默认1
      "targetPoint": {
        "jp": [10, 20, 30, 40, 50, 60],          // 关节角（度），优先级高于cp
        "cp": [100, 200, 300, 10, 20, 30],        // 笛卡尔坐标（mm, °）
        "rj": [10, 20, 30, 40, 50, 60],           // 参考关节角（可选，用于逆解）
        "ep": []                                   // 附加轴（可选）
      },
      "middlePoint": {    // 仅 movC 和 movCircle 有效且必传
        "cp": [100, 200, 300, 10, 20, 30],
        "rj": [10, 20, 30, 40, 50, 60]
      }
      // "coor": [...],  // 可选，用户坐标系，不需要时不传（勿传空数组）
      // "tool": [...],  // 可选，工具坐标系，不需要时不传（勿传空数组）
    }
  ]
}
```

### 7.7 运动暂停 / 恢复 / 停止

| 功能 | 接口类型 |
|------|---------|
| 暂停运动 | `Robot/pause` |
| 恢复运动 | `Robot/resume` |
| 停止运动 | `Robot/stopMove` |

```json
{"id": 1, "ty": "Robot/pause", "db": ""}
```

### 7.8 速度倍率设置

```json
// 手动运动倍率（1~100）
{"id": 1, "ty": "Robot/setManualMoveRate", "db": 70}
// 自动运动倍率（1~100）
{"id": 1, "ty": "Robot/setAutoMoveRate", "db": 70}
```

---

## 8. 机器人计算接口

### 8.1 正解（关节 → 笛卡尔）

```
接口：Robot/apostocpos
```

```json
// 请求
{
  "id": 1,
  "ty": "Robot/apostocpos",
  "db": {
    "jp": [10, 20, 30, 40, 50, 60],          // 关节角，单位 deg
    "coor": [100, 200, 300, 10, 20, 30],      // 用户坐标系
    "tool": [100, 200, 300, 10, 20, 30],      // 工具坐标系
    "ep": []
  }
}
// 响应 db: [x, y, z, a, b, c]  笛卡尔坐标
```

### 8.2 逆解（笛卡尔 → 关节）

```
接口：Robot/cpostoapos
说明：参考关节角默认 [20,20,20,20,20,20]；若返回空，尝试调整 rj
```

```json
// 请求
{
  "id": 1,
  "ty": "Robot/cpostoapos",
  "db": {
    "cp": [100, 200, 300, 40, 50, 60],   // 末端位置，单位 mm, deg
    "rj": [10, 20, 30, 40, 50, 60],      // 参考关节角，单位 deg
    "ep": []                              // 可选
  }
}
// 响应 db: [j1, j2, j3, j4, j5, j6]  单位 deg
```

### 8.3 笛卡尔坐标偏移计算

```
接口：Robot/calculateRelativePose
```

```json
{
  "id": 1,
  "ty": "Robot/calculateRelativePose",
  "db": {
    "pos": [485.447, -256.717, 63.845, 179.968, 2.951, -90.062],  // 当前TCP坐标
    "offset": [10, 0, 0, 0, 0, 0],   // 偏移量 [x,y,z,a,b,c]，mm和度
    "coorType": "tool",               // "user" 或 "tool"
    "posCoor": [...],                 // 可选，当前TCP所在坐标系
    "coor": [0, 0, 0, 0, 0, 0]       // 可选，coorType为user时有效
  }
}
// 响应 db: [x, y, z, a, b, c]  偏移后坐标
```

---

## 9. IO 接口

### 9.1 获取 IO 值

```
接口：IOManager/GetIOValue
IO 类型：DI（数字输入）/ DO（数字输出）/ AI（模拟输入）/ AO（模拟输出）
```

```json
// 请求
{
  "id": 1,
  "ty": "IOManager/GetIOValue",
  "db": [
    {"type": "DI", "port": 0},
    {"type": "DO", "port": 10},
    {"type": "AI", "port": 1},
    {"type": "AO", "port": 2}
  ]
}
// 响应 db: 同格式，增加 "value" 字段
// DI/DO: 0或1；AI/AO: 浮点数
```

### 9.2 写入 IO 值

```
接口：IOManager/SetIOValue
说明：仅 DO 和 AO 可写入
```

```json
{
  "id": 1,
  "ty": "IOManager/SetIOValue",
  "db": {"type": "DO", "port": 10, "value": 1}
}
```

---

## 10. 寄存器接口

### 10.1 获取寄存器值

```
接口：RegisterManager/GetRegisterValue
```

```json
// 请求
{"id": 1, "ty": "RegisterManager/GetRegisterValue", "db": [10000, 20000]}

// 响应
{
  "id": 1,
  "ty": "RegisterManager/GetRegisterValue",
  "db": [
    {"address": 10000, "value": 0},
    {"address": 20000, "value": 0}
  ]
}
```

### 10.2 写入寄存器值

```
接口：RegisterManager/SetRegisterValue
```

```json
{
  "id": 1,
  "ty": "RegisterManager/SetRegisterValue",
  "db": {"address": 10000, "value": 0}
}
```

### 10.3 扩展数组类型设置

```
接口：RegisterManager/setExtendArrayType
支持类型：Bool / UInt8 / Int8 / UInt16 / Int16 / UInt32 / Int32 / Float32
```

```json
{
  "id": 1,
  "ty": "RegisterManager/setExtendArrayType",
  "db": {"index": 999, "type": "Bool"}  // index: 0-999
}
```

---

## 11. 主题订阅 / 推送接口

### 11.1 订阅方式

```
接口：publish/{topic}
说明：连接到 9001 后发送订阅请求
     数据只在变化时推送，实际推送周期 ≥ 订阅周期
```

```json
// 请求（无需 id）
{"ty": "publish/RobotStatus", "tc": 100}  // tc: 订阅周期 ms，默认0
// 响应持续推送：
{"ty": "publish/RobotStatus", "db": {...}}
```

### 11.2 可用主题列表

#### `publish/ProjectState` — 工程状态

```json
{
  "ty": "publish/ProjectState",
  "db": {
    "id": "工程ID",
    "state": 3,           // 0=空闲, 1=加载中, 2=运行中, 3=暂停
    "isStep": false,      // 是否单步运行
    "projectType": 0,     // 0=普通工程, 1=远程执行脚本, 2=远程脚本模式
    "scripts": {          // state=2或3时有此字段
      "脚本ID": {"line": 8}  // 当前执行行号
    }
  }
}
```

#### `publish/RobotStatus` — 机器人状态

```json
{
  "ty": "publish/RobotStatus",
  "db": {
    "mode": 0,            // 0=手动, 1=自动, 2=远程
    "state": 2,           // 0=未使能, 1=使能中, 2=空闲, 3=点动中, 4=RunTo, 5=拖动中
    "moveRate": 1,        // 自动运行速度倍率
    "manualMoveRate": 0.3,
    "isSimulation": true, // 是否仿真模式
    "ToolId": 0,
    "PayloadId": 0,
    "CoordinateId": 1,
    "type": "S20-180-ECO-V2",  // 机器人型号
    "stateName": "空闲",
    "runDuration": 1374   // 累计运行时间（秒）
  }
}
```

#### `publish/RobotPosture` — 机器人位姿

```json
{
  "ty": "publish/RobotPosture",
  "db": {
    "joint": [3.432, 0.0, 90.0, 0.0, 0.0, 0.0],  // 关节角，单位：度
    "end": {                                         // TCP末端位置
      "x": 203.002, "y": 1206.332, "z": 1066.0,
      "a": 0.0,     "b": 90.0,     "c": 93.432,
      "mode": -1
    },
    "ep": []  // 附加轴
  }
}
```

#### `publish/RobotCoordinate` — 坐标系

```json
{
  "ty": "publish/RobotCoordinate",
  "db": {
    "tool": {"x": 0.0, "y": 0.0, "z": 0.0, "a": 0.0, "b": 0.0, "c": 0.0},
    "user": {"x": 700.0, "y": -770.0, "z": 0.0, "a": 0.0, "b": 0.0, "c": 0.0}
  }
}
```

#### `publish/VarUpdate` — 变量变化推送

```json
{
  "ty": "publish/VarUpdate",
  "db": {
    "global/var/bool1": "34",    // 全局普通变量
    "project/var/dvs": "3",      // 工程普通变量
    "project/point/p1": {...}    // 工程点位变量
  }
}
// key 格式：{定义域}/{类型}/{变量名}
// 定义域：global=全局变量, project=工程变量
// 类型：var=普通变量, point=点变量
```

#### `publish/Log` — 系统日志

```json
{
  "ty": "publish/Log",
  "db": [
    [日志级别, 错误码, 时间戳, "日志内容"],
    [6, 13561, 1760946003.582, "机器人发生<腕关节>奇异."]
  ]
}
```

#### `publish/Error` — 系统错误

```json
{
  "ty": "publish/Error",
  "db": [
    [4, 2023, 1760946003.582, "奇异位置."]
  ]
}
```

---

## 12. 远程脚本模式（端口9002）

### 12.1 进入方式

1. 发送 `project/enterRemoteScriptMode` 请求（9001端口）
2. 或在脚本中调用 `enterRemoteScriptMode()` 指令

### 12.2 通信方式

- **端口**：9002（独立 TCP 连接）
- **格式**：JSON + UTF-8

### 12.3 请求 / 响应格式

```json
// 请求（发送给9002端口）
{
  "script": "movJ(p1, 60, 150)",  // 要执行的 Lua 脚本
  "vars": ["P1", "v1"],           // 可选，执行后返回这些变量的值
  "id": "自定义ID",               // 可选，自定义字段会原值返回
  "command": "resume"             // 可选，控制命令（存在时其他字段无效）
                                  // 可选值: resume=恢复, stop=停止, interrupt=中断当前指令
}

// 响应
{
  "code": 0,         // 0=成功，其他=失败
  "msg": "OK",
  "vars": {
    "P1": [x, y, z, a, b, c],
    "v1": 100
  },
  "id": "自定义ID"   // 原值返回
}
```

### 12.4 运行机制

- 脚本加入运行队列（队列长度64），按顺序执行
- 队列满时返回错误
- `local` 变量仅在当前指令脚本中有效
- 无限循环脚本可通过发送 `interrupt` 命令中断
- 可用工程控制接口（pause / resume / stop）控制工程状态

---

## 13. CRI 实时控制接口

> CRI（Codroid RealTime Interface）基于 UDP，适用于高频率位置控制场景。

### 13.1 数据结构定义

```c
// 推送数据（控制器 → 客户端）
struct PushData {
    Bool    isControlling;     // 是否在实时控制中
    UInt8   errorCode;         // 错误码
    UInt8   nc[6];             // 保留
    Float64 jointPosition[6];  // 当前关节位置
    Float64 endPosition[6];    // 当前末端位置
}

// 指令数据（客户端 → 控制器）
struct CommandData {
    Int64   timestamp;         // 时间戳（当前版本未使用）
    Float64 position[6];       // 目标关节或末端位置
    UInt8   type;              // 0=关节, 1=末端
    UInt8   nc[7];             // 保留字节
}
```

### 13.2 开启数据流推送

```
接口：CRI/StartDataPush（TCP 9001发送）
数据流协议：UDP
```

```json
{
  "id": 1,
  "ty": "CRI/StartDataPush",
  "db": {
    "ip": "192.168.1.200",  // 推送目标 IP
    "port": 9040,            // 推送目标端口（1000-65534）
    "duration": 1            // 推送间隔，单位 ms，≥1 整数
  }
}
```

### 13.3 关闭数据流推送

```json
{"id": 1, "ty": "CRI/StopDataPush"}
```

### 13.4 开启实时控制

```json
{
  "id": 1,
  "ty": "CRI/StartControl",
  "db": {
    "filterType": 0,      // 0=关闭滤波, 1=平均滤波, 2=二阶低通, 3=椭圆滤波
    "duration": 1,        // 指令间隔，ms，范围 [1, 16]
    "startBuffer": 3      // 启动缓冲点数，范围 [1, 100]，缓冲满后机器人开始运动
  }
}
```

### 13.5 关闭实时控制

```json
{"id": 1, "ty": "CRI/StopControl"}
```

---

## 14. 末端 RS485 接口

### 14.1 初始化

```json
{
  "id": 1,
  "ty": "EC2RS485/init",
  "db": {
    "baudrate": 115200,  // 支持: 300/600/1200/2400/4800/9600/14400/19200/38400/56000/57600/115200/128000/230400
    "stopBit": 1,        // 可选，1或2，默认1
    "dataBit": 8,        // 可选，固定8位
    "parity": 0          // 可选，0=无校验, 1=奇, 2=偶，默认0
  }
}
```

### 14.2 清空读取缓存

```json
{"id": 1, "ty": "EC2RS485/flushReadBuffer"}
```

### 14.3 读取数据

```json
// 请求
{"id": 1, "ty": "EC2RS485/read", "db": {"length": 8, "timeout": 3000}}
// timeout: 可选，0~3000 ms，默认3000

// 响应 db: [2, 10]  每字节的无符号整数值，失败返回空数组
```

### 14.4 发送数据

```json
// 请求 db: 无符号整数数组，最大长度127
{"id": 1, "ty": "EC2RS485/write", "db": [2, 10]}
```

---

## 15. ModbusTCP 主站接口

### 15.1 创建设备连接

```json
{
  "id": 1, "ty": "ModbusTcp/setDevice",
  "db": {
    "name": "deviceName",      // 设备名称（唯一）
    "ip": "192.168.1.100",
    "port": 502,
    "slaveId": 1,
    "endian": 1               // 1=大端, 2=小端，默认1
  }
}
```

### 15.2 删除设备

```json
{"id": 1, "ty": "ModbusTcp/removeDevice", "db": {"name": "deviceName"}}
```

### 15.3 创建/修改通信表

```json
{
  "id": 1, "ty": "ModbusTcp/setTable",
  "db": {
    "name": "设备名",
    "tableName": "表名",
    "functionCode": 3,    // 支持: 0x01,0x02,0x03,0x04,0x05,0x06,0x0F,0x10
    "addr": 0,
    "count": 10,
    "period": 1000        // 通信周期，ms
  }
}
```

### 15.4 修改地址的值

```json
{
  "id": 1, "ty": "ModbusTcp/setVal",
  "db": {
    "name": "设备名", "tableName": "表名",
    "addr": 0, "val": 33  // 线圈寄存器值为0或1
  }
}
```

### 15.5 获取设备配置

```json
{"id": 1, "ty": "ModbusTcp/getConfig"}
// 支持的数据类型：I16, I32, I64, U16, U32, U64, F32, F64
```

### 15.6 获取设备状态

```json
{"id": 1, "ty": "ModbusTcp/getState"}
// 响应中每个表含 syncCount(同步次数), syncError(错误次数), val(当前值)
```

---

## 16. 拖动功能接口

| 功能 | 接口类型 |
|------|---------|
| 初始化拖动参数 | `Robot/SetDefaultServoDragParam` |
| 获取拖动灵敏度（0-100） | `Robot/GetDragSensitivity` |
| 获取拖动模式 | `Robot/GetDragMode` |
| 开启/关闭拖动姿态锁 | `Robot/SetCartOriLock` |
| 获取拖动姿态锁状态 | `Robot/GetCartOriLockState` |

```json
// 开启姿态锁示例
{"id": 1, "ty": "Robot/SetCartOriLock", "db": {"cartOriLock": true}}

// 拖动模式值：0=老版拖动, 2=新版拖动
```

---

## 17. 传送带接口

> ⚠️ 传送带相关接口标注为**待测试**，生产使用前请充分验证。

| 功能 | 接口类型 | 说明 |
|------|---------|------|
| 获取编码器计数 | `Conveyor/getEncoderCount` | 传入 `index`（传送带编号） |
| 使能传送带 | `Conveyor/enableConveyor` | 高速IO同步编码器 |
| 取消使能 | `Conveyor/disableConveyor` | — |
| 传送带标定 | `Conveyor/calibrateConveyor` | 传入起/终点坐标和编码器计数 |
| 设置配置参数 | `Conveyor/setConfig` | 类型/坐标系/触发方式/相机等 |
| 获取配置参数 | `Conveyor/getConfig` | — |
| 连接相机 | `Conveyor/connectCamera` | — |
| 断开相机 | `Conveyor/disconnectCamera` | — |
| 获取相机状态 | `Conveyor/getCameraState` | — |
| 发消息给相机 | `Conveyor/sendMsgToCamera` | — |

---

## 18. 接口速查表

### 工程控制

| 功能 | 接口 |
|------|------|
| 运行脚本 | `project/runScript` |
| 进入远程脚本模式 | `project/enterRemoteScriptMode` |
| 运行工程（ID） | `project/run` |
| 运行工程（索引） | `project/runByIndex` |
| 暂停工程 | `project/pause` |
| 恢复工程 | `project/resume` |
| 停止工程 | `project/stop` |
| 单步运行 | `project/runStep` |
| 设置启动行 | `project/setStartLine` |
| 清除启动行 | `project/clearStartLine` |

### 机器人控制

| 功能 | 接口 |
|------|------|
| 上使能 | `Robot/switchOn` |
| 下使能 | `Robot/switchOff` |
| 切换手动 | `Robot/toManual` |
| 切换自动 | `Robot/toAuto` |
| 切换远程 | `Robot/toRemote` |
| 切换仿真 | `Robot/toSimulation` |
| 切换实机 | `Robot/toActual` |
| 开启拖拽 | `Robot/startDrag` |
| 关闭拖拽 | `Robot/stopDrag` |
| 清除错误 | `System/clearError` |

### 运动控制

| 功能 | 接口 |
|------|------|
| 点动 | `Robot/jog` |
| 点动心跳 | `Robot/jogHeartbeat` |
| 停止点动 | `Robot/stopJog` |
| RunTo | `Robot/moveTo` |
| RunTo心跳 | `Robot/moveToHeartbeat` |
| 运动指令 | `Robot/move` |
| 暂停运动 | `Robot/pause` |
| 恢复运动 | `Robot/resume` |
| 停止运动 | `Robot/stopMove` |
| 手动倍率 | `Robot/setManualMoveRate` |
| 自动倍率 | `Robot/setAutoMoveRate` |

### 数据读写

| 功能 | 接口 |
|------|------|
| 获取IO | `IOManager/GetIOValue` |
| 写入IO | `IOManager/SetIOValue` |
| 获取寄存器 | `RegisterManager/GetRegisterValue` |
| 写入寄存器 | `RegisterManager/SetRegisterValue` |
| 获取全局变量 | `globalVar/getVars` |
| 保存全局变量 | `globalVar/saveVars` |
| 删除全局变量 | `globalVar/removeVars` |
| 获取工程变量 | `globalVar/GetProjectVarUpdate` |

### 订阅主题

| 主题 | 说明 |
|------|------|
| `publish/ProjectState` | 工程状态 |
| `publish/RobotStatus` | 机器人状态（模式/使能/速度等）|
| `publish/RobotPosture` | 实时位姿（关节角 + TCP）|
| `publish/RobotCoordinate` | 工具/用户坐标系 |
| `publish/VarUpdate` | 变量变化推送 |
| `publish/Log` | 系统日志 |
| `publish/Error` | 系统错误 |

---

*文档整理时间：2026-06-12 | 适用固件版本：≥ V2.3（部分接口需 ≥ 2.3.2.6）*
