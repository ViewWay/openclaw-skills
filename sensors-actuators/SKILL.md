# 传感器与执行器工程师手册

> 面向嵌入式/机器人/物联网工程师的实用参考。每个条目包含：原理 → 参数 → 接口 → 选型 → 代码。

---

# 一、环境传感器

## 1. 温湿度传感器

### DHT11 / DHT22

**原理：** 湿度元件为高分子湿度电阻（电容式），温度为 NTC 热敏电阻。单总线数字输出。

**关键参数：**

| 参数 | DHT11 | DHT22 |
|------|-------|-------|
| 温度范围 | 0~50°C | -40~80°C |
| 温度精度 | ±2°C | ±0.5°C |
| 湿度范围 | 20~90%RH | 0~100%RH |
| 湿度精度 | ±5%RH | ±2%RH |
| 采样周期 | ≥1s | ≥2s |

**接口：** 单总线（需上拉电阻 4.7kΩ），GPIO 模拟时序。

**选型：** DHT11 适合入门/低成本场景；DHT22 精度高，适合大多数项目。两者都不适合电池供电（持续上电耗电大）。

**代码 (Arduino)：**
```cpp
#include <DHT.h>
#define DHTPIN 2
#define DHTTYPE DHT22
DHT dht(DHTPIN, DHTTYPE);

void setup() {
  Serial.begin(9600);
  dht.begin();
}
void loop() {
  delay(2000);
  float h = dht.readHumidity();
  float t = dht.readTemperature();
  if (isnan(h) || isnan(t)) { Serial.println("Read error"); return; }
  Serial.printf("T=%.1f°C H=%.1f%%\n", t, h);
}
```

---

### SHT3x / SHT4x (Sensirion)

**原理：** 电容式湿度 + 带隙温度传感器，I2C 数字输出，内部 ADC。

**关键参数：**

| 参数 | SHT31 | SHT40 |
|------|-------|-------|
| 温度精度 | ±0.3°C | ±0.2°C |
| 湿度精度 | ±2%RH | ±1.8%RH |
| 供电 | 2.15~5.5V | 1.08~3.6V |
| I2C 地址 | 0x44/0x45 | 0x44 |
| 功耗(测量) | 2.7mW | 0.4mW |

**接口：** I2C（最高 1MHz）。SHT3x 有报警引脚。

**选型：** 工业级首选 SHT3x；SHT4x 更低功耗、更小封装，适合可穿戴/电池供电。

**代码 (Arduino)：**
```cpp
#include <Adafruit_SHT31.h>
Adafruit_SHT31 sht31 = Adafruit_SHT31();

void setup() {
  Serial.begin(9600);
  if (!sht31.begin(0x44)) Serial.println("SHT31 not found");
}
void loop() {
  float t = sht31.readTemperature();
  float h = sht31.readHumidity();
  Serial.printf("T=%.2f H=%.2f\n", t, h);
  delay(1000);
}
```

---

### BME280 / BME680 (Bosch)

**原理：** BME280 = 温度 + 湿度 + 气压。BME680 额外集成金属氧化物气体传感器（VOC/IAQ）。

**关键参数：**

| 参数 | BME280 | BME680 |
|------|--------|--------|
| 温度精度 | ±1.0°C | ±1.0°C |
| 湿度精度 | ±3%RH | ±3%RH |
| 气压范围 | 300~1100hPa | 300~1100hPa |
| 气压精度 | ±1.0hPa | ±0.12hPa |
| 气体 | ❌ | VOC/IAQ |
| I2C 地址 | 0x76/0x77 | 0x76/0x77 |

**接口：** I2C / SPI（CS 引脚选择）。

**选型：** 气象站、无人机高度计、室内空气质量监测首选。BME680 IAQ 需要运行 Bosch BSEC 库。

**代码 (Arduino)：**
```cpp
#include <Adafruit_BME280.h>
Adafruit_BME280 bme;

void setup() {
  if (!bme.begin(0x76)) Serial.println("BME280 not found");
}
void loop() {
  Serial.printf("T=%.1f H=%.1f P=%.0f Alt=%.1f\n",
    bme.readTemperature(), bme.readHumidity(),
    bme.readPressure() / 100.0F,
    bme.readAltitude(1013.25));
  delay(1000);
}
```

---

### AHT20 (ASAIR)

**原理：** 电容式湿度 + ASIC 温度补偿，I2C 接口，国产低成本方案。

**关键参数：**
- 温度：-40~85°C，精度 ±0.3°C
- 湿度：0~100%RH，精度 ±2%RH
- I2C 地址：0x38
- 供电：2.0~5.5V

**选型：** 性价比极高，DHT22 的替代升级。常见于国产开发板。

**代码 (Arduino)：**
```cpp
#include <Adafruit_AHTX0.h>
Adafruit_AHTX0 aht;

void setup() {
  if (!aht.begin()) Serial.println("AHT20 not found");
}
void loop() {
  sensors_event_t h, t;
  aht.getEvent(&h, &t);
  Serial.printf("T=%.1f H=%.1f\n", t.temperature, h.relative_humidity);
  delay(1000);
}
```

---

## 2. 气压传感器

### BMP280 / BMP388 (Bosch)

**原理：** 压电电阻式（MEMS 压力传感器），配合温度补偿和 ADC。

**关键参数：**

| 参数 | BMP280 | BMP388 |
|------|--------|--------|
| 范围 | 300~1100hPa | 300~1250hPa |
| 精度 | ±1.0hPa | ±0.03hPa（高精度模式）|
| 分辨率 | 0.01hPa | 0.01hPa |
| 功耗 | 2.7µA（待机）| 3.4µA（待机）|

**接口：** I2C / SPI。

**选型：** BMP280 性价比高，适合高度计、气象站。BMP388 精度更高、噪声更低，适合无人机/精密应用。

