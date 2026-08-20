# Cornix ZMK 固件功能说明

下面是根据当前仓库里的 board、shield、keymap 和配置文件整理出的功能总结。

## 一、固件整体能力

- 支持 Cornix 分体键盘的左右半区固件。
- 支持普通分体模式：
  - `cornix_left//zmk`
  - `cornix_right//zmk`
- 支持 dongle/中心件模式：
  - `cornix_ph_left//zmk`
  - `cornix_dongle_adapter`
  - `cornix_dongle_eyelash`
- 同时支持 USB 和 BLE。
- 支持 ZMK split，左侧可作为 central，右侧/phantom 左侧作为 peripheral。
- 支持 settings 存储到 NVS，适合保留蓝牙配对信息和设备状态。
- 支持 ZMK Studio（主要在 dongle 方案里启用）。
- 支持电池电量上报。
- 支持 50 键物理布局，同时仓库也保留了 42 键导出映射。

## 二、按键与层功能

### 当前板级 keymap 的主要功能

- 主层是标准 QWERTY 布局。
- 有符号层、数字层等分层输入能力。
- 支持 home-row mod：
  - 同手滚动时更偏向 tap
  - 跨手组合时更容易判定为 hold
- 支持 Space 作为“按下输出空格、长按切层”的复合行为。
- 支持 tap-dance：
  - 单击可切换数字层
  - 长按可临时进入数字层
- 支持组合键输入括号等符号。
- 支持自定义宏：
  - `M0`：快速输入一串固定文本并回车
  - `M1`：输入一段带标记的模板文本
- 支持指点设备行为：
  - 鼠标移动
  - 鼠标按键

### 旧式 `config/` keymap 里保留的能力

- 经典三层结构：`default / lower / raise`
- 还额外保留了 Windows / Symbols / Function / Number / BT 这类更细的层划分思路
- `lower` 负责数字、蓝牙配对选择、方向键
- `raise` 负责常见符号输入
- 保留蓝牙清配对、选择 profile 的入口

## 三、旋钮/传感器功能

- 板上定义了左右两个 EC11 编码器。
- 编码器在不同层上可以映射为：
  - 音量增减
  - 亮度增减
  - 滚动
- 旋转触发步进由 `triggers-per-rotation = 20` 控制。

## 四、灯光逻辑

### 默认状态

- 板级默认关闭通用 RGB underglow：
  - `CONFIG_ZMK_RGB_UNDERGLOW=n`
  - `CONFIG_ZMK_RGB_UNDERGLOW_EXT_POWER=n`
- 也就是说，固件默认不是“常亮灯条模式”。
- 指示灯也不是“闲置几分钟后自动亮”的逻辑。
- 当前配置是：灯效事件发生时点亮，空闲后自动断电。

### `cornix_indicator` 指示灯

- 这是可选的生产用 RGB 指示方案，不是默认必开。
- 使用 2 颗 WS2812 灯珠，作为同一组状态指示灯链。
- 依赖 `zmk-rgbled-widget`。
- 主要用途是：
  - 电池状态指示
  - 连接状态指示
- 亮度默认是 64。
- 共享灯珠输出已启用。
- 灯带外部供电会在指示灯空闲后 1000 ms 自动关闭，以降低待机功耗。
- 当没有动画或静态指示保持时，WS2812 电源轨会自动断开。
- 重新使用键盘本身不会直接触发“常亮灯”，只有电池/连接/上电这类状态事件会让它再次闪一下。

### 相关实现点

- `EXT_POWER` 由板级设备树节点控制。
- `cornix_indicator` 会把 `status-ws2812` 和 `widget,rgb` 指向同一条 WS2812 链。
- 这套逻辑更像“状态指示灯”，不是完整的 RGB 背光或氛围灯。
- 仓库当前没有把 `&ind_bat` 或 `&ind_con` 放到按键层里，所以没有“按键主动呼出指示灯”的专门按键。

### 颜色说明

`zmk-rgbled-widget` 的默认颜色定义如下，Cornix 目前没有在仓库里改这些默认值：

| 场景 | 颜色 | 含义 |
| --- | --- | --- |
| 电量高 | 绿 | 默认高电量，通常是 80% 以上 |
| 电量中 | 黄 | 默认中等电量，介于低电量和高电量之间 |
| 电量低/临界 | 红 | 默认低电量，临界值以下也用红 |
| 电量缺失/对端离线 | 品红 | 用于电量读取不到，或 split 对端离线 |
| 连接正常 | 蓝 | 蓝牙已连接 |
| 正在广播 | 黄 | 设备在等待连接 |
| 断开 | 红 | 蓝牙未连接 |
| USB 优先 | 青色 | 仅在显式打开 USB 指示时使用 |

### 触发时机

- 上电时会先显示电量，再显示连接状态。
- Central 侧在切换 BT profile 后也会显示连接状态。
- 低电量时，每次电量变化都可能再次闪红。
- 指示结束后，1 秒无活动会断开 WS2812 外部供电。

## 五、Dongle / 显示相关

- `cornix_dongle_adapter` 负责 dongle 的中心件角色。
- `cornix_dongle_eyelash` 可选接入 OLED 显示。
- 显示屏是 SH1106，128x64，走 I2C。
- 如果 dongle 板本身没有 `zephyr,display`，才需要挂这个 eyelash overlay。

## 六、刷机和保存

- 当前仓库走的是 NVS settings 路线，适合 split 配对保存。
- 左右半区和 dongle 最好按角色分别刷对应 UF2。
- settings reset 目标已经单独保留，便于清配对和恢复。
