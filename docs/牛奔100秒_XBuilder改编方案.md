# 《坚持100秒 · 牛奔100秒》XBuilder/spx 改编方案

## 一、技术底座确认（已核实）

| 项      | 结论                                                                           | 来源                          |
| ------ | ---------------------------------------------------------------------------- | --------------------------- |
| 引擎     | spx（基于 XGo + Godot）                                                          | goplus/spx                  |
| 项目格式   | `.xbp` = ZIP，含 `main.spx` + `assets/` + `builder-meta.json`                  | XBuilder Project Management |
| 精灵配置   | `assets/sprites/<Name>/index.json`                                           | spx config schema           |
| 造型     | `costumes: RawCostumeConfig[]` 数组，每项独立 `path` + `pivot` + `bitmapResolution` | XBuilder costume.ts         |
| SVG 支持 | ✅ `img: File(png, jpg, svg)`，SVG 的 `bitmapResolution=1`                      | XBuilder costume.ts         |
| 命名造型   | `items: [{namePrefix, n}]` → 生成 `prefix-0, prefix-1...`                      | spx config                  |
| 事件     | `onStart / onClick / onKey KeyX / onMsg "x" / onCloned`                      | spx README                  |
| 键盘状态   | `KeyPressed(KeyLeft)` 返回 bool（Game 方法）                                       | pkg.go.dev spx v0.5.0       |
| 碰撞     | `Touching("SpriteName")` / `Touching(sprite)`                                | spx API                     |
| 克隆     | `Sprite.clone` → `onCloned` 中 `setXYpos` 定位                                  | tutorial/03-Clone           |
| 列表     | `List` 类型：`Init/Append/At/Len/Set/Contains/Delete`                           | spx list.go                 |
| 随机     | `rand(from, to)`                                                             | spx utils                   |
| 广播     | `broadcast "msg"` / `broadcastAndWait "msg"`                                 | spx README                  |
| 监视器    | `res/index.json` 中 `type: monitor` + `val: "getVar:score"`                   | spx config                  |

**为什么用 SVG 而不是 PNG**：XBuilder 自带「X绘图」AI 生图能力，产出就是矢量 SVG；且 SVG 体积小、可无损缩放，主包 ≤4MB 的约束下更友好。

---

## 二、玩法改编核心：把「地铁跑酷」翻译成「spx 2D 侧视」

原型是 3 车道俯视伪 3D。spx 是 2D 引擎（无深度），所以改编采用**横向侧视跑酷**（类似 Jetpack Joyride / FlappyCalf）：

| 原型       | spx 改编               | 说明             |
| -------- | -------------------- | -------------- |
| 3 车道横向变道 | 3 个高度层纵向移动（上层/中层/下层） | 障碍从右侧向左推进      |
| ↑ 跳跃     | ↑ 跳跃                 | 跨过「低矮牛」        |
| ↓ 滑铲     | ↓ 滑铲                 | 钻过「高架牛」        |
| 持续加速     | `step` 步长随时间递增       | 障碍左移速度递增       |
| 100 秒通关  | 全局计时器到 100 → 通关      | `onStart` 循环计时 |
| 随机事件任务   | 任务箱掉落 → 限时挑战         | 广播消息驱动         |

**分层的核心设计**：三条"道"改为三个 **Y 高度层**（Y = 150 / 0 / -150）。玩家在层间切换，障碍牛按层出现。这样 spx 的纯 2D 也能承载"多路径选择"的跑酷手感。

---

## 三、牛系障碍物体系（核心创意）

用户要求"障碍物都设计成不同形式的牛"。设计原则：**每种牛对应一种躲避方式**，让牛的外观直接传达玩法信息——这是儿童向游戏的关键设计手法（形→意映射）。

### 3.1 需要【跳跃】的牛（低矮，Y 层低处）

| ID         | 造型名 | 描述         | 判定  |
| ---------- | --- | ---------- | --- |
| `cow_low1` | 卧倒牛 | 趴在地上的牛，只露背 | 跳过去 |
| `cow_low2` | 犊牛  | 小牛犊低头吃草    | 跳过去 |
| `cow_low3` | 滚草牛 | 缩成球的牛（滚动）  | 跳过去 |

### 3.2 需要【滑铲】的牛（高架，占上层空间）

| ID          | 造型名 | 描述            | 判定  |
| ----------- | --- | ------------- | --- |
| `cow_high1` | 倒挂牛 | 倒挂的牛，四条腿朝上    | 滑过去 |
| `cow_high2` | 吊牛  | 被吊绳吊着的牛       | 滑过去 |
| `cow_high3` | 举铃牛 | 高举铜铃的牛（占上方空间） | 滑过去 |

### 3.3 需要【换层】的牛（满层阻挡）

| ID         | 造型名 | 描述        | 判定   |
| ---------- | --- | --------- | ---- |
| `cow_mid1` | 立牛  | 站立的牛，正面冲撞 | 换层躲开 |
| `cow_mid2` | 独角牛 | 冲撞的独角牛    | 换层躲开 |
| `cow_mid3` | 斗牛  | 斗牛场的红斗牛   | 换层躲开 |

### 3.4 需要【特殊应对】的牛（彩蛋/高阶）

| ID          | 造型名 | 描述           | 判定        |
| ----------- | --- | ------------ | --------- |
| `cow_boss`  | 牛王  | 大型牛王，占满两层    | 必须连续两次换层  |
| `cow_candy` | 糖牛  | 糖果牛，碰到得分但会减速 | 主动撞它（陷阱式） |
| `cow_gold`  | 金牛  | 金色牛，碰+100 分  | 主动撞它      |

