# 11. Doğrulama, Kalibrasyon ve Uçuş Öncesi Riskler

Kurulum tamamlandı. Bu bölüm önce her şeyin gerçekten çalıştığını
ölçerek doğrular, sonra uçuş öncesi yapılması gerekenleri anlatır.

## 11.1 — PWM Kanallarının Doğrulanması

```bash
for c in /sys/class/pwm/pwmchip*; do
  for p in $c/pwm*/; do [ -d "$p" ] || continue
    echo "$(basename $c)/$(basename $p): period=$(cat $p/period) duty=$(cat $p/duty_cycle) en=$(cat $p/enable)"
  done
done
```

Beklenen — **7 kanal**, 20 ms periyot (= 50 Hz):

```
pwmchip0/pwm0: period=20000000 duty=0 en=1
pwmchip1/pwm0: period=20000000 duty=0 en=1
pwmchip2/pwm0: period=20000000 duty=0 en=1
pwmchip3/pwm0: period=20000000 duty=0 en=1
pwmchip3/pwm1: period=20000000 duty=0 en=1
pwmchip4/pwm0: period=20000000 duty=0 en=1
pwmchip4/pwm1: period=20000000 duty=0 en=1
```

`pwmchip5` bu listede **olmamalıdır** — orası soğutma fanıdır.

## 11.2 — Sensör Sağlığı (MAVLink)

```python
from pymavlink import mavutil

m = mavutil.mavlink_connection('tcp:192.168.7.2:5760')
m.wait_heartbeat(timeout=20)

BITS = [(1 << 0, '3D_GYRO'), (1 << 1, '3D_ACCEL'), (1 << 2, '3D_MAG'),
        (1 << 3, 'ABS_PRESSURE'), (1 << 5, 'GPS'), (1 << 15, 'MOTOR_OUTPUTS'),
        (1 << 21, 'AHRS'), (1 << 28, 'PREARM_CHECK')]

msg = m.recv_match(type='SYS_STATUS', blocking=True, timeout=25)
p = msg.onboard_control_sensors_present
e = msg.onboard_control_sensors_enabled
h = msg.onboard_control_sensors_health
print('%-16s %-8s %s' % ('SENSOR', 'ENABLED', 'HEALTHY'))
for bit, name in BITS:
    if p & bit:
        print('%-16s %-8s %s' % (name, 'yes' if e & bit else 'no',
                                 'YES' if h & bit else 'NO'))
```

Doğrulanmış çıktı:

```
present=0x5330fc0f enabled=0x50009c0f health=0x47109c0b

SENSOR           ENABLED  HEALTHY
3D_GYRO          yes      YES
3D_ACCEL         yes      YES
3D_MAG           yes      NO       <-- kalibrasyonsuz, bkz. 11.5
ABS_PRESSURE     yes      YES
MOTOR_OUTPUTS    yes      YES
AHRS             no       NO       <-- GPS yok + pusula kalibrasyonsuz, beklenen
PREARM_CHECK     yes      NO       <-- kalibrasyon yapılmadan normal
```

`GPS` satırı:

...

## 11.3 — Gerçek Sensör Verisi

```python
m.mav.request_data_stream_send(m.target_system, m.target_component,
                               mavutil.mavlink.MAV_DATA_STREAM_RAW_SENSORS, 10, 1)
print(m.recv_match(type='RAW_IMU', blocking=True))
print(m.recv_match(type='SCALED_PRESSURE', blocking=True))
```

Düz bir masada duran kart için doğrulanmış çıktı:

```
RAW_IMU {xacc: 66, yacc: -23, zacc: -1003, xgyro: -1, ygyro: -1, zgyro: 0,
         xmag: 11, ymag: -358, zmag: 862, temperature: 4334}
SCALED_PRESSURE {press_abs: 1005.66, temperature: 3891}
```

Nasıl okunur:

| Ölçüm | Değer | Yorum |
|---|---|---|
| `zacc` | `-1003` mG | Düz kartta ≈ −1 g. **Doğru** — işaret ters olsaydı `+1000` olurdu. |
| `xgyro/ygyro/zgyro` | ≈ 0 | Kart hareketsiz. Doğru. |
| `xmag/ymag/zmag` | 11 / −358 / 862 | Pusula **veri üretiyor** (sağlıksız raporlanmasına rağmen) |
| `press_abs` | 1005.66 hPa | [7. bölümdeki](07-kaynak-kodu-duzeltmeleri.md) ham okuma 1005.9 hPa idi → sürücü doğru ölçekliyor |

## 11.4 — Cihaz Kimlikleri

```python
for name in ('BARO1_DEVID', 'INS_ACC_ID', 'COMPASS_DEV_ID'):
    m.mav.param_request_read_send(m.target_system, m.target_component, name.encode(), -1)
```

