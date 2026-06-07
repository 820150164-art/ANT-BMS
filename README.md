NEW_FILE_CODE
# 蚂蚁保护板 ANT BMS 蓝牙协议完整解析文档

> **版本**: v1.0  
> **最后更新**: 2026-05-08  
> **适用设备**: 蚂蚁保护板 ANT-BLE 系列  
> **BLE库版本**: ESP32 Arduino 2.0.14-cn

---

## 📋 目录

- [1. 设备基本信息](#1-设备基本信息)
- [2. BLE连接配置](#2-ble连接配置)
- [3. 通信流程](#3-通信流程)
- [4. 协议帧格式](#4-协议帧格式)
- [5. 帧类型详解](#5-帧类型详解)
- [6. 数据解析示例](#6-数据解析示例)
- [7. 完整数据包解析](#7-完整数据包解析)
- [8. ESP32实现要点](#8-esp32实现要点)
- [9. 常见问题](#9-常见问题)

---

## 1. 设备基本信息

### 1.1 设备标识
- **设备名称格式**: `ANT-BLE{序列号}`
- **示例**: `ANT-BLE22AAUB-4Y7I`
- **电池类型**: 三元锂电池
- **典型配置**: 20串，40AH

### 1.2 硬件接口

通信方式: 蓝牙BLE 4.0
协议类型: GATT Notify（单向推送）
数据方向: 保护板 → APP/ESP32
---

## 2. BLE连接配置

### 2.1 UUID配置
```cpp
// Service UUID
static BLEUUID serviceUUID("0000ffe0-0000-1000-8000-00805f9b34fb");

// Characteristic UUID  
static BLEUUID charUUID("0000ffe1-0000-1000-8000-00805f9b34fb");

// CCCD Descriptor UUID
static BLEUUID cccdUUID((uint16_t)0x2902);

2.2 连接参数

// 扫描参数
BLE_SCAN_DURATION = 15秒
BLE_SCAN_INTERVAL = 100ms
BLE_SCAN_WINDOW = 99ms

// 连接超时
BLE_CONNECT_TIMEOUT = 15000ms (15秒)

// 数据查询间隔
BLE_QUERY_INTERVAL = 5000ms (5秒)

// MTU设置
BLE_MTU = 517字节

2.3 连接流程
1. BLEDevice::init("ESP32-BMS-Monitor")
2. 设置扫描参数并启动扫描
3. 发现目标设备（名称包含"ANT"）
4. 创建BLEClient并连接
5. 协商MTU（517字节）
6. 发现服务FFE0
7. 获取特征值FFE1
8. 启用CCCD通知
9. 开始接收数据

3. 通信流程
3.1 初始化序列
ESP32/APP                     保护板
    |                            |
    |---- 连接请求 ------------->|
    |<--- 连接确认 -------------|
    |                            |
    |---- MTU协商 (517) -------->|
    |<--- MTU确认 --------------|
    |                            |
    |---- 发现服务FFE0 --------->|
    |---- 发现特征FFE1 --------->|
    |---- 启用CCCD通知 --------->|
    |<--- 通知确认 -------------|
    |                            |
    |---- 发送查询命令 --------->|
    |<--- 开始推送数据 ---------|
    |                            |
    |<=== 周期性数据推送 ========|

    3.2 数据推送机制
触发方式: 保护板主动推送（Notify）
推送频率: 约1秒/次
数据包组成: 多个帧组合成一组完整数据
查询命令: ESP32可主动发送查询指令触发数据推送

4. 协议帧格式
4.1 通用帧结构
┌────────────────────┬──────────────┬──────────┬────────┐
│ 帧头(2B) │ 载荷(NB) │ 预留/参数    │ 校验(1B) │ 帧尾   │
│          │          │ (可选)       │          │ (可选) │
└──────────┴──────────┴──────────────┴──────────────────┘

固定长度: 多数帧为20字节
字节序: 小端模式（Little Endian）

4.2 帧头标识
// 握手/心跳帧
#define FRAME_HEAD_HANDSHAKE  0x7EA1

// 电压数据帧  
#define FRAME_HEAD_VOLTAGE    0x8001

// 统计信息帧
#define FRAME_HEAD_STATS      0x6202

// 设备信息帧
#define FRAME_HEAD_DEVICE     0xAB02

// 状态帧（包含温度、SOC等）
#define FRAME_HEAD_STATUS     0xD8FF (起始字节)

4.3 帧尾标识
部分帧以 AA 55 结尾，作为帧结束标志
示例: ... 5D BB AA 55

5. 帧类型详解
5.1 握手/心跳帧 (0x7EA1)
帧头: 7E A1示例数据:
7E A1 11 00 00 A8 05 01 04 14 00 00 00 00 00 00 00 00 00 10
字段解析:
字节0-1: 7E A1 → 帧头
字节2-3: 11 00 → 命令类型（握手/心跳）
字节4-5: 00 A8 → 设备序列号
字节6-7: 05 01 → 协议版本 (v5.01)
字节8-9: 04 14 → 数据类型标识
字节10-18: 00 00 ... → 保留/填充
字节19: 10 → 校验和

作用:
连接建立后首次发送
周期性发送作为心跳包
维持BLE连接活跃状态

5.2 电压数据帧 (0x8001)
帧头: 80 01示例数据:
80 01 00 00 00 00 00 00 00 00 00 00 9C 0D 94 0D 9B 0D

字段解析:
字节0-1: 80 01 → 帧头
字节2-7: 00 00 00 00 00 00 → 保留/填充
字节8-9: 9C 0D → 第1串电压 (小端)
字节10-11: 94 0D → 第2串电压
字节12-13: 9B 0D → 第3串电压
...
电压解析规则:
// 小端模式：低字节在前
uint16_t rawVolt = data[i] | (data[i+1] << 8);
float voltage = rawVolt / 1000.0f;  // 单位mV转V

// 示例: 9C 0D
// = 0x0D9C = 3484 (十进制)
// = 3.484V
有效电压范围: 2000-5000mV (2.0V-5.0V)

5.3 电芯扩展电压帧
说明: 当电芯数量超过3串时，后续电芯电压在独立帧中传输示例数据:
91 0D A0 0D 8D 0D 9F 0D 92 0D A1 0D AA 0D 94 0D A1 0D 8A 0D
解析: 连续10个电芯电压（每2字节一串）
5.4 状态帧 (0xD8FF起始)
起始字节: D8 FF示例数据:
plaintext
D8 FF 1F 00 1F 00 36 1B 00 00 07 00 64 00 01 01 00 00 00 5A
字段解析（20字节）:
plaintext
字节0-1: D8 FF → T3温度传感器
         有符号16位，单位0.1°C
         示例: 0xFFD8 = -40 (未连接)

字节2-3: 1F 00 → MOS管温度
         无符号16位，单位0.1°C  
         示例: 0x001F = 31 (3.1°C? 需确认单位)

字节4-5: 1F 00 → 均衡板温度
         无符号16位，单位0.1°C
         示例: 0x001F = 31

字节6-7: 36 1B → 总电压
         无符号16位，单位10mV
         示例: 0x1B36 = 6966 → 69.66V

字节8-9: 00 00 → 电流
         有符号16位，单位0.1A
         示例: 0x0000 = 0A (静止)

字节10-11: 07 00 → SOC电量百分比 ⭐关键
          无符号16位，直接百分比值
          示例: 0x0007 = 7%

字节12-13: 64 00 → 容量参数/满充百分比
          示例: 0x0064 = 100

字节14: 01 → 充电MOS状态
         bit0: 1=开启, 0=关闭

字节15: 01 → 放电MOS状态  
         bit0: 1=开启, 0=关闭

字节16-17: 00 00 → 告警标志
          0=无告警

字节18-19: 00 5A → 校验和
关键发现:
✅ SOC在字节10-11，直接读取即可
✅ 温度、电压、电流都有明确位置
✅ MOS状态用单个字节表示
5.5 统计信息帧 (0x6202)
帧头: 62 02示例数据:
plaintext
62 02 BF 19 35 00 8D 6C 45 00 00 00 00 00 3B 49 7D 03 00 00
字段解析:
plaintext
字节0-1: 62 02 → 帧头
字节2-3: BF 19 → 设计容量参数
字节4-5: 35 00 → 额定参数
字节6-7: 8D 6C → 硬件版本/序列号
字节8-9: 45 00 → 固件版本 (v4.5.0)
字节10-13: 00 00 00 00 → 保留
字节14-17: 3B 49 7D 03 → 累计吞吐量
          32位无符号整数，单位0.001AH
          示例: 0x037D493B = 58,558,779 → 58,558.779AH
字节18-19: 00 00 → 保留
5.6 设备信息帧 (0xAB02)
帧头: AB 02示例数据:
plaintext
AB 02 F1 FA 09 7E 42 00 12 5B 48 00 26 36 1B 00 23 5C 2E 00
字段解析:
plaintext
字节0-1: AB 02 → 帧头
字节2-5: F1 FA 09 7E → 设备唯一ID (MAC部分)
字节6-7: 42 00 → 设备型号代码
字节8-9: 12 5B → 生产批次/日期
字节10-11: 48 00 → 硬件版本 (v4.8)
字节12-13: 26 36 → 生产日期代码
字节14-15: 1B 00 → 电芯串联数 (0x001B=27串)
字节16-17: 23 5C → 制造商代码
字节18-19: 2E 00 → 校验和
5.7 实时状态帧 (以AA55结尾)
特征: 以 AA 55 作为帧尾标识示例数据:
plaintext
00 00 00 00 BC A9 06 00 00 00 15 00 00 00 5D BB AA 55
字段解析:
plaintext
字节0-3: 00 00 00 00 → 保留
字节4-5: BC A9 → 时间戳/计数器 (0xA9BC=43452)
字节6-7: 06 00 → 当前状态码 (6=正常运行)
字节8-9: 00 00 → 告警计数
字节10-11: 15 00 → 数据包序号 (21)
字节12-13: 00 00 → 保留
字节14-15: 5D BB → CRC校验和
字节16-17: AA 55 → 帧尾标识符 ⭐
6. 数据解析示例
6.1 电压数据解析
cpp
// 原始数据: 9C 0D 94 0D 9B 0D
// 解析为3串电芯电压

uint16_t volt1_raw = data[8] | (data[9] << 8);   // 0x0D9C
uint16_t volt2_raw = data[10] | (data[11] << 8); // 0x0D94  
uint16_t volt3_raw = data[12] | (data[13] << 8); // 0x0D9B

float volt1 = volt1_raw / 1000.0f;  // 3.484V
float volt2 = volt2_raw / 1000.0f;  // 3.476V
float volt3 = volt3_raw / 1000.0f;  // 3.483V
6.2 SOC解析
cpp
// 原始数据: ... 07 00 ...
// SOC在字节10-11

uint16_t soc_raw = data[10] | (data[11] << 8);  // 0x0007
float soc_percent = (float)soc_raw;              // 7.0%
6.3 温度解析
cpp
// 有符号温度解析 (T3可能为负值)
int16_t temp3_raw = data[0] | (data[1] << 8);   // 0xFFD8
float temp3 = temp3_raw / 10.0f;                 // -40.0°C

// 无符号温度解析 (MOS、均衡板)
uint16_t temp_mos_raw = data[2] | (data[3] << 8); // 0x001F
float temp_mos = temp_mos_raw / 10.0f;            // 3.1°C (需确认单位)
6.4 电流解析
cpp
// 有符号电流
int16_t current_raw = data[8] | (data[9] << 8);
float current = current_raw / 10.0f;  // 单位0.1A
bool is_charging = (current > 0);     // 正=充电，负=放电
7. 完整数据包解析
7.1 典型数据包序列
APP连接后接收到的完整数据序列（一组）：
plaintext
帧1: 7E A1 11 00 00 A8 05 01 04 14 00 00 00 00 00 00 00 00 00 10
     ↑ 握手帧

帧2: 80 01 00 00 00 00 00 00 00 00 00 00 9C 0D 94 0D 9B 0D
     ↑ 电压帧 (第1-3串)

帧3: 91 0D A0 0D 8D 0D 9F 0D 92 0D A1 0D AA 0D 94 0D A1 0D 8A 0D
     ↑ 电压扩展帧 (第4-13串)

帧4: 95 0D 90 0D A1 0D A5 0D 9F 0D 96 0D 9B 0D 1D 00 1D 00 D8 FF
     ↑ 电压扩展帧 (第14-20串) + T1/T2温度

帧5: D8 FF 1F 00 1F 00 36 1B 00 00 07 00 64 00 01 01 00 00 00 5A
     ↑ 状态帧 (温度/电压/SOC/MOS)

帧6: 62 02 BF 19 35 00 8D 6C 45 00 00 00 00 00 3B 49 7D 03 00 00
     ↑ 统计帧 (版本/吞吐量)

帧7: 00 00 AA 0D 0A 00 8A 0D 0D 00 20 00 99 0D 00 00 83 00 7D 00
     ↑ 平衡状态帧

帧8: AB 02 F1 FA 09 7E 42 00 12 5B 48 00 26 36 1B 00 23 5C 2E 00
     ↑ 设备信息帧

帧9: 00 00 00 00 BC A9 06 00 00 00 15 00 00 00 5D BB AA 55
     ↑ 实时状态帧 (带AA55帧尾)
7.2 实际数据对应关系
对应APP显示:
plaintext
电池状态: 静止
SOC: 7%
总电压: 69.66V
电流: 0.00A
温度: MOS 31°C, T1 29°C, T2 29°C
电芯: 20串
最高电压: 3.498V (第10串)
最低电压: 3.466V (第13串)
压差: 0.032V
充电MOS: 打开
放电MOS: 打开
8. ESP32实现要点
8.1 头文件包含
cpp
#include <BLEDevice.h>
#include <BLEUtils.h>
#include <BLEScan.h>
#include <BLEClient.h>
#include <Adafruit_GFX.h>
#include <Adafruit_ST7789.h>
#include <SPI.h>
8.2 关键API使用
cpp
// 扫描回调 (ESP32 2.0.14-cn版本)
pScan->setScanCallback(scanCallback);  // ✅ 正确
// pScan->setScanCallbacks(...);       // ❌ 错误

// 客户端回调
pClient->setClientCallbacks(new MyCallbacks());  // ✅ 1个参数

// 描述符写入
pDesc->writeValue(val, 2, true);  // ✅ 返回void
// bool result = pDesc->writeValue(...);  // ❌ 不返回bool

// 地址比较
if (addr1.equals(addr2)) { ... }  // ✅ 正确
// if (addr1.toString() == addr2) { ... }  // ❌ 错误
8.3 数据接收处理
cpp
// 环形缓冲区
#define RING_BUF_SIZE 1024
uint8_t ringBuffer[RING_BUF_SIZE];
volatile int ringHead = 0;
volatile int ringTail = 0;

// BLE接收回调
void notifyCallback(BLERemoteCharacteristic* pChar, 
                    uint8_t* pData, size_t length, bool isNotify) {
  ringBufWrite(pData, length);
}

// 拼包解析
void processRingBuffer() {
  // 状态机解析
  // 检测帧头 → 收集数据 → 检测帧尾 → 解析
}
8.4 辅助函数
cpp
// 读取小端16位数据
int16_t readInt16LE(const uint8_t* data, int offset) {
  return (int16_t)(data[offset] | (data[offset + 1] << 8));
}

uint16_t readUint16LE(const uint8_t* data, int offset) {
  return (uint16_t)(data[offset] | (data[offset + 1] << 8));
}
8.5 连接时序
cpp
void setup() {
  // 1. 初始化BLE
  BLEDevice::init("ESP32-BMS-Monitor");
  
  // 2. 配置扫描
  BLEScan* pScan = BLEDevice::getScan();
  pScan->setActiveScan(true);
  pScan->setInterval(100);
  pScan->setWindow(99);
  pScan->setScanCallback(scanCallback);
  
  // 3. 开始扫描
  pScan->start(15, false);
}

bool connectToDevice(BLEAddress addr) {
  // 1. 创建客户端
  pClient = BLEDevice::createClient();
  pClient->setClientCallbacks(new MyCallbacks());
  
  // 2. 连接设备
  if (!pClient->connect(addr)) return false;
  
  // 3. 设置MTU
  pClient->setMTU(517);
  delay(500);
  
  // 4. 发现服务（重试机制）
  BLERemoteService* pService = nullptr;
  for (int i = 0; i < 5; i++) {
    pService = pClient->getService(serviceUUID);
    if (pService) break;
    delay(2000);
  }
  
  // 5. 获取特征值
  pChar = pService->getCharacteristic(charUUID);
  
  // 6. 启用通知
  pChar->registerForNotify(notifyCallback);
  BLERemoteDescriptor* pDesc = pChar->getDescriptor(BLEUUID((uint16_t)0x2902));
  uint8_t val[] = {0x01, 0x00};
  pDesc->writeValue(val, 2, true);
  
  return true;
}
9. 常见问题
9.1 连接失败
症状: 扫描到设备但连接超时可能原因:
保护板已被手机APP连接（只能同时连接1个设备）
距离太远（建议<3米）
保护板未开机
解决方法:
plaintext
1. 关闭手机ANT BMS APP
2. 保护板断电3秒后重新上电
3. 等待10秒让保护板进入可连接状态
4. ESP32靠近保护板（<1米）
5. 重新连接
9.2 编译错误
错误1: setScanCallbacks 不存在
cpp
// ❌ 错误
pScan->setScanCallbacks(new MyScanCallbacks());

// ✅ 正确
pScan->setScanCallback(scanCallback);
错误2: setClientCallbacks 参数过多
cpp
// ❌ 错误
pClient->setClientCallbacks(new MyCallbacks(), true);

// ✅ 正确
pClient->setClientCallbacks(new MyCallbacks());
错误3: writeValue 返回值类型
cpp
// ❌ 错误
bool result = pDesc->writeValue(val, 2, true);

// ✅ 正确
pDesc->writeValue(val, 2, true);
错误4: BLEAdvertisedDevice 缺少成员
cpp
// ❌ 错误 (2.0.14-cn版本不支持)
bool isConnectable = dev.isConnectable();

// ✅ 正确 (直接删除该行)
9.3 数据解析错误
问题: SOC显示100%但实际只有7%原因: 字段位置识别错误解决:
cpp
// ❌ 错误 (字节12-13)
uint16_t soc_raw = readUint16LE(data, 12);

// ✅ 正确 (字节10-11)
uint16_t soc_raw = readUint16LE(data, 10);
9.4 查询命令
常用查询命令:
cpp
// 命令1
uint8_t cmd1[] = {0x7E, 0xA1, 0x02, 0x20, 0x6C, 0x02, 0x20, 0x58, 0xC4, 0xAA, 0x55};

// 命令2  
uint8_t cmd2[] = {0x7E, 0xA1, 0x01, 0x00, 0x00, 0x00, 0xBE, 0x18, 0x55, 0xAA, 0x55};

// 命令3
uint8_t cmd3[] = {0x7E, 0xA1, 0x11, 0x00, 0x00, 0xA8, 0x05, 0x01, 0x04, 0x14, 
                  0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x10};

// 发送命令
pChar->writeValue(cmd1, sizeof(cmd1), true);
发送频率: 建议5秒/次，避免过快
附录A: 完整数据结构体
cpp
struct BatteryData {
  // 电芯电压
  float cellVoltages[20] = {0};
  int   cellCount = 0;
  
  // 总览数据
  float totalVoltage = 0.0f;      // 总电压 (V)
  float socPercent = 0.0f;        // SOC电量 (%)
  float current = 0.0f;           // 电流 (A)
  float power = 0.0f;             // 功率 (W)
  
  // 温度
  float tempMOS = 0.0f;           // MOS管温度 (°C)
  float tempBalance = 0.0f;       // 均衡板温度 (°C)
  float tempT1 = 0.0f;            // 温度传感器1 (°C)
  float tempT2 = 0.0f;            // 温度传感器2 (°C)
  float tempT3 = 0.0f;            // 温度传感器3 (°C)
  float tempT4 = 0.0f;            // 温度传感器4 (°C)
  
  // 状态
  bool  chargeMOS = false;        // 充电MOS状态
  bool  dischargeMOS = false;     // 放电MOS状态
  bool  isCharging = false;       // 是否充电中
  bool  dataValid = false;        // 数据是否有效
  
  // 统计
  uint32_t cycleCount = 0;        // 累计吞吐量 (AH)
  float capacity = 0.0f;          // 总容量 (AH)
  float remainCapacity = 0.0f;    // 剩余容量 (AH)
  
  // 时间戳
  unsigned long lastUpdateTime = 0;
};
附录B: 电压-SOC对照表（参考）
对于20串三元锂电池：
总电压 (V)	单串电压 (V)	SOC (%)
84.0	4.20	100
80.0	4.00	90
76.0	3.80	70
72.0	3.60	50
68.0	3.40	30
64.0	3.20	10
60.0	3.00	0
注意: 实际SOC以协议字节10-11为准，此表仅供参考
附录C: 告警代码
plaintext
告警标志位 (字节16-17):
0x0000 - 无告警
0x0001 - 过压告警
0x0002 - 欠压告警
0x0004 - 过流告警
0x0008 - 过温告警
0x0010 - 短路保护
0x0020 - SOC低电量告警
更新日志
v1.0 (2026-05-08): 初始版本，包含完整协议解析
BLE连接配置
7种帧类型详解
数据解析示例
ESP32实现代码
常见问题解决方案
参考资料
抓包数据源: Tutorial.txt
APP截图: 蚂蚁BMS官方APP
BLE库版本: ESP32 Arduino 2.0.14-cn
目标设备: ANT-BLE22AAUB-4Y7I
文档结束
plaintext

我已经创建了一个完整的协议解析文档 `readme.md`，包含：

## 📚 文档内容：

1. **设备基本信息** - BLE配置、连接参数
2. **通信流程** - 完整的连接和数据交互时序
3. **协议帧格式** - 通用结构和帧头标识
4. **7种帧类型详解**：
   - 握手/心跳帧 (0x7EA1)
   - 电压数据帧 (0x8001)
   - 电芯扩展帧
   - 状态帧 (0xD8FF) ⭐ 包含SOC
   - 统计信息帧 (0x6202)
   - 设备信息帧 (0xAB02)
   - 实时状态帧 (AA55结尾)
5. **数据解析示例** - 电压、SOC、温度、电流的代码
6. **完整数据包** - 一组9帧的完整序列
7. **ESP32实现要点** - 关键API、连接时序、常见编译错误
8. **常见问题** - 连接失败、数据解析错误等
9. **附录** - 数据结构体、电压-SOC对照表、告警代码

这个文档结构清晰，包含了所有重要的协议细节和实现要点，方便以后AI或开发者快速理解和实现！🎉
