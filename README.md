# Ensemble Beat Grid API

合奏节拍网格服务：提交一小节的**目标时长**和**至少两个声部**（每个声部是依次排列的
发声/休止事件，时长为正有理数字符串），服务用**精确分数**校验各声部总时长是否等于
目标；通过后保存小节，并可按 ID 查询由所有事件边界并集切分出的**半开时间段网格**。

技术栈：Python 3.12 · FastAPI · SQLite（标准库 `sqlite3`）· Docker / Compose

## 启动（一条命令）

```bash
docker compose up --build
```

该命令完成依赖安装、SQLite 数据库初始化（应用启动时自动建表）并启动服务，**无需任何
额外步骤**；服务监听容器内 **8000** 端口，Compose 将宿主机 **8000** 端口映射到它。

- API 地址： http://localhost:8000
- 交互式文档（Swagger UI）： http://localhost:8000/docs

## 配置

| 环境变量         | 默认值          | 说明                     |
| ---------------- | --------------- | ------------------------ |
| `DATABASE_PATH`  | `/data/app.db`（容器内）/ `./app.db`（本地） | SQLite 数据库文件路径 |
| （端口映射）      | `8000:8000`     | 在 `docker-compose.yml` 中修改 |

Compose 中数据库文件挂载在命名卷 `beat-grid-data` 上，容器重启数据不丢失。

## API

### `POST /measures` — 提交小节

```bash
curl -s -X POST http://localhost:8000/measures \
  -H 'Content-Type: application/json' \
  -d '{
    "target_duration": "4/4",
    "voices": [
      {"name": "melody", "events": [
        {"kind": "sound", "duration": "1/2"},
        {"kind": "rest",  "duration": "1/4"},
        {"kind": "sound", "duration": "1/4"}
      ]},
      {"name": "bass", "events": [
        {"kind": "rest",  "duration": "1/4"},
        {"kind": "sound", "duration": "3/4"}
      ]}
    ]
  }'
```

- 时长格式：正有理数字符串，如 `"2"`、`"1/4"`、`"3/8"`（拒绝 `0`、负数、小数、
  分母为 0）。所有时长以 `fractions.Fraction` 精确累加，无浮点误差。
- 任一一声部总时长 ≠ 目标时长时返回 **422**，逐声部给出 `expected` / `actual` /
  `difference`（= actual − expected，负值表示不足），且**不保存任何数据**：

```json
{
  "detail": {
    "message": "voice durations do not add up to the target duration; measure was not saved",
    "mismatches": [
      {"voice": "melody", "expected": "1", "actual": "3/4", "difference": "-1/4"}
    ]
  }
}
```

- 全部匹配时返回 **201** 及保存的小节（含每个事件的 ID `v{声部序号}e{事件序号}`、
  规范化的起止偏移）。

### `GET /measures/{id}` — 查询小节

返回小节、各声部及全部事件（种类、时长、起止偏移）。

### `GET /measures/{id}/grid` — 查询节拍网格

对所有事件的起止点取并集，按相邻点生成**半开时间段** `[start, end)`；每段列出各声部
正在发声的事件 ID，休止为 `null`。半开区间保证相邻事件边界处既不丢音也不重复计音：

```json
{
  "measure_id": 1,
  "target_duration": "1",
  "voices": ["melody", "bass"],
  "segments": [
    {"start": "0",   "end": "1/4", "events": {"melody": "v1e1", "bass": null}},
    {"start": "1/4", "end": "1/2", "events": {"melody": "v1e1", "bass": "v2e2"}},
    {"start": "1/2", "end": "3/4", "events": {"melody": null,   "bass": "v2e2"}},
    {"start": "3/4", "end": "1",   "events": {"melody": "v1e3", "bass": "v2e2"}}
  ]
}
```

### `POST /measures/{id}/quantization-preview` — 量化到等分刻度（预览，不写库）

把已保存小节**所有声部的不同事件边界**（各事件起止点的并集）统一映射到把目标时长
等分为 `divisions` 份的刻度网格上（刻度间隔 = 目标时长 / `divisions`）。映射必须：

1. **固定首尾**：边界 `0` 映射到刻度 0，边界 `目标时长` 映射到刻度 `divisions`；
2. **严格递增**：随边界顺序映射到严格递增的不同刻度（多个边界不能挤到同一刻度）；
3. **共享边界一致**：多个声部在同一时刻的边界只映射一次，量化后各声部在该点完全对齐；
4. **逐点限偏**：每个边界的偏移绝对值 `|量化后 − 原边界|` 都不超过 `max_shift`。