```
BARO1_DEVID     = 1638658      bus_type=SPI  bus=0 addr=0x01 devtype=0x19
INS_ACC_ID      = 2884354      bus_type=SPI  bus=0 addr=0x03 devtype=0x2C
COMPASS_DEV_ID  = 590594       bus_type=SPI  bus=0 addr=0x03 devtype=0x09
```

Bit yerleşimi:

```python
bus_type = v & 0x07          # 1=I2C 2=SPI 3=DRONECAN 4=SITL
bus      = (v >> 3)  & 0x1F
address  = (v >> 8)  & 0xFF
devtype  = (v >> 16) & 0xFF
```

| Alan | Çözümü |
|---|---|
| baro devtype `0x19` | `DEVTYPE_BARO_LPS22DF` — bu çalışmada eklenmiştir |
| baro addr `0x01` | SPI bus 0, CS 1 → `/dev/spidev0.1` ✔ |
| ins devtype `0x2C` | `DEVTYPE_INS_ICM20948` ✔ |
| compass devtype `0x09` | `DEVTYPE_AK0991x` (AK09916) ✔ |

Ya da komut satırından:

```bash
python3 Tools/scripts/decode_devid.py 1638658 BARO
# bus_type:SPI(2)  bus:0 address:1(0x1) devtype:25(0x19)
```

## 11.5 — Uçuş Öncesi Kalibrasyon

Kurulum bittiğinde araç **hâlâ arm olmaz**. Bu normaldir — sıradaki
adımlar fiziksel müdahale gerektirir ve yer istasyonundan yapılır.

1. **Gövde tipi** — `FRAME_CLASS` / `FRAME_TYPE` gerçek gövdenize göre
   ayarlanır. Varsayılan dosya Quad/X varsayar.
2. **İvmeölçer kalibrasyonu** — Mission Planner → *Setup → Mandatory
   Hardware → Accel Calibration*. Kartı 6 yönde tutmanız istenir.
3. **Pusula kalibrasyonu** — *Compass*. Kartı her eksende
   döndürürsünüz.
4. **Radyo kalibrasyonu** — kumanda bağlıysa.

### Pusula Neden "Sağlıksız" Görünüyor?

11.2'de `3D_MAG` sağlıksız çıkar ama 11.3'te veri üretir. Bu bir
çelişki değildir:

```
COMPASS_OFS_X  0.0
COMPASS_OFS_Y  0.0
COMPASS_OFS_Z  0.0     <-- hepsi sıfır = hiç kalibre edilmemiş
```

Kalibre edilmemiş pusula tutarlılık (consistency) kontrolünü geçemez,
ArduPilot da onu sağlıksız raporlar. Kalibrasyondan sonra düzelir.

Ölçüm bunu destekler: yatay bileşen 358, dikey 862 → eğim
(inclination) `atan(862/358) ≈ 67°`. Türkiye için beklenen ~58–60°.
Fark, kalibre edilmemiş hard-iron kaymasındandır. Sapma büyük değildir
— **eksen işaretleri doğrudur**, yalnızca ofset vardır.

### Arm Denemesi ve Hata Mesajını Okuma

```python
m.mav.command_long_send(m.target_system, m.target_component,
                        mavutil.mavlink.MAV_CMD_COMPONENT_ARM_DISARM,
                        0, 1, 0, 0, 0, 0, 0, 0)
while True:
    msg = m.recv_match(blocking=True, timeout=2)
    if msg and msg.get_type() == 'STATUSTEXT':
        print('[%d] %s' % (msg.severity, msg.text))
```

**`FRAME_CLASS` ayarlanmadan önce:**

```
ARM ACK result=4
[2] Arm: Motors: Check frame class and type
```

Bu mesaj **sensörlerle ilgili değildir** — yalnızca gövde tipinin
tanımlanmadığını söyler.

**`FRAME_CLASS=1` uygulandıktan sonra (doğrulanmış çıktı):**

```
ARM ACK result=4
[2] PreArm: RC not found
[2] PreArm: 3D Accel calibration needed
```

Engel artık gövde tanımı değil, **fiziksel müdahale gerektiren iki
adımdır**: kumanda bağlanması ve ivmeölçer kalibrasyonu. Bu, kurulumun
yazılım tarafının bittiği anlamına gelir — mesajın buraya kadar
ilerlemesi doğru işarettir.

> Parametre varsayılan dosyasının etkili olması için ArduPilot'un
> **yeniden başlatılması** gerekir. Değişikliğin uygulandığını teyit
> edin:
>
> ```python
> m.mav.param_request_read_send(m.target_system, m.target_component, b'FRAME_CLASS', -1)
> print(m.recv_match(type='PARAM_VALUE', blocking=True).param_value)   # 1.0
> ```