### 3.5 造型设计要点（面向儿童）

1. **剪影可辨**：即使缩到 60px 也能靠轮廓认出是牛（角、耳朵、尾巴、躯干块面）
2. **色彩编码**：
   - 蓝白花斑（奶牛本色）→ 常规障碍
   - 橙黄（暖色）→ 需要跳跃的低矮牛
   - 紫蓝（冷色）→ 需要滑铲的高架牛
   - 深红（危险色）→ 必须换层的阻挡牛
   - 金色 → 可主动撞的奖励牛
3. **每个牛自带提示符号**：造型上加一个小的图标（↑ / ↓ / ← →），实现「形→意」直连
4. **表情 friendly**：不出现写实凶恶，避免儿童不适

---

## 四、随机事件任务库（spx 版，保留 20 个）

沿用原型的事件设计，但按 spx 能力做等价映射：

| 原型机制      | spx 实现                                |
| --------- | ------------------------------------- |
| 金币收集任务    | `coinCount` 计数，任务箱广播 `"task", "coin"` |
| 禁触金币      | 任务期金币 sprite 隐藏，检测到收集即失败              |
| 开摆躺平（不操作） | 任务期忽略所有按键输入，检测到输入即失败                  |
| 单侧通行      | 记录 `lastLaneDir`，方向不符即失败              |
| 定点守株      | 锁定目标 Y 层，离开即失败                        |
| 护盾不破      | 碰撞时消耗护盾而非立即失败                         |
| 闪避零失误     | `dodgeCount` 累计                       |
| 连击不断      | `combo` 计时器                           |
| 双倍收益      | 任务期金币分值 ×2                            |
| 完美吃光      | 统计任务期金币总数与漏采数                         |

**速度与距离映射**：spx 中 `step` 是像素步长，`wait` 控制节奏。加速实现为 `stepLen = baseStep + accel * elapsed`，每帧 `changeXpos(-stepLen)`。反应窗口 = 障碍间距 / 左移速度。

---

## 五、精灵清单

| 精灵        | 职责                  | 关键脚本                |
| --------- | ------------------- | ------------------- |
| `Player`  | 玩家牛角色，3 层移动 + 跳 + 滑 | onKey / onStart 主循环 |
| `Cow`     | 所有牛障碍（多造型）          | onCloned 推进 + 碰撞检测  |
| `Coin`    | 金币                  | onCloned 推进         |
| `TaskBox` | 任务箱                 | onCloned 推进 + 触发任务  |
| `Stage`   | 背景、计时、计分、事件派发       | main.spx 全局变量       |
| `HUD`     | 计时器/分数/任务进度显示       | monitors + say      |

**为什么牛用单个 `Cow` 精灵 + 多造型**：`costumes[]` 数组支持每个造型独立 SVG，`SetCostume("cow_mid1")` 切换。克隆时随机选造型 → 一个脚本管全部牛，避免 12 个精灵各自重复代码。这是 spx 项目化的关键实践。

---

## 六、三条赛道的牛群配置

| 赛道    | 加速度             | 末速       | 牛群构成                                         | 任务  |
| ----- | --------------- | -------- | -------------------------------------------- | --- |
| L1 晨曦 | 0.12            | 20 m/s   | 低矮牛 40% / 高架牛 30% / 阻挡牛 30%                  | 无   |
| L2 霓虹 | 0.06            | 14 m/s   | 低矮牛 35% / 高架牛 35% / 阻挡牛 30%                  | 2 次 |
| L3 熔岩 | 0.12 + 任务期×1.08 | 21.6 m/s | 低矮牛 30% / 高架牛 30% / 阻挡牛 25% / 牛王 5% / 金牛 10% | 3 次 |

---

## 七、文件清单

```
牛奔100秒/
├── main.spx                      # 入口：全局变量 + run
├── Player.spx                    # 玩家牛
├── Cow.spx                       # 牛障碍（12 种造型）
├── Coin.spx                      # 金币
├── TaskBox.spx                   # 任务箱
├── assets/
│   ├── index.json                # 舞台：背景 + zorder + monitors
│   ├── backdrops/
│   │   ├── dawn.svg              # L1 晨曦
│   │   ├── neon.svg              # L2 霓虹
│   │   └── lava.svg              # L3 熔岩
│   └── sprites/
│       ├── Player/
│       │   ├── index.json
│       │   └── player.svg
│       ├── Cow/
│       │   ├── index.json        # costumes[] 12 项
│       │   ├── cow_low1.svg ... cow_gold.svg
│       ├── Coin/
│       │   ├── index.json
│       │   └── coin.svg
│       └── TaskBox/
│           ├── index.json
│           └── taskbox.svg
└── builder-meta.json
```

---

## 八、待实现要点备忘

- [ ] 12 个牛的 SVG 造型（统一 viewBox 便于 pivot 计算）
- [ ] 每个造型的 pivot（决定命中判定框，必须贴合视觉）
- [ ] 玩家牛 SVG（跑动用 2 帧 + 跳跃/滑铲造型）
- [ ] 三个赛道背景 SVG（横向可平铺）
- [ ] Player.spx：3 层移动 + 跳 + 滑 + 碰撞死亡
- [ ] Cow.spx：克隆推进 + 随机造型 + 碰撞回报
- [ ] TaskBox.spx：任务触发 + 20 个事件判定
- [ ] main.spx：全局状态 + 计时 + 计分 + 赛道切换
- [ ] 打包为 .xbp 并做结构校验