**代码 (Arduino)：**
```cpp
#include <Adafruit_BMP280.h>
Adafruit_BMP280 bmp;
void setup() {
  bmp.begin(0x76);
  bmp.setSampling(Adafruit_BMP280::MODE_NORMAL,
    Adafruit_BMP280::SAMPLING_X2, Adafruit_BMP280::SAMPLING_X16,
    Adafruit_BMP280::FILTER_X16, Adafruit_BMP280::STANDBY_MS_500);
}
void loop() {
  Serial.printf("P=%.2fhPa Alt=%.1fm\n", bmp.readPressure()/100, bmp.readAltitude(1013.25));
  delay(500);
}
```

---

### MS5611 (Measurement Specialties)

**原理：** 压阻式压力传感器 + 24bit ADC，高精度。

**关键参数：**
- 范围：10~1200mbar
- 精度：±1.5mbar（高分辨率模式）
- 分辨率：0.01mbar（24bit）
- 接口：I2C / SPI

**选型：** 无人机定高、精密气象站首选之一。比 BMP280 精度更高。

---

## 3. 气体传感器

### MQ 系列 (MQ-2/MQ-3/MQ-4/MQ-7/MQ-135)

**原理：** 金属氧化物半导体（SnO₂），目标气体在加热表面发生氧化还原反应，改变电阻值。

**关键参数（通用）：**

| 型号 | 目标气体 | 检测范围 | 加热电流 |
|------|---------|---------|---------|
| MQ-2 | 烟雾、可燃气体 | 300~10000ppm | ~150mA |
| MQ-3 | 酒精 | 0.05~10mg/L | ~150mA |
| MQ-4 | 甲烷(CH₄) | 200~10000ppm | ~150mA |
| MQ-7 | 一氧化碳(CO) | 20~2000ppm | 交替高低电流 |
| MQ-135 | 空气质量(CO₂/NH₃等) | 10~1000ppm | ~150mA |

**接口：** 模拟输出（需 ADC）+ 数字阈值输出。

**选型：** 成本极低（<¥5），但需要预热（1-5min）、需要标定、非线性输出。适合教学/原型验证，不适合精密场景。

**代码 (Arduino)：**
```cpp
const int mqPin = A0;
void setup() {
  Serial.begin(9600);
  // 预热 2 分钟
  Serial.println("Warming up...");
  delay(120000);
}
void loop() {
  int val = analogRead(mqPin);
  float ppm = map(val, 0, 1023, 0, 1000); // 需根据实际标定曲线修改
  Serial.printf("Raw=%d Approx=%.1fppm\n", val, ppm);
  delay(1000);
}
```

> ⚠️ MQ 系列实际使用需要**先在洁净空气中获取 Ro（基线电阻）**，再用 Rs/Ro 比值查数据手册曲线换算 ppm。简单 map() 不准确。

---

### SGP30 (Sensirion)

**原理：** 金属氧化物热板传感器，通过多脉冲加热循环精确测量 CO₂ 和 TVOC。

**关键参数：**
- CO₂ 等效：0~60000ppm，精度 ±(30ppm + 3%)
- TVOC：0~60000ppb
- 接口：I2C，地址 0x58
- 内置自校准算法

**选型：** 比 MQ 系列精度高很多，适合室内空气质量监测。无需手动标定。

**代码 (Arduino)：**
```cpp
#include <Adafruit_SGP30.h>
Adafruit_SGP30 sgp;

void setup() {
  if (!sgp.begin()) Serial.println("SGP30 not found");
}
void loop() {
  if (!sgp.IAQmeasure()) return;
  Serial.printf("eCO2=%d TVOC=%d\n", sgp.eCO2, sgp.TVOC);
  delay(1000);
}
```

---

### CCS811 (ams/ScioSense)

**原理：** 金属氧化物气体传感器 + MCU，直接输出 eCO₂ 和 TVOC 数字值。

**关键参数：**
- eCO₂：400~8192ppm
- TVOC：0~1187ppb
- 接口：I2C，地址 0x5A/0x5B
- 有 WAKE 和 INTn 引脚

**选型：** 类似 SGP30 但更便宜，功耗稍高。适合空气质量检测器。

---

## 4. 粉尘传感器

### PMS5003 / PM7003 (攀藤 Plantower)

**原理：** 激光散射法。气泵抽气通过激光照射区，光电二极管检测散射光强度换算颗粒物浓度。

**关键参数：**
- 检测粒径：0.3~10µm（PM1.0/PM2.5/PM10）
- PM2.5 分辨率：1µg/m³
- 接口：UART (9600bps)，3.3V
- 工作电流：≤100mA

**接口：** UART 串口，32 字节固定帧格式。

**选型：** 空气净化器/空气质量站标准方案。PMS5003 是经典款，PM7003 更小。需要主动抽气（有风扇噪声）。

**代码 (Arduino)：**
```cpp
#include <SoftwareSerial.h>
SoftwareSerial pmsSerial(2, 3); // RX, TX

struct PMS5003Data {
  uint16_t pm1_0, pm2_5, pm10;
  uint16_t pm1_0a, pm2_5a, pm10a; // 大气环境
} pms;

bool readPMS() {
  uint8_t buf[32];
  if (pmsSerial.available() < 32) return false;
  pmsSerial.readBytes(buf, 32);
  // 校验帧头 0x42 0x4D
  if (buf[0] != 0x42 || buf[1] != 0x4D) return false;
  // 校验和
  uint16_t sum = 0;
  for (int i = 0; i < 30; i++) sum += buf[i];
  if ((buf[30] << 8 | buf[31]) != sum) return false;
  pms.pm1_0  = buf[4]<<8 | buf[5];
  pms.pm2_5  = buf[6]<<8 | buf[7];
  pms.pm10   = buf[8]<<8 | buf[9];
  return true;
}

void setup() {
  Serial.begin(9600);
  pmsSerial.begin(9600);
}
void loop() {
  if (readPMS())
    Serial.printf("PM1.0=%d PM2.5=%d PM10=%d\n", pms.pm1_0, pms.pm2_5, pms.pm10);
}
```