## 11.6 — Sensör Oryantasyonu

`hwdef.dat`:

```
IMU     Invensensev2 SPI:icm20948 ROTATION_ROLL_180_YAW_90
COMPASS AK09916:probe_ICM20948 0 ROTATION_ROLL_180
```

Ölçümle doğrulananlar:

| Bileşen | Durum | Kanıt |
|---|---|---|
| IMU roll/pitch bileşeni | ✔ doğru | Düz kartta `zacc = -1003` mG |
| Pusula Z işareti | ✔ doğru | Eğim ≈ 67°, Türkiye beklentisi ~58–60° ile uyumlu |
| IMU yaw bileşeni | ... | ... |
| Pusula yaw bileşeni | ... | ... |

**Datasheet'in söylediği (DS-000189 Rev 1.6, §15, Şekil 12–13):**
ICM-20948 içindeki AK09916 manyetometrenin eksenleri jiro/ivme
eksenlerinden farklıdır — `mag_X = +accel_X`, `mag_Y = −accel_Y`,
`mag_Z = −accel_Z`. Yani manyetometre çerçevesi, jiro çerçevesine göre
içsel olarak **ROLL_180** kadar dönüktür.

`AP_Compass_AK09916.cpp` ham register değerlerini doğrudan alır —

```cpp
raw_field = Vector3f(regs.val[0], regs.val[1], regs.val[2]);
```

— ve hwdef'teki dönüşü uygular (`set_rotation(_rotation)`). Dolayısıyla
içsel farkı hwdef satırının hesaba katması gerekir.

**Ampirik test:**

1. Kartı yer istasyonuna bağlayın, *Flight Data* ekranında yapay ufku
   açın
2. Kartı burnu yukarı kaldırın → **pitch artmalı**
3. Sağa yatırın → **roll sağa gitmeli**
4. Saat yönünde çevirin → **heading artmalı** (0° kuzey)

Tepki beklenenden farklıysa `hwdef.dat`'taki `ROTATION_*` değerini
düzeltip yeniden derleyin. Değerler `libraries/AP_Math/rotations.h`
içinde listelidir; dönüşüm matrisleri `libraries/AP_Math/vector3.cpp`
içindedir:

```cpp
case ROTATION_YAW_90:          tmp = x; x = -y; y = tmp;            // (x,y,z) -> (-y, x,  z)
case ROTATION_ROLL_180:        y = -y; z = -z;                      // (x,y,z) -> ( x,-y, -z)
case ROTATION_ROLL_180_YAW_90: tmp = x; x = y; y = tmp; z = -z;     // (x,y,z) -> ( y, x, -z)
```

Test sonucu:

...

### PWM Pinlerinin Fiziksel Ölçümü

[8. bölümdeki](08-device-tree-overlay-pwm.md) kanal→pin haritası device
tree takma adlarından (`hat-29`, `hat-08`, …) türetilmiştir ve yazılım
tarafı çalışmaktadır. ESC bağlamadan önce her kanalı tek tek ölçün.

Ölçüm sonucu:

...

### CAN Arayüzü

```bash
ip -d link show can0
```

```
4: can0: <NOARP,ECHO> mtu 16 qdisc noop state DOWN
   can state STOPPED  ... m_can ... clock 80000000 ... parentdev 20701000.can
```

Controller (`m_can`) tanınır, TCAN1462-Q1 salt fiziksel transceiver
olduğu için sürücü gerektirmez.

Arayüzün ayağa kaldırılması ve bit hızı ayarı:

...
## 11.7 — Kartta Kalıcı Olarak Bırakılanlar

| Yol | İçerik |
|---|---|
| `/opt/ardupilot/bin/arducopter` | Statik linkli ArduCopter binary'si |
| `/opt/ardupilot/etc/ardupilot.parm` | Parametre varsayılanları |
| `/opt/ardupilot/var/{log,terrain,storage}` | Çalışma dizinleri |
| `/etc/systemd/system/ardupilot.service` | Servis — **enabled**, açılışta otomatik başlar |
| `/boot/uEnv.txt` | 7 PWM overlay'i etkin |
| `/boot/uEnv.txt.bak` | Değişiklik öncesi yedek |

Servisi durdurmak / otomatik başlatmayı kapatmak:

```bash
sudo systemctl stop ardupilot
sudo systemctl disable ardupilot
```

Değiştirilmeyenler: kernel, `cooling_fan` yapılandırması, ana device
tree (`.dtb`), `/boot/overlays/` altındaki dosyalar.

---

Bir hatayla karşılaşırsanız [12-sorun-giderme.md](12-sorun-giderme.md),
başa dönmek için [README.md](../README.md).