在所有可行映射中，使**偏移绝对值总和最小**；总和相同时，取**刻度序列字典序最小**
（越靠前的边界越优先取更小刻度）。全部运算使用 `fractions.Fraction` 精确完成。

- 请求体：`divisions` 为 **1～128** 的整数；`max_shift` 为**非负有理数字符串**
  （允许 `"0"`，其余规则同时长格式）。越界或格式错误返回 **422**（请求校验错误）。
- 小节 ID 不存在返回 **404**。
- 不存在满足上述条件的映射时返回 **422**，detail 给出 `reason` / `divisions` /
  `max_shift`；该接口为**只读预览，不写入数据库**，已保存小节保持不变。

```bash
curl -s -X POST http://localhost:8000/measures/1/quantization-preview \
  -H 'Content-Type: application/json' \
  -d '{"divisions": 8, "max_shift": "1/8"}'
```

例：小节时长 `1`，声部边界在 `1/3`、`1/2` 等处；8 等分刻度间隔 `1/8`，`1/3` 就近
映射到刻度 3（`3/8`，偏移 `+1/24`），`1/2` 恰在刻度 4，偏移总和 `1/24`：

```json
{
  "measure_id": 1,
  "target_duration": "1",
  "divisions": 8,
  "grid_step": "1/8",
  "max_shift": "1/8",
  "total_abs_shift": "1/24",
  "boundaries": [
    {"original": "0",    "tick": 0, "quantized": "0",   "shift": "0"},
    {"original": "1/3",  "tick": 3, "quantized": "3/8", "shift": "1/24"},
    {"original": "1/2",  "tick": 4, "quantized": "1/2", "shift": "0"},
    {"original": "1",    "tick": 8, "quantized": "1",   "shift": "0"}
  ],
  "voices": [
    {"name": "x", "events": [
      {"id": "v1e1", "kind": "sound", "start": "0",   "end": "3/8", "duration": "3/8"},
      {"id": "v1e2", "kind": "rest",  "start": "3/8", "end": "1",   "duration": "5/8"}
    ]},
    {"name": "y", "events": [
      {"id": "v2e1", "kind": "rest",  "start": "0",   "end": "1/2", "duration": "1/2"},
      {"id": "v2e2", "kind": "sound", "start": "1/2", "end": "1",   "duration": "1/2"}
    ]}
  ]
}
```

各字段含义：

| 字段               | 说明                                                                 |
| ------------------ | -------------------------------------------------------------------- |
| `grid_step`        | 等分刻度间隔（= `target_duration / divisions`，精确分数）            |
| `total_abs_shift`  | 全部不同边界偏移绝对值之和（最小化目标，精确分数）                   |
| `boundaries`       | 原边界 → 新边界对应关系：`original` / `tick`（刻度序号，0 起）/ `quantized` / `shift`（= quantized − original，可为负） |
| `voices[].events`  | 各声部**原事件 ID** 与种类，以及量化后的 `start` / `end` / `duration`；时长严格为正且逐声部首尾拼成目标时长 |

`max_shift` 过紧、或刻度数不足以容纳所有边界（内部不同边界数超过 `divisions - 1`）时
不可行，例如 3 等分网格（刻度在 `1/3`、`2/3`）下边界 `1/2` 距两个相邻刻度都是 `1/6`，
`max_shift = "1/8"` 时返回 **422**：

```json
{
  "detail": {
    "message": "no quantization of the measure's event boundaries onto the requested grid stays within max_shift; nothing was saved",
    "reason": "boundary 1/2 has no grid tick within max_shift 1/8",
    "divisions": 3,
    "max_shift": "1/8"
  }
}
```

### `GET /measures/{id}/rhythm-matches?voice={声部名}` — 检索节奏等价声部

以指定小节中的一个声部为**源声部**，在全部已保存小节中检索与它**节奏等价**的其他声部
（不含源声部自身）。两个声部节奏等价，当且仅当存在某个循环位移 `shift`（0 ≤ shift < N）
使下列条件同时成立：