---

### SDS011 (Nova Fitness)

**原理：** 激光散射，UART 输出。

**关键参数：**
- 检测范围：0~999µg/m³
- 分辨率：10µg/m³
- 接口：UART 9600bps
- 工作周期可配置（连续/1分钟周期）

**选型：** 成本低，适合个人空气检测项目。精度不如 PMS5003。

---

## 5. 光照传感器

### BH1750 (Rohm)

**原理：** 光电二极管 + 集成 ADC，直接输出光照度数字值（lx）。

**关键参数：**
- 范围：1~65535 lx
- 分辨率：1 lx（高分辨率模式）
- 接口：I2C，地址 0x23/0x5C
- 供电：3~5V

**选型：** 环境光感知、自动亮度调节首选。输出已经是 lx，无需换算。

**代码 (Arduino)：**
```cpp
#include <BH1750.h>
#include <Wire.h>
BH1750 lightMeter;
void setup() {
  Wire.begin();
  lightMeter.begin();
}
void loop() {
  float lux = lightMeter.readLightLevel();
  Serial.printf("Light: %.0f lx\n", lux);
  delay(1000);
}
```

---

### MAX44009 (Maxim)

**原理：** 集成环境光传感器，I2C 接口，超低功耗。

**关键参数：**
- 范围：0.045~188000 lx
- 功耗：0.65µA
- I2C 地址：0x4A/0x4B

**选型：** 电池供电场景首选（极低功耗）。

---

### 光敏电阻 (LDR)

**原理：** 硫化镉光敏电阻，光照越强电阻越小。

**关键参数：**
- 亮电阻：~5kΩ（10lx）
- 暗电阻：~200kΩ
- 响应时间：20-30ms（上升），秒级（下降）

**接口：** 分压电路 + ADC。

**选型：** 最简单的光照检测，但非线性、响应慢、有老化和温度漂移。适合简单的明暗检测。

**代码 (Arduino)：**
```cpp
const int ldrPin = A0;
void setup() { Serial.begin(9600); }
void loop() {
  int val = analogRead(ldrPin);
  Serial.printf("Light level: %d\n", val);
  delay(500);
}
```

---

## 6. 声音传感器

### MAX4466 / MAX9814

**原理：** 电容式驻极体麦克风 + 可变增益运放，模拟输出（音频波形）。

**关键参数：**
- MAX4466：增益可调（外接电阻），功耗仅 24µA
- MAX9814：固定增益 60dB，自动增益控制(AGC)，输出音频质量更好

**接口：** 模拟输出 → ADC 或比较器。

**选型：** MAX4466 做声音检测（clap 检测等）；MAX9814 做音频采集（录音等）。需要 ADC 采样率足够高。

**代码 (Arduino) — 声音检测：**
```cpp
const int micPin = A0;
const int threshold = 512 + 100; // 基于静默电平
void setup() { Serial.begin(9600); }
void loop() {
  int val = analogRead(micPin);
  if (abs(val - 512) > 100) {
    Serial.println("Sound detected!");
    delay(200); // 消抖
  }
}
```

---

## 7. 紫外线传感器

### VEML6070 (Vishay)

**原理：** 硅光电二极管，UVA 波段（320-410nm）检测。

**关键参数：**
- UVA 检测范围：0~11 (UV Index 1-11+)
- 分辨率：0.05
- 接口：I2C

**代码 (Arduino)：**
```cpp
#include <Adafruit_VEML6070.h>
Adafruit_VEML6070 uv = Adafruit_VEML6070();
void setup() { uv.begin(VEML6070_1_T); }
void loop() {
  Serial.printf("UV: %d\n", uv.readUV());
  delay(500);
}
```

---

# 二、运动与姿态传感器

## 8. 加速度计 / 陀螺仪 / IMU

### MPU6050 (InvenSense)

**原理：** 3 轴 MEMS 加速度计 + 3 轴 MEMS 陀螺仪，片上 DMP（数字运动处理器）可做传感器融合。

**关键参数：**
- 加速度：±2/4/8/16g，16bit
- 陀螺仪：±250/500/1000/2000°/s，16bit
- 接口：I2C，地址 0x68/0x69
- 自带 FIFO 和 DMP

**选型：** 最经典的 6 轴 IMU，资料多、社区活跃。适合平衡车、无人机、运动检测。

**代码 (Arduino)：**
```cpp
#include <MPU6050_light.h>
MPU6050 mpu(Wire);

void setup() {
  Wire.begin();
  mpu.begin();
  mpu.calcOffsets(); // 自动校准
}
void loop() {
  mpu.update();
  Serial.printf("A: %.2f %.2f %.2f G: %.2f %.2f %.2f\n",
    mpu.getAccX(), mpu.getAccY(), mpu.getAccZ(),
    mpu.getGyroX(), mpu.getGyroY(), mpu.getGyroZ());
  delay(50);
}
```

---

### MPU9250 / ICM-20948 (9 轴)

**原理：** MPU9250 = 3 轴加速度 + 3 轴陀螺 + AK8963 3 轴磁力计。ICM-20948 是其继任者。

**选型：** 需要 9 轴数据（含航向角）时使用。ICM-20948 更新、更省电。

---

### ADXL345

**原理：** 3 轴数字加速度计，支持自由落体、敲击、活动检测。

**关键参数：**
- 范围：±2/4/8/16g
- 分辨率：13bit（4mg/LSB @ ±16g）
- 接口：SPI / I2C
- 中断：支持双击/自由落体/活动检测

**选型：** 适合低功耗场景（检测运动/倾斜），自带运动检测中断可唤醒 MCU。

---

### BMI160 (Bosch)

**原理：** 3 轴加速度 + 3 轴陀螺，超低功耗。