1. 两段事件**数量相同**（都为 N）；
2. 候选的第 `j` 个事件与源的第 `(j + shift) mod N` 个事件**发声/休止种类逐项一致**；
3. 每一对对应事件的**时长之比都是同一个正有理数** `ratio`（= 候选时长 / 源时长，即整体
   变速比例；`"1"` 表示速度不变）——允许整体拉伸或压缩，但不允许内部相对节奏变形。

全部比较使用 `fractions.Fraction` **精确分数**完成，无浮点误差。若有多个可行位移，
取**最小**的 `shift`（从 0 起逐个尝试）。

- 查询参数 `voice` 必填，值为源声部名；源小节不存在、或该声部名不在源小节中时返回
  **404**；缺少 `voice` 参数返回 **422**。
- 结果按**小节 ID 升序**排列，同一小节内按声部的**原始保存顺序**排列；没有匹配时
  `matches` 为空数组。

```bash
curl -s 'http://localhost:8000/measures/1/rhythm-matches?voice=lead'
```

例：源小节声部 `lead` 为「发声 1/4、休止 1/4、发声 1/2」，另一小节的 `echo` 为
「发声 1、发声 1/2、休止 1/2」——整体时长放大 2 倍且循环位移 2 位后逐项对应：

```json
{
  "measure_id": 1,
  "voice": "lead",
  "matches": [
    {"measure_id": 1, "voice": "twin", "ratio": "1", "shift": 0},
    {"measure_id": 2, "voice": "echo", "ratio": "2", "shift": 2}
  ]
}
```

各字段含义：

| 字段         | 说明                                                                 |
| ------------ | -------------------------------------------------------------------- |
| `measure_id` | 候选小节 ID                                                          |
| `voice`      | 候选声部名（仅按 `(小节 ID, 声部名)` 排除源声部自身；同名的其他声部仍正常参与匹配） |
| `ratio`      | 候选/源的统一时长比例（正有理数字符串），`"2"` 表示候选整体慢一倍     |
| `shift`      | 循环位移位数：候选第 `j` 个事件对应源第 `(j + shift) mod N` 个事件，取最小可行值 |

### `POST /loop-alignments` — 两小节循环对齐（排练用）

接收两个**已保存小节**的 ID（同一小节可同时作为 A、B），把它们当作两条不同长度的
节奏循环，用精确分数求两条循环**首次同时回到起点**的时刻（两小节时长的最小公倍数），
并把两条循环的事件分别重复到该时刻。时间线按两条循环**全部事件边界**的并集切成
半开区间；每段分别在 `a` / `b` 下按各声部给出正在发声事件的**原事件 ID** 和
**从 1 开始的循环序号**，休止为 `null`。

- 任一小节 ID 不存在返回 **404**（detail 指明是 `a` 还是 `b`），不产生时间线。
- 展开的事件副本总数（= A 重复次数 × A 事件数 + B 重复次数 × B 事件数）超过
  **2000** 时返回 **422**，detail 给出 `expanded_events` / `limit` / `horizon` /
  `repetitions` 等信息，**不输出任何截断的时间线**（响应中没有 `segments`）。

```bash
curl -s -X POST http://localhost:8000/loop-alignments \
  -H 'Content-Type: application/json' \
  -d '{"a_measure_id": 1, "b_measure_id": 2}'
```

例：A 小节时长 `1`、B 小节时长 `3/2`，首次重合在 `3`（A 重复 3 次、B 重复 2 次）：

```json
{
  "a_measure_id": 1,
  "b_measure_id": 2,
  "horizon": "3",
  "length_a": "1",
  "length_b": "3/2",
  "repetitions": {"a": 3, "b": 2},
  "voices": {"a": ["melody", "bass"], "b": ["melody", "bass"]},
  "expanded_events": 21,
  "segments": [
    {
      "start": "0",
      "end": "1/4",
      "a": {"melody": {"event_id": "v1e1", "loop": 1}, "bass": null},
      "b": {"melody": {"event_id": "v1e1", "loop": 1}, "bass": null}
    },
    {
      "start": "1",
      "end": "5/4",
      "a": {"melody": {"event_id": "v1e1", "loop": 2}, "bass": null},
      "b": {"melody": null, "bass": null}
    }
  ]
}
```

`event_id` 是事件在**原小节**中的 ID（如 `v1e1`，与 `GET /measures/{id}` 返回的一致）；
`loop` 表示该发声来自第几次循环。半开区间 `[start, end)` 保证循环边界处既不丢音也不
重复计音：恰好在某次循环起点休止的声部，在该起点的段中为 `null`。

### `POST /measures/{id}/avoidance-preview` — 双声部冲突避让（预览，不写库）

对已保存小节中的**两个不同声部**做碰撞避让预览。把两个声部的每个**发声事件**
（`kind = "sound"`，**休止事件不参与**）视为**不可拆分**的整体：当两个发声事件的
半开区间具有**正长度交集**时构成冲突——仅在边界相接（一个事件的 `end` 恰好等于另一
事件的 `start`）不算冲突，交集长度必须严格大于 0。

消除冲突的唯一手段是把任意冲突事件**整体静音**（静音一个事件即同时解除它参与的全部
冲突）。服务在所有可行静音方案中，用**精确分数**求**被静音事件原时长之和最小**的一组
方案。该问题等价于以两声部发声事件为二分部顶点、事件原时长为点权、冲突关系为边的
**二分图最小权顶点覆盖**，内部用分数容量的**最小割 / 最大流**精确求解；所有时长均为
`fractions.Fraction`，无浮点误差。

- 请求体：`voice_a`、`voice_b` 为两个声部名（1～100 字符）。小节 ID 不存在返回
  **404**；任一声部名不在该小节中、或两个声部名相同返回 **422**（detail 给出原因）。
- 响应给出：`muted`（建议整体静音的事件：事件 ID、所属声部、原起止、**原时长**）、
  `total_muted`（被静音事件原时长之和，最小化目标，精确分数字符串）、`conflicts`
  （**全部原始冲突事件对**：双方事件信息与正长度重叠时长 `overlap`）。没有冲突时
  `muted` / `conflicts` 均为空、`total_muted` 为 `"0"`。
- 该接口为**只读预览，不写入数据库**，已保存小节保持不变。

```bash
curl -s -X POST http://localhost:8000/measures/1/avoidance-preview \
  -H 'Content-Type: application/json' \
  -d '{"voice_a": "lead", "voice_b": "bass"}'
```

例：小节时长 `1`，`lead` 在 `[0, 1/3)` 发声，`bass` 在 `[0, 1/4)`、`[1/4, 3/8)`
连续发声。两处冲突的正长度重叠分别为 `1/4` 与 `1/12`；静音 `lead` 的整事件花费
`1/3`，而静音 bass 的两个事件要花 `1/4 + 1/8 = 3/8`，故最优方案是整体静音
`lead` 的 `v1e1`，总静音时长 `1/3`：

```json
{
  "measure_id": 1,
  "target_duration": "1",
  "voice_a": "lead",
  "voice_b": "bass",
  "muted": [
    {"id": "v1e1", "voice": "lead", "start": "0", "end": "1/3", "duration": "1/3"}
  ],
  "total_muted": "1/3",
  "conflicts": [
    {
      "a": {"id": "v1e1", "voice": "lead", "start": "0", "end": "1/3", "duration": "1/3"},
      "b": {"id": "v2e1", "voice": "bass", "start": "0", "end": "1/4", "duration": "1/4"},
      "overlap": "1/4"
    },
    {
      "a": {"id": "v1e1", "voice": "lead", "start": "0", "end": "1/3", "duration": "1/3"},
      "b": {"id": "v2e2", "voice": "bass", "start": "1/4", "end": "3/8", "duration": "1/8"},
      "overlap": "1/12"
    }
  ]
}
```

各字段含义：

| 字段              | 说明                                                                 |
| ----------------- | -------------------------------------------------------------------- |
| `muted`           | 一组整体静音的发声事件：`id` / `voice` / `start` / `end` / `duration`（**原事件时长**，非重叠时长）；按声部存储顺序排列 |
| `total_muted`     | `muted` 中各事件原时长之和（最小化目标值，精确分数）                 |
| `conflicts`       | 全部原始冲突对：`a` / `b` 为双方发声事件（ID、声部、起止与时长），`overlap` 为正长度重叠时长；边界相接与休止均不出现在其中 |

> 当存在多组同样最优的静音方案（最小总时长相同）时，接口返回其中一组；最优总时长
> `total_muted` 唯一。返回的 `muted` 保证覆盖 `conflicts` 中的每一对冲突。

小节不存在与声部参数错误示例：

```json
{"detail": "measure not found"}
```