**关键参数：**
- 加速度噪声：1.3mg/√Hz
- 功耗：正常模式 < 1mA，低功耗模式 < 0.5mA
- 接口：I2C / SPI

**选型：** 可穿戴设备首选（低功耗 + 小封装）。

---

## 9. 磁力计

### HMC5883L / QMC5883L

**原理：** 各向异性磁阻（AMR）传感器，检测地磁场方向。

**关键参数：**
- 范围：±1.3~8.3 Gauss
- 精度：1~2°
- 接口：I2C

**选型：** 电子罗盘/指南针。HMC5883L 已停产，QMC5883L 是国产替代。

> ⚠️ 磁力计需要**硬铁校准**（八字校准法）才能准确。

---

## 10. GPS / 北斗

### u-blox NEO-6M / NEO-M8N

**原理：** 接收 GPS/GNSS 卫星信号，解算位置、速度、时间。

**关键参数：**

| 参数 | NEO-6M | NEO-M8N |
|------|--------|---------|
| 定位精度 | ~2.5m | ~2.0m |
| 卫星系统 | GPS | GPS+GLONASS+BeiDou |
| 冷启动 | ~34s | ~26s |
| 接口 | UART 9600bps | UART 9600/115200bps |

**选型：** NEO-M8N 支持多星座，搜星更快。两者都通过 NMEA 协议输出。

**代码 (Arduino)：**
```cpp
#include <TinyGPS++.h>
#include <SoftwareSerial.h>
TinyGPSPlus gps;
SoftwareSerial ss(4, 3); // RX, TX

void setup() {
  ss.begin(9600);
  Serial.begin(9600);
}
void loop() {
  while (ss.available() > 0)
    gps.encode(ss.read());
  if (gps.location.isUpdated())
    Serial.printf("Lat=%.6f Lon=%.6f\n",
      gps.location.lat(), gps.location.lng());
}
```

---

### ATGM336H (中科微)

**原理：** 国产北斗/GPS 双模模块。

**选型：** 国产替代，成本低，支持北斗 B1I + GPS L1，性能与 NEO-6M 相当。

---

# 三、距离与位置传感器

## 11. 超声波测距

### HC-SR04 / JSN-SR04T

**原理：** 发射 40kHz 超声波脉冲，测量回波时间计算距离。

**关键参数：**
- 范围：2cm~400cm（HC-SR04），2cm~450cm（JSN-SR04T）
- 精度：±3mm
- 接口：Trig(触发) + Echo(回波)，GPIO

**选型：** HC-SR04 最便宜但非防水。JSN-SR04T 探头可外置、防水，适合户外/水下。

**代码 (Arduino)：**
```cpp
const int trigPin = 9, echoPin = 10;
void setup() {
  pinMode(trigPin, OUTPUT); pinMode(echoPin, INPUT);
  Serial.begin(9600);
}
void loop() {
  digitalWrite(trigPin, LOW); delayMicroseconds(2);
  digitalWrite(trigPin, HIGH); delayMicroseconds(10);
  digitalWrite(trigPin, LOW);
  long duration = pulseIn(echoPin, HIGH);
  float dist = duration * 0.034 / 2; // 声速 340m/s
  Serial.printf("Distance: %.1f cm\n", dist);
  delay(100);
}
```

---

## 12. 激光测距

### VL53L0X / VL53L1X (ST ToF)

**原理：** 飞行时间(ToF) 测距，发射 940nm 红外激光，测量光子往返时间。

**关键参数：**

| 参数 | VL53L0X | VL53L1X |
|------|---------|---------|
| 范围 | ~2~200cm | ~4~400cm |
| 精度 | ±3mm | ±5mm |
| 测量速度 | ~30Hz | ~50Hz |
| 接口 | I2C (0x29) | I2C (0x29) |

**选型：** 短距离精密测距首选。VL53L1X 范围更远、有区域可编程。用于避障、手势检测。

**代码 (Arduino)：**
```cpp
#include <Adafruit_VL53L0X.h>
Adafruit_VL53L0X lox = Adafruit_VL53L0X();
void setup() {
  if (!lox.begin()) Serial.println("VL53L0X not found");
}
void loop() {
  VL53L0X_RangingMeasurementData_t measure;
  lox.rangingTest(&measure, false);
  if (measure.RangeStatus != 4)
    Serial.printf("Distance: %d mm\n", measure.RangeMilliMeter);
  delay(100);
}
```

---

### TFmini (北醒 Benewake)

**原理：** ToF 测距，UART/I2C 输出。

**关键参数：**
- 范围：10cm~12m
- 精度：±5cm（@6m 内 ±1cm）
- 帧率：100Hz
- 接口：UART 115200bps

**选型：** 无人机定高、中距离测距。比 VL53L 系列远得多。

---

## 13. 红外测距

### Sharp GP2Y0A21YK / GP2Y0A02YK

**原理：** 三角测量法。红外 LED 发射，PSD 位置敏感检测器接收，角度换算距离。

**关键参数：**

| 型号 | 范围 | 输出 |
|------|------|------|
| GP2Y0A21YK | 10~80cm | 模拟电压 |
| GP2Y0A02YK | 20~150cm | 模拟电压 |

**接口：** 模拟输出（0.4~3.1V）→ ADC。

**选型：** 非接触距离检测，适合避障。输出非线性，需要查表/拟合曲线。

---

## 14. 接近传感器

### VCNL4010 / APDS-9960

**原理：** VCNL4010 = 红外 LED + 光电二极管，检测反射光强。APDS-9960 = 红外接近 + RGB + 手势。

**选型：** VCNL4010 做物体接近检测（3-200mm）。APDS-9960 额外支持手势识别（上/下/左/右）。

---

# 四、力与压力传感器

## 15. HX711 称重模块

**原理：** 24bit 高精度 ADC，专用于桥式称重传感器（应变片）。