```json
{
  "detail": {
    "message": "voice_a and voice_b must be two different voices",
    "voice_a": "lead",
    "voice_b": "lead"
  }
}
```

```json
{
  "detail": {
    "message": "voice not found in measure",
    "missing_voices": ["drums"]
  }
}
```

### `POST /measures/{id}/time-preview` — 速度段时间预览（不写库）

把已保存小节的每个事件边界从**全音符偏移**（全音符时长为 1）换算成**自小节开始的秒数**。
请求体给出一个**速度段数组** `segments`，每段含三个字段：

| 字段         | 说明                                                                 |
| ------------ | -------------------------------------------------------------------- |
| `end`        | 段终点的偏移（正有理数字符串，格式同时长）；各段 `end` 必须**严格递增**，且**末段 `end` 等于小节目标时长** |
| `start_bpm`  | 段起点处的速度（**正十进制字符串**，如 `"120"`、`"132.5"`，表示每分钟四分音符数） |
| `end_bpm`    | 段终点处的速度（同上）                                               |

第一段从偏移 `0` 开始，之后每段起点为前一段的 `end`；段内 BPM 随偏移**线性变化**，段与段之间
允许 BPM **跳变**。换算规则：全音符 = 4 个四分音符，故恒定 BPM `b` 时一个全音符历时
`240 / b` 秒；段内线性变化时对 `240 / bpm(x)` 积分（对数形式），段间跳变不影响共享边界处的
累计秒值。全部计算使用高精度十进制运算（50 位有效数字），仅在输出时舍入。

- 响应按声部给出每个事件的 `id`、`kind` 及 `start_seconds` / `end_seconds`
  （**六位小数字符串**，半进位舍入），并给出全小节总秒数 `total_seconds`；**同一偏移只积分
  一次**，凡重复出现的偏移（声部内相邻事件的共享边界、跨声部同一时刻）复用同一秒值。
- 速度段无效（数组为空、`end` 未严格递增、末段 `end` 不等于目标时长、BPM 或 `end` 格式
  非法）返回 **422**；小节 ID 不存在返回 **404**；该接口为**只读预览，不写入数据库**。

```bash
curl -s -X POST http://localhost:8000/measures/1/time-preview \
  -H 'Content-Type: application/json' \
  -d '{
    "segments": [
      {"end": "1/2", "start_bpm": "120", "end_bpm": "120"},
      {"end": "1",   "start_bpm": "120", "end_bpm": "60"}
    ]
  }'
```

例：小节时长 `1`，前半段恒定 120 BPM（一个全音符 2 秒，偏移 `1/4` → `0.5` 秒、`1/2` →
`1` 秒），后半段从 120 线性渐慢到 60（整段历时 `2·ln 2 ≈ 1.386294` 秒）：

```json
{
  "measure_id": 1,
  "target_duration": "1",
  "total_seconds": "2.386294",
  "voices": [
    {"name": "melody", "events": [
      {"id": "v1e1", "kind": "sound", "start_seconds": "0.000000", "end_seconds": "1.000000"},
      {"id": "v1e2", "kind": "rest",  "start_seconds": "1.000000", "end_seconds": "1.575364"},
      {"id": "v1e3", "kind": "sound", "start_seconds": "1.575364", "end_seconds": "2.386294"}
    ]},
    {"name": "bass", "events": [
      {"id": "v2e1", "kind": "rest",  "start_seconds": "0.000000", "end_seconds": "0.500000"},
      {"id": "v2e2", "kind": "sound", "start_seconds": "0.500000", "end_seconds": "2.386294"}
    ]}
  ]
}
```

速度段无效时的响应示例：

```json
{
  "detail": {
    "message": "invalid tempo segments; nothing was saved",
    "reason": "the final segment end 3/4 must equal the measure target duration 1",
    "target_duration": "1"
  }
}
```

### `POST /measures/{id}/lead-handoff-preview` — 领奏接力预览（只读，不写库）

在节拍网格（所有事件边界并集切出的半开段 `[start, end)`，与 `GET /measures/{id}/grid`
完全相同）上为每段选一位**领奏声部**，用于预览整小节的领奏接力：

- **有声段**（至少一个声部正在发声）必须从该段**正在发声的声部**中选一位领奏者；
- **全休止段**（所有声部都休止）领奏为 `null`（`event_id` 也为 `null`），并且
  **打断接力**——其后第一个有声段相当于接力重新开始；
- 相邻两个有声段的领奏声部不同，计一次**换手**；领奏相同（即使该声部的事件已换成
  另一个事件 ID）不计换手；**跨全休止段不计换手**。

服务在所有合法领奏方案中使**整小节换手总数最少**；换手总数相同时，按声部的**存储顺序**
比较**完整领奏序列**，取**字典序最小**者（越早的段越优先取存储顺序靠前的声部；全休止位置
在每个方案中都是同样的 `null`，不参与平局比较）。全部运算使用 `fractions.Fraction`
精确完成。

- 请求**无请求体**（直接 POST），响应按网格段给出有理起止、领奏声部名及其**事件 ID**，
  并给出 `handoff_count`（换手总数）。
- 小节 ID 不存在返回 **404**；该接口为**只读预览，不写入数据库**。

```bash
curl -s -X POST http://localhost:8000/measures/1/lead-handoff-preview
```

例：小节时长 `1`，`melody` 在 `[0, 1/2)`、`[3/4, 1)` 发声，`bass` 在 `[1/4, 1)`
发声。网格四段依次只有 melody、两者皆有、只有 bass、两者皆有；让 melody 领前两段、
bass 领后两段只需在中间换手一次，为最少换手方案：

```json
{
  "measure_id": 1,
  "target_duration": "1",
  "voices": ["melody", "bass"],
  "handoff_count": 1,
  "segments": [
    {"start": "0",   "end": "1/4", "leader": "melody", "event_id": "v1e1"},
    {"start": "1/4", "end": "1/2", "leader": "melody", "event_id": "v1e1"},
    {"start": "1/2", "end": "3/4", "leader": "bass",   "event_id": "v2e2"},
    {"start": "3/4", "end": "1",   "leader": "bass",   "event_id": "v2e2"}
  ]
}
```

全休止段打断接力的例子：`a` 只在 `[0, 1/3)` 发声、`b` 只在 `[2/3, 1)` 发声，中间
`[1/3, 2/3)` 所有人休止。中段返回 `null` 并打断接力，因此 b 在休止后接过领奏**不计
换手**，`handoff_count` 为 `0`：

```json
{
  "measure_id": 2,
  "target_duration": "1",
  "voices": ["a", "b"],
  "handoff_count": 0,
  "segments": [
    {"start": "0",   "end": "1/3", "leader": "a", "event_id": "v1e1"},
    {"start": "1/3", "end": "2/3", "leader": null, "event_id": null},
    {"start": "2/3", "end": "1",   "leader": "b", "event_id": "v2e2"}
  ]
}
```

各字段含义：

| 字段              | 说明                                                                 |
| ----------------- | -------------------------------------------------------------------- |
| `handoff_count`   | 整小节换手总数（最小化目标）；只统计相邻有声段之间的领奏声部变化     |
| `segments[].leader`    | 该段领奏声部名；全休止段为 `null`（同时打断接力）                |
| `segments[].event_id`  | 领奏声部在该段正在发声的事件 ID（与 `GET /measures/{id}` 一致）；全休止段为 `null` |

小节不存在时：

```json
{"detail": "measure not found"}
```

### `POST /measures/{id}/practice-tracks` — 排练双轨分组预览（只读，不写库）

把已保存小节的声部（**至多 16 个**）分成 A / B 两条排练轨：只要两个声部的发声事件
存在**正时长重叠**（休止不参与；仅在边界相接、即一个事件的 `end` 等于另一个的
`start`，不算冲突），它们就必须分到不同的轨。声部间的冲突关系构成一张图，能分成
两轨当且仅当该图可**二染色**；若图中存在奇数长度的冲突环（例如三个声部两两互相
重叠），则任何分组都不可能。

- 每轨**负载** = 该轨各声部**发声事件时长之和**（休止不计入），全部用
  `fractions.Fraction` 精确计算。在所有可行分组中，服务选取**两轨负载差绝对值
  最小**的方案；并列时按小节**原声部顺序**比较完整的轨标记序列，取**字典序最小**
  者（`A` 先于 `B`，即越靠前的声部越优先分到 A 轨）。
- 请求**无请求体**（直接 POST）。响应给出 A / B 两轨的声部列表（各自按原声部
  顺序）、各轨负载 `load`（精确分数字符串）及负载差 `load_difference`
  （= |A 轨负载 − B 轨负载|）。