**关键参数：**
- 通道：2 通道（A/B），A 通道 128 增益，B 通道 64 增益
- 速率：10/80 SPS
- 接口：4 线（DOUT/ PD_SCK/ VCC/ GND），GPIO 模拟时序

**选型：** 电子秤、称重系统必备。配合 5kg/10kg/50kg 称重传感器使用。

**代码 (Arduino)：**
```cpp
#include <HX711.h>
HX711 scale;
#define DOUT 3
#define CLK  2

void setup() {
  scale.begin(DOUT, CLK);
  scale.set_scale(2280.f); // 需要标定后填入
  scale.tare(); //去皮
}
void loop() {
  Serial.printf("Weight: %.1f g\n", scale.get_units(5));
  delay(200);
}
```

> 标定方法：放已知重量砝码，计算 `scale_value = raw_value / known_weight`。

---

### FSR 力敏电阻 (FSR-402)

**原理：** 导电聚合物薄膜，压力越大电阻越小。

**关键参数：**
- 力范围：0~20N（FSR-402）
- 接口：分压电路 + ADC

**选型：** 抓握检测、交互式装置、简单力传感。非线性、有滞后，不适合精密测量。

---

### Flex Sensor (弯曲传感器)

**原理：** 导电碳墨层，弯曲时电阻增大。

**关键参数：**
- 平直电阻：~10kΩ
- 弯曲 90° 电阻：~30~40kΩ
- 寿命：>100万次

**选型：** 手套控制器、角度检测、交互装置。

---

## 16. 水位传感器

### 电容式水位传感器

**原理：** 两个电极板，水位变化改变电容值。

**选型：** 相比电阻式，电容式不腐蚀电极，寿命更长。输出模拟电压 → ADC。

---

# 五、生物与化学传感器

## 17. MAX30102 (PPG / 心率 / 血氧)

**原理：** 光电容积脉搏波描记法(PPG)。红光 LED(660nm) + 红外 LED(880nm) + 光电二极管。

**关键参数：**
- LED 电流：可编程 0~50mA
- ADC：18bit
- 采样率：50~3200 SPS
- 接口：I2C，地址 0x57

**选型：** 心率 + SpO₂ 测量。算法需要自己实现或使用开源库（如 MAX30105 库的算法）。

**代码 (Arduino)：**
```cpp
#include <MAX30105.h>
#include "heartRate.h"
MAX30105 particleSensor;

void setup() {
  particleSensor.begin(Wire, I2C_SPEED_FAST);
  particleSensor.setup();
  particleSensor.setPulseAmplitudeRed(0x0A);
}
void loop() {
  long irValue = particleSensor.getIR();
  if (checkForBeat(irValue)) {
    // 使用 heartRate 库处理
  }
}
```

---

## 18. 土壤湿度传感器

### 电容式土壤湿度传感器

**原理：** 电极间电容随土壤介电常数变化（含水量越高介电常数越大）。

**关键参数：**
- 输出：模拟电压 0~3.3V
- 供电：3.3~5.5V

**选型：** 相比电阻式，电容式不腐蚀电极、长期稳定。强烈推荐。

**代码 (Arduino)：**
```cpp
const int soilPin = A0;
const int airValue = 3200;  // 干燥时 ADC 值（需标定）
const int waterValue = 1300; // 浸水时 ADC 值
void setup() { Serial.begin(9600); }
void loop() {
  int val = analogRead(soilPin);
  int moisture = map(val, airValue, waterValue, 0, 100);
  moisture = constrain(moisture, 0, 100);
  Serial.printf("Soil moisture: %d%%\n", moisture);
  delay(1000);
}
```

---

## 19. pH 传感器

### 模拟 pH 传感器模块

**原理：** pH 玻璃电极产生毫伏级电压（~59mV/pH），模块放大后输出。

**关键参数：**
- 范围：pH 0~14
- 精度：±0.1pH
- 输出：模拟电压

**选型：** 水质监测、水培系统。需要定期校准（pH 4.0/7.0/10.0 标准液）。

---

# 六、电流电压传感器

## 20. ACS712 电流传感器

**原理：** 霍尔效应。电流流过铜导体产生的磁场被霍尔元件检测。

**关键参数：**

| 型号 | 量程 | 灵敏度 |
|------|------|--------|
| ACS712-05A | ±5A | 185mV/A |
| ACS712-20A | ±20A | 100mV/A |
| ACS712-30A | ±30A | 66mV/A |

**接口：** 模拟输出（Vcc/2 = 0A），需 ADC。

**代码 (Arduino)：**
```cpp
const int acsPin = A0;
const float sensitivity = 0.185; // 5A 型
const float vRef = 5.0;
void setup() { Serial.begin(9600); }
void loop() {
  float voltage = analogRead(acsPin) * vRef / 1024.0;
  float current = (voltage - vRef / 2) / sensitivity;
  Serial.printf("Current: %.2f A\n", current);
  delay(500);
}
```

---

### INA219 高精度电流/电压传感器

**原理：** 高边电流检测 + 12bit ADC，I2C 数字输出。

**关键参数：**
- 电压范围：0~26V
- 电流范围：取决于分流电阻（默认 ±3.2A）
- 分辨率：0.1mA（电流）、4mV（电压）
- 接口：I2C，地址 0x40~0x4F（A0/A1 地址引脚）

**选型：** 电池监测、功耗分析首选。比 ACS712 精度高、无零漂问题。

**代码 (Arduino)：**
```cpp
#include <Adafruit_INA219.h>
Adafruit_INA219 ina219;
void setup() {
  ina219.begin();
  ina219.setCalibration_32V_2A(); // 根据量程选择
}
void loop() {
  float shuntV = ina219.getShuntVoltage_mV();
  float busV   = ina219.getBusVoltage_V();
  float current = ina219.getCurrent_mA();
  float power  = busV * current / 1000;
  Serial.printf("V=%.2f I=%.0fmA P=%.2fW\n", busV, current, power);
  delay(1000);
}
```