- 小节 ID 不存在返回 **404**；声部数超过 **16** 返回 **422**（detail 给出 `voices`
  与 `limit`）；冲突关系无法二分时返回 **422**，detail 的 `cycle` 列出一组构成矛盾
  的声部（一个奇数冲突环，按环序排列，末位声部与首位也互相冲突）。
- 该接口为**只读预览，不写入数据库**，已保存小节保持不变。

```bash
curl -s -X POST http://localhost:8000/measures/1/practice-tracks
```

例：小节时长 `1`，`melody` 在 `[0, 1/2)`、`[3/4, 1)` 发声（负载 3/4），`bass` 在
`[1/4, 1)` 发声（负载 3/4）。两声部在 `[1/4, 1/2)` 正时长重叠，必须分轨；3/4 对
3/4 已是最均衡分法，并列时按原声部顺序取字典序最小的标记序列，故 melody 分 A 轨：

```json
{
  "measure_id": 1,
  "target_duration": "1",
  "track_a": {"voices": ["melody"], "load": "3/4"},
  "track_b": {"voices": ["bass"], "load": "3/4"},
  "load_difference": "0"
}
```

各字段含义：

| 字段              | 说明                                                                 |
| ----------------- | -------------------------------------------------------------------- |
| `track_a` / `track_b` | 一条排练轨：`voices` 为分到的声部名（按原声部顺序），`load` 为这些声部发声事件时长之和（精确分数） |
| `load_difference` | 两轨负载差的绝对值（最小化目标，精确分数）                           |

三个声部两两互相重叠（奇数冲突环）时返回 **422**：

```json
{
  "detail": {
    "message": "voice conflicts cannot be split into two practice tracks",
    "cycle": ["b", "a", "c"]
  }
}
```

声部数超过上限时同样返回 **422**：

```json
{
  "detail": {
    "message": "too many voices for a practice-track split",
    "voices": 17,
    "limit": 16
  }
}
```

### `POST /measures/{id}/rehearsal-chunks-preview` — 排练分段预览（只读，不写库）

把已保存小节切成若干**排练片段**。切点只允许选在该小节**所有事件起止边界的并集**上
（与 `GET /measures/{id}/grid` 完全相同的那些内部点）；`0` 与小节目标时长固定为首尾
端点，不计入切点。每个片段为半开区间 `[start, end)`，**长度不得超过**请求给出的正有理数
`max_duration`。

切点 `p` **截断**的发声事件，指满足 `start < p < end`（切点严格落在事件内部）的
`sound` 事件；恰好在切点结束或开始的事件**不算**被截断，休止事件也从不计入。同一切点
截断多个声部的事件时，事件 ID 按声部的**存储顺序**列出。

服务在所有可行切法中依次按下列标准择优：

1. **片段数最少**；
2. 片段数相同时，**跨切点仍在发声的事件总次数最少**（各切点截断的发声事件数之和；一个
   事件被多个切点截断则按多次计）；
3. 前两项仍相同时，**切点序列按时间的字典序最早**（逐个切点比较，越早的切点越优先）。

全部运算使用 `fractions.Fraction` 精确完成；该接口为**只读预览，不写入数据库**，已保存
小节保持不变。

- 请求体：`max_duration` 为**正有理数字符串**（格式同时长，拒绝 `0`、负数、小数、分母为
  0）；格式错误或缺少请求体返回 **422**（请求校验错误）。
- 小节 ID 不存在返回 **404**。
- 若**相邻两个并集边界之间的距离**本身就超过 `max_duration`，则该网格段无法再被合法切点
  分割，**不存在可行切法**，返回 **422**，detail 给出 `max_duration` 与越界区间
  `gap`（`start` / `end` / `duration`）及 `reason`。
- 当 `max_duration` 不小于目标时长时，整小节就是一个片段，`cuts` 为空数组、
  `cut_event_count` 为 `0`。

```bash
curl -s -X POST http://localhost:8000/measures/1/rehearsal-chunks-preview \
  -H 'Content-Type: application/json' \
  -d '{"max_duration": "1/2"}'
```

例：小节时长 `1`，`melody` 在 `[0, 1/2)` 发声、`[1/2, 3/4)` 休止、`[3/4, 1)` 发声，
`bass` 在 `[0, 1/4)` 休止、`[1/4, 1)` 发声。上限 `1/2` 时最少切成 2 段；切点 `1/2`
处 melody 的首事件恰好在该点结束（不计），只有 bass 的 `v2e2` 仍在发声：

```json
{
  "measure_id": 1,
  "target_duration": "1",
  "max_duration": "1/2",
  "chunk_count": 2,
  "cut_event_count": 1,
  "chunks": [
    {"start": "0",   "end": "1/2"},
    {"start": "1/2", "end": "1"}
  ],
  "cuts": [
    {"point": "1/2", "sounding_events": ["v2e2"]}
  ]
}
```

各字段含义：

| 字段               | 说明                                                                 |
| ------------------ | -------------------------------------------------------------------- |
| `chunk_count`      | 片段数（第一择优目标，最小值）                                       |
| `cut_event_count`  | 所有切点截断的发声事件次数之和（第二择优目标，最小值）               |
| `chunks[]`         | 片段的半开有理起止 `start` / `end`，首尾无缝拼成 `[0, 目标时长)`     |
| `cuts[]`           | 内部切点 `point` 及该点截断的发声事件 ID `sounding_events`（按声部存储顺序；没有事件被截断时为空数组） |

第二择优标准的例子：`a` 整小节发声，`b` 在 `[1/4, 3/4)` 发声（两端休止），`c` 只在
`[3/4, 1)` 发声。上限 `1/2` 时最少 3 段；切在 `1/4、3/4` 只截断 `a` 一个事件两次
（共 2 次），而任何包含切点 `1/2` 的切法还会截断 `b` 的中间事件（共 3 次），故取
`{1/4, 3/4}`：

```json
{
  "measure_id": 2,
  "target_duration": "1",
  "max_duration": "1/2",
  "chunk_count": 3,
  "cut_event_count": 2,
  "chunks": [
    {"start": "0",   "end": "1/4"},
    {"start": "1/4", "end": "3/4"},
    {"start": "3/4", "end": "1"}
  ],
  "cuts": [
    {"point": "1/4", "sounding_events": ["v1e1"]},
    {"point": "3/4", "sounding_events": ["v1e1"]}
  ]
}
```

无可行切法时返回 **422**（本例并集相邻点 `0` 与 `1/4` 相距 `1/4`，超过上限 `1/8`，
中间没有任何合法切点可用）：

```bash
curl -s -X POST http://localhost:8000/measures/1/rehearsal-chunks-preview \
  -H 'Content-Type: application/json' \
  -d '{"max_duration": "1/8"}'
```

```json
{
  "detail": {
    "message": "no boundary-anchored cut keeps every rehearsal chunk within max_duration; nothing was saved",
    "reason": "adjacent event boundaries 0 and 1/4 are 1/4 apart, more than the max chunk duration 1/8; no boundary-anchored cut can split that span",
    "max_duration": "1/8",
    "gap": {"start": "0", "end": "1/4", "duration": "1/4"}
  }
}
```

小节不存在、请求体非法时：

```json
{"detail": "measure not found"}
```

> 在浏览器中打开 http://localhost:8000/ 可查看服务与接口清单，
> http://localhost:8000/docs 为可直接发起请求的 Swagger UI。

## 本地开发与测试

```bash
pip install -r requirements-dev.txt
uvicorn app.main:app --reload          # 本地启动（默认 ./app.db）
pytest                                 # 全部测试（纯逻辑 + API）
python3 -m unittest discover -s tests -p 'test_grid.py'   # 仅核心逻辑（零依赖）
```

## 项目结构

```
app/
  grid.py      # 核心逻辑：有理数解析、精确累加、边界并集切分网格、节奏等价匹配、循环对齐、等分刻度量化、双声部冲突静音（二分图最小权顶点覆盖）、速度段积分换算秒数、领奏接力最省换手预览、排练双轨分组（冲突图二染色 + 负载均衡）、排练分段（边界切点上的最少片段/最少截断/最早切序列，纯标准库）
  db.py        # SQLite 连接、建表、读写（纯标准库）
  schemas.py   # Pydantic 请求/响应模型与校验
  main.py      # FastAPI 应用与路由（启动时初始化数据库）
tests/         # 核心逻辑测试（stdlib 可跑）+ API 测试（TestClient）
Dockerfile
docker-compose.yml
```