---

### SCT-013 非接触电流互感器

**原理：** 穿心式电流互感器（CT），被测导线穿过磁环。

**关键参数：**
- SCT-013-000：100A:50mA（需接负载电阻 + ADC）
- SCT-013-030：30A/1V（内置采样电阻，直接 ADC）

**接口：** 模拟输出，需要整流/偏置电路。

**选型：** 家用电力监测。需配合整流电路或有效值转换芯片（如 AD736）。

---

# 七、其他传感器

## 21. PIR 人体红外

### HC-SR501 / AM312

**原理：** 热释电红外传感器(PIR)，检测人体红外辐射变化。

**关键参数：**
- HC-SR501：检测范围 ~7m/120°，可调灵敏度和延时
- AM312：更小，固定参数，~3m/100°
- 接口：单 GPIO 数字输出

**选型：** HC-SR501 可调参数多，适合安防。AM312 小体积，适合低功耗。

**代码 (Arduino)：**
```cpp
const int pirPin = 2;
void setup() {
  pinMode(pirPin, INPUT);
  Serial.begin(9600);
}
void loop() {
  if (digitalRead(pirPin) == HIGH)
    Serial.println("Motion detected!");
  delay(500);
}
```

---

## 22. 霍尔传感器

### A3144（开关型） / 49E（线性型） / AS5048A（编码器）

**原理：** 霍尔效应，磁场变化产生电压。

**关键参数：**
- A3144：数字开关输出，检测磁场极性，需磁铁配合
- 49E：模拟线性输出，灵敏度 ~2.5mV/G
- AS5048A：14bit 角度编码器，I2C/SPI

**选型：** A3144 做限位/转速检测。49E 做磁场/电流检测。AS5048A 做电机角度反馈。

---

## 23. RFID / NFC

### RC522 (SPI) / PN532 (I2C/SPI/UART)

**原理：** 13.56MHz RFID 读写，RC522 用 SPI，PN532 支持多种接口。

**关键参数：**
- 支持：MIFARE 1K/4K、NTAG213/215/216 等
- 读取距离：~5cm（取决于天线）
- RC522：SPI 接口
- PN532：I2C(0x24) / SPI / UART

**选型：** RC522 资料多、便宜。PN532 支持 P2P 模式（NFC），功能更强。

**代码 (Arduino) — RC522：**
```cpp
#include <MFRC522.h>
#define SS_PIN 10
#define RST_PIN 9
MFRC522 rfid(SS_PIN, RST_PIN);
void setup() {
  SPI.begin();
  rfid.PCD_Init();
  Serial.begin(9600);
}
void loop() {
  if (!rfid.PICC_IsNewCardPresent()) return;
  if (!rfid.PICC_ReadCardSerial()) return;
  Serial.print("UID:");
  for (byte i = 0; i < rfid.uid.size; i++)
    Serial.printf(" %02X", rfid.uid.uidByte[i]);
  Serial.println();
  rfid.PICC_HaltA();
}
```

---

## 24. 水流量传感器

### YF-S201

**原理：** 水流推动叶轮旋转，霍尔传感器输出脉冲。

**关键参数：**
- 流量范围：1~30 L/min
- 精度：±10%
- 脉冲系数：约 450 脉冲/升（1Hz = 1L/450s）
- 接口：GPIO 中断计数

**代码 (Arduino)：**
```cpp
const int flowPin = 2;
volatile int pulseCount = 0;
float flowRate;
void pulse() { pulseCount++; }
void setup() {
  pinMode(flowPin, INPUT_PULLUP);
  attachInterrupt(digitalPinToInterrupt(flowPin), pulse, RISING);
  Serial.begin(9600);
}
void loop() {
  cli(); pulseCount = 0; sei();
  delay(1000);
  cli();
  flowRate = pulseCount * 1000.0 / 450.0; // mL/min
  sei();
  Serial.printf("Flow: %.0f mL/min\n", flowRate);
}
```

---

# 八、执行器 — 电机

## 25. 直流有刷电机 + L298N

**原理：** H 桥驱动，通过 PWM 控制转速和方向。

**关键参数（L298N）：**
- 电压：5~35V
- 每通道电流：2A（峰值 3A）
- 逻辑电压：5V
- 控制信号：ENA/PWM + IN1/IN2

**选型：** L298N 最经典但效率低（压降~2V）。替代方案：TB6612FNG（效率高、体积小）、DRV8833。

**代码 (Arduino)：**
```cpp
#define ENA 5
#define IN1 6
#define IN2 7
void setup() {
  pinMode(ENA, OUTPUT); pinMode(IN1, OUTPUT); pinMode(IN2, OUTPUT);
}
// speed: 0~255, dir: 1=正转 -1=反转
void motorRun(int speed, int dir) {
  if (dir > 0) { digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW); }
  else         { digitalWrite(IN1, LOW);  digitalWrite(IN2, HIGH); }
  analogWrite(ENA, abs(speed));
}
void motorStop() {
  digitalWrite(IN1, LOW); digitalWrite(IN2, LOW); analogWrite(ENA, 0);
}
```

---

## 26. 步进电机 + A4988 / TMC2209

**原理：** 通过脉冲控制步进角度（1.8°/步），驱动细分提高精度。

**关键参数：**

| 驱动 | 电流 | 细分 | 特点 |
|------|------|------|------|
| A4988 | ≤2A | 1~16 | 经典，便宜 |
| TMC2209 | ≤2A | 1~256 | 静音，StallGuard |

**选型：** A4988 入门首选。TMC2209 近乎静音，适合 3D 打印机/CNC。

**代码 (Arduino) — 步进电机：**
```cpp
#define STEP_PIN 2
#define DIR_PIN  3
const int stepsPerRev = 200;
void stepMotor(int steps, int rpm) {
  int delayUs = 60000000L / (stepsPerRev * rpm);
  digitalWrite(DIR_PIN, steps > 0 ? HIGH : LOW);
  steps = abs(steps);
  for (int i = 0; i < steps; i++) {
    digitalWrite(STEP_PIN, HIGH);
    delayMicroseconds(delayUs / 2);
    digitalWrite(STEP_PIN, LOW);
    delayMicroseconds(delayUs / 2);
  }
}
void setup() {
  pinMode(STEP_PIN, OUTPUT); pinMode(DIR_PIN, OUTPUT);
  stepMotor(1600, 60); // 转 8 圈（16 细分）, 60 RPM
}
```

---

## 27. 舵机 (SG90 / MG996R)

**原理：** PWM 信号控制角度。周期 20ms，脉宽 0.5~2.5ms 对应 0°~180°。

**关键参数：**

| 型号 | 扭矩 | 电压 | 重量 |
|------|------|------|------|
| SG90 | 1.8kg·cm | 4.8~6V | 9g |
| MG996R | 11kg·cm | 4.8~7.2V | 55g |

**选型：** SG90 适合小型项目（机器人手臂/舵机云台）。MG996R 大扭矩，适合负载较大场景。

**代码 (Arduino)：**
```cpp
#include <Servo.h>
Servo myServo;
void setup() {
  myServo.attach(9);
}
void loop() {
  myServo.write(0);   delay(1000);
  myServo.write(90);  delay(1000);
  myServo.write(180); delay(1000);
}
```

---

## 28. BLDC 无刷电机 + ESC 电调

**原理：** ESC（Electronic Speed Controller）接收 PWM/UART 信号，驱动三相无刷电机。

**关键参数：**
- PWM 信号：1000~2000µs 脉宽
- 电压：7.4V~22.2V（2S~6S 锂电）
- ESC 可选：有传感器/无传感器

**选型：** 无人机动力、电动滑板等。FOC 控制更精密（SimpleFOC 库）。

**代码 (Arduino) — 简单 PWM 控制：**
```cpp
#include <Servo.h>
Servo esc;
void setup() {
  esc.attach(9, 1000, 2000);
  esc.writeMicroseconds(1000); // 最小油门
  delay(2000); // ESC 初始化等待
}
void loop() {
  // 1000 = 停止, 1500 = 50%, 2000 = 100%
  esc.writeMicroseconds(1200);
}
```

---

# 九、执行器 — 显示

## 29. OLED SSD1306

**原理：** I2C/SPI 驱动 OLED 显示屏。

**关键参数：**
- 尺寸：0.96" (128×64) / 1.3" (128×64)
- 接口：I2C (0x3C/0x3D) 或 SPI
- 驱动芯片：SSD1306

**选型：** 最常用的嵌入式小屏幕。1.3" SH1106 驱动也不错。

**代码 (Arduino)：**
```cpp
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
Adafruit_SSD1306 display(128, 64, &Wire, -1);
void setup() {
  display.begin(SSD1306_SWITCHCAPVCC, 0x3C);
  display.clearDisplay();
  display.setTextSize(2);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 0);
  display.println("Hello!");
  display.display();
}
```

---

## 30. WS2812 (NeoPixel) RGB LED

**原理：** 单线数字协议控制颜色和亮度，每个 LED 内置控制器。

**关键参数：**
- 电压：5V（WS2812B）
- 协议：800kHz 单线，NRZ
- 每帧 24bit（G-R-B 各 8bit）
- 级联：数据从 DIN→DOUT 串联

**选型：** 氛围灯、可穿戴、状态指示。3.3V 版本可用 SK6812。

**代码 (Arduino)：**
```cpp
#include <Adafruit_NeoPixel.h>
#define PIN 6
#define NUMPIXELS 16
Adafruit_NeoPixel pixels(NUMPIXELS, PIN, NEO_GRB + NEO_KHZ800);
void setup() { pixels.begin(); }
void loop() {
  for (int i = 0; i < NUMPIXELS; i++) {
    pixels.setPixelColor(i, pixels.Color(0, 150, 0)); // 绿色
  }
  pixels.show();
  delay(500);
}
```

---

## 31. TFT 彩色屏 ILI9341 / ST7789

**原理：** SPI 并行接口驱动 TFT 液晶。

**关键参数：**
- ILI9341：2.4"/2.8" 320×240
- ST7789：1.3" 240×240, 1.54" 240×240
- 接口：SPI（~40MHz）

**选型：** ST7789 小尺寸圆形/方形屏首选。ILI9341 适合仪表盘/UI 界面。

---

## 32. 电子墨水 e-Paper (Waveshare)

**原理：** 电泳显示技术，双稳态（断电保持画面），阳光下可读。

**关键参数：**
- 刷新时间：~2~3s（全刷新）
- 寿命：~100万次刷新
- 尺寸：1.54"/2.13"/2.9"/4.2"/7.5"

**选型：** 电子价签、低功耗显示、阅读器。不适合视频/动画。

---

# 十、执行器 — 声光与开关

## 33. 蜂鸣器

**原理：** 有源蜂鸣器内置振荡电路，给电就响；无源蜂鸣器需 PWM 驱动，可控制频率。

**选型：** 有源：报警/提示音。无源：播放音乐/自定义音调。

**代码 (Arduino)：**
```cpp
#define BUZZER 8
// 无源蜂鸣器播放音符
void playTone(int freq, int duration) {
  tone(BUZZER, freq, duration);
  delay(duration);
}
void setup() {
  playTone(523, 200); // C5
  playTone(659, 200); // E5
  playTone(784, 200); // G5
}
```

---

## 34. 继电器 / MOSFET / 晶体管

### 继电器

**原理：** 电磁线圈吸合触点，隔离控制高电压/大电流。

**关键参数：**
- 电磁继电器：5V/12V 线圈，触点 10A/250VAC
- 固态继电器(SSR)：光耦隔离，零交叉触发，寿命更长

**选型：** 电磁继电器便宜、触点压降小。SSR 无机械磨损、响应快、无火花。

---

### MOSFET (IRF540N)

**原理：** 电压控制型开关，栅极电压控制漏源导通。

**关键参数：**
- Vds：100V
- Id：33A
- Rds(on)：44mΩ
- Vgs(th)：2~4V

**选型：** 逻辑电平 MOSFET（如 IRLZ44N）可直接用 3.3V/5V 驱动。IRF540N 需 10V 栅压才能完全导通。

**代码 (Arduino)：**
```cpp
#define MOSFET_PIN 9
void setup() {
  pinMode(MOSFET_PIN, OUTPUT);
}
void loop() {
  // PWM 控制负载（如 LED、电机、加热片）
  for (int i = 0; i <= 255; i++) {
    analogWrite(MOSFET_PIN, i);
    delay(10);
  }
}
```

---

### 晶体管 (S8050/S8550)

**原理：** 电流控制型开关。S8050 NPN，S8550 PNP。

**选型：** 驱动小电流负载（LED、蜂鸣器、小继电器）。基极电阻 1kΩ~10kΩ。

---

# 十一、其他执行器

## 35. 电磁阀

**原理：** 电磁线圈控制阀门开闭。

**关键参数：**
- 常闭型(NC)：断电关闭，通电打开
- 电压：12V/24V DC（直动式）/ AC
- 接口：常温水/气/油

**选型：** 自动浇水、气控系统。配合继电器/MOSFET 驱动。

---

## 36. 加热片 + PID 温控

**原理：** PTC 陶瓷加热片或电阻加热丝，配合温度传感器和 PID 算法控温。

**选型：** 3D 打印机热床、恒温箱、热饮杯。PID 参数需调试（Ziegler-Nichols 法）。

---

# 十二、信号调理

## 37. 运放电路

### 同相放大器
- 增益：Av = 1 + Rf/R1
- 输入阻抗高，适合传感器信号缓冲

### 仪表放大器 (INA128/AD620)
- 增益：G = 1 + 50kΩ/Rg
- 高 CMRR（共模抑制比），适合桥式传感器（应变片、称重）

### 差分放大器
- 抑制共模噪声，适合长线传输信号

---

## 38. 滤波电路

### RC 低通滤波器
- 截止频率：fc = 1/(2π·R·C)
- 用途：ADC 输入端抗混叠、去高频噪声
- 典型值：R=10kΩ, C=0.1µF → fc≈160Hz

### RC 高通滤波器
- 用途：去除直流偏移、低频噪声

### 有源滤波器 (Sallen-Key)
- 二阶低通/高通/带通，可用运放搭建

---

## 39. 电平转换

**3.3V ↔ 5V 常用方案：**

| 方案 | 方向 | 速度 | 备注 |
|------|------|------|------|
| 电阻分压 | 5V→3.3V | 慢 | 简单，单向 |
| MOSFET (BSS138) | 双向 | 中 | I2C/UART |
| 电平转换芯片 (TXS0108E) | 双向 | 快 | 多通道 |
| 光耦 | 单向 | 中 | 隔离 |

---

## 40. ADC 前端

- **抗混叠滤波：** RC 低通，fc < fs/2（奈奎斯特）
- **采样保持：** 高速 ADC 内置，低速可用电容保持
- **参考电压：** 精度决定 ADC 精度，推荐稳压源（如 LM4040）

---

## 41. 隔离

| 类型 | 器件 | 特点 |
|------|------|------|
| 光耦 | PC817/6N137 | 便宜、慢(ms级)/快(ns级) |
| 磁隔离 | ADuM120x/ISO774x | 快、长寿命 |
| 电容隔离 | ISO1540 | 平衡性能 |

**选型：** 光耦最便宜适合低速信号。磁/电容隔离适合高速信号（SPI/CAN）。

---

# 附录：选型速查表

| 场景 | 推荐传感器 |
|------|-----------|
| 温湿度（低成本）| DHT22 / AHT20 |
| 温湿度（高精度）| SHT40 / BME280 |
| 空气质量 | SGP30 + PMS5003 |
| 运动检测 | MPU6050 |
| 测距（短）| VL53L1X |
| 测距（中）| TFmini |
| 测距（简单）| HC-SR04 |
| 电流监测 | INA219 |
| GPS 定位 | NEO-M8N |
| 称重 | HX711 + 称重传感器 |
| 人体检测 | HC-SR501 |
| 心率 | MAX30102 |
| 土壤湿度 | 电容式土壤传感器 |

| 场景 | 推荐执行器 |
|------|-----------|
| 直流电机 | L298N / TB6612FNG |
| 步进电机 | TMC2209 |
| 舵机 | SG90 / MG996R |
| 无刷电机 | ESC + BLDC |
| 小屏幕 | SSD1306 OLED |
| 彩色屏 | ST7789 |
| RGB 灯 | WS2812 |
| 大功率开关 | 继电器 / SSR |
| 小功率开关 | MOSFET / 晶体管 |

---

# 常用库汇总

| 库 | 用途 |
|----|------|
| Adafruit_BME280 | BME280 |
| Adafruit_INA219 | INA219 |
| Adafruit_VL53L0X | VL53L0X |
| Adafruit_SSD1306 + Adafruit_GFX | OLED 显示 |
| Adafruit_NeoPixel | WS2812 |
| MPU6050_light | MPU6050 |
| HX711 | 称重 |
| MFRC522 | RC522 RFID |
| TinyGPS++ | GPS |
| AccelStepper | 步进电机高级控制 |
| Servo (内置) | 舵机 |

---

> 📌 **使用提示：** 本文档可作为嵌入式项目选型和代码快速参考。所有代码基于 Arduino 平台，移植到 ESP32/STM32/RP2040 时只需调整引脚和库。涉及精度要求高的场景，务必进行标定。
