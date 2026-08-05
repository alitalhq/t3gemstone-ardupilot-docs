# 7. Gereken Kaynak Kodu Düzeltmeleri

Bu kartı çalışır hâle getirmek için ArduPilot kaynağında **iki**
düzeltme gerekmiştir. Bu düzeltmeler yapılmadan derlenen binary kartta
ya barometreyi bulamaz ya da başlangıçta çöker.

## 7.1 — LPS22DF Barometre Desteği

**Belirti:** Barometre hiç bulunamaz, ArduPilot "no barometer" ile
takılır.

**Sebep:**

```c
// libraries/AP_Baro/AP_Baro_LPS2XH.cpp
#define LPS22HB_WHOAMI 0xB1
#define LPS25HB_WHOAMI 0xBD
```

`_check_whoami()` yalnızca bu iki değeri kabul eder. Kartımızdaki çip
`0xB4` döndürdüğü için reddedilir.

### Düzeltmeden Önce: Register Yerleşimini Kartta Kanıtlayın

Sürücü koduna geçmeden önce LPS22DF'in register haritasının
varsayıldığı gibi olduğu kartta doğrulanmıştır. Aşağıdaki betik
`CTRL_REG1`/`CTRL_REG2`'ye yazar ve geri okur:

```python
#!/usr/bin/env python3
"""Verify LPS22DF register layout on-board before committing to driver code."""
import array, fcntl, struct, sys, time

SPI_IOC_WR_MODE = 0x40016B01
SPI_IOC_MESSAGE_1 = 0x40206B00

REG_WHO_AM_I = 0x0F
REG_CTRL_REG1 = 0x10
REG_CTRL_REG2 = 0x11
REG_STATUS = 0x27
REG_PRESS_OUT_XL = 0x28
REG_TEMP_OUT_L = 0x2B

WHOAMI_LPS22DF = 0xB4

# CTRL_REG1: ODR[6:3] = 0110 (75 Hz), AVG[2:0] = 010 (16 samples)
CTRL_REG1_VALUE = (0b0110 << 3) | 0b010
# CTRL_REG2: BDU (bit 3) only -- deliberately not touching IF_CTRL/SIM
CTRL_REG2_VALUE = 1 << 3

PRESSURE_LSB_PER_HPA = 4096.0
TEMP_LSB_PER_C = 100.0


def xfer(fd, txbytes, speed_hz=1000000):
    tx = array.array('B', txbytes)
    rx = array.array('B', [0] * len(txbytes))
    tx_addr, _ = tx.buffer_info()
    rx_addr, _ = rx.buffer_info()
    msg = struct.pack('QQIIHBBBBBB', tx_addr, rx_addr, len(txbytes),
                      speed_hz, 0, 8, 0, 0, 0, 0, 0)
    fcntl.ioctl(fd, SPI_IOC_MESSAGE_1, msg)
    return list(rx)


def read_regs(fd, reg, count):
    return xfer(fd, [reg | 0x80] + [0x00] * count)[1:]


def write_reg(fd, reg, value):
    xfer(fd, [reg, value])


def main(path):
    with open(path, 'rb+', buffering=0) as fd:
        fcntl.ioctl(fd, SPI_IOC_WR_MODE, struct.pack('B', 3))

        whoami = read_regs(fd, REG_WHO_AM_I, 1)[0]
        print('WHO_AM_I         = 0x%02X (expect 0x%02X)' % (whoami, WHOAMI_LPS22DF))
        if whoami != WHOAMI_LPS22DF:
            print('unexpected chip, aborting')
            return 1

        write_reg(fd, REG_CTRL_REG2, CTRL_REG2_VALUE)
        write_reg(fd, REG_CTRL_REG1, CTRL_REG1_VALUE)
        time.sleep(0.1)

        print('CTRL_REG1 wrote 0x%02X, read 0x%02X'
              % (CTRL_REG1_VALUE, read_regs(fd, REG_CTRL_REG1, 1)[0]))
        print('CTRL_REG2 wrote 0x%02X, read 0x%02X'
              % (CTRL_REG2_VALUE, read_regs(fd, REG_CTRL_REG2, 1)[0]))

        for i in range(5):
            time.sleep(0.05)
            status = read_regs(fd, REG_STATUS, 1)[0]
            p = read_regs(fd, REG_PRESS_OUT_XL, 3)
            t = read_regs(fd, REG_TEMP_OUT_L, 2)

            raw_p = p[0] | (p[1] << 8) | (p[2] << 16)
            if raw_p & 0x800000:
                raw_p -= 1 << 24
            raw_t = t[0] | (t[1] << 8)
            if raw_t & 0x8000:
                raw_t -= 1 << 16

            print('[%d] STATUS=0x%02X raw_p=%d -> %.2f hPa | raw_t=%d -> %.2f C'
                  % (i, status, raw_p, raw_p / PRESSURE_LSB_PER_HPA,
                     raw_t, raw_t / TEMP_LSB_PER_C))
    return 0


if __name__ == '__main__':
    sys.exit(main(sys.argv[1] if len(sys.argv) > 1 else '/dev/spidev0.1'))
```

Doğrulanmış çıktı:

```
WHO_AM_I         = 0xB4 (expect 0xB4)
CTRL_REG1 wrote 0x32, read 0x32
CTRL_REG2 wrote 0x08, read 0x08
[0] STATUS=0x33 raw_p=4120096 -> 1005.88 hPa | raw_t=4506 -> 45.06 C
[1] STATUS=0x33 raw_p=4120421 -> 1005.96 hPa | raw_t=4506 -> 45.06 C
[2] STATUS=0x33 raw_p=4120269 -> 1005.93 hPa | raw_t=4507 -> 45.07 C
```

Yazılan değerlerin aynen geri okunması ve basıncın makul aralıkta,
canlı biçimde değişiyor olması register yerleşimini kanıtlar.

Datasheet (ST DS13316 Rev 3) ile çapraz kontrol:

| Öğe | Datasheet | Uyum |
|---|---|---|
| `WHO_AM_I` = `0xB4` | §9.5 | ✔ |
| `CTRL_REG1`: bit7=0, ODR`[6:3]`, AVG`[2:0]` | §9.6 | ✔ |
| ODR `0110` = 75 Hz | Tablo 17 | ✔ |
| AVG `010` = 16 | Tablo 18 | ✔ |
| `CTRL_REG2`: BOOT7, LFPF_CFG5, EN_LPFP4, **BDU3**, SWRESET2, ONESHOT0 | §9.7 | ✔ |
| `STATUS`: T_OR5, P_OR4, T_DA1, P_DA0 | §9.20 | ✔ (`0x33`) |
| 4096 LSB/hPa, 100 LSB/°C | §3.1 | ✔ |
| SPI maks. saat 10 MHz (`fc(SPC)`) | — | hwdef 10 MHz ✔ |

Datasheet §9.7 dipnotu — *"BDU'nun doğru çalışması için PRESS_OUT_H
(2Ah) en son okunan adres olmalıdır"* — mevcut `_update_pressure()`
zaten `0x28`'den 3 bayt okur, son adres `0x2A` olur. Ek değişiklik
gerekmemiştir.

> **`IF_CTRL (0x0E)` neden yazılmadı:** LPS22HB sürücüsü I2C'yi
> kapatmak için `CTRL_REG2 = 0x18` yazar. LPS22DF'te o bit orada
> değil, `IF_CTRL` içindedir — ve aynı register SIM (3-telli SPI)
> bitini de taşır. Yanlış bit haberleşmeyi tamamen keser. Kartta I2C
> hatları bağlı olmadığı için bu register'a hiç dokunulmamıştır.

### Değiştirilen Dosyalar

| Dosya | Değişiklik |
|---|---|
| `libraries/AP_Baro/AP_Baro_LPS2XH.h` | LPS22DF varyantı ve register tanımları |
| `libraries/AP_Baro/AP_Baro_LPS2XH.cpp` | `0xB4` WHO_AM_I kabulü, init ve okuma yolu |
| `libraries/AP_Baro/AP_Baro_Backend.h` | `DEVTYPE_BARO_LPS22DF = 0x19` |
| `Tools/scripts/decode_devid.py` | Yeni devtype'ın çözümlenmesi |

## 7.2 — PWM Kanal Haritası (Soğutma Fanı Çakışması)

**Belirti:** ArduPilot başlarken çöker:

```
LinuxPWM_Sysfs:Unable to open file /sys/class/pwm/pwmchip5/pwm0/duty_cycle:
    No such file or directory
```

**Teşhis:**

```bash
sudo bash -c "echo 0 > /sys/class/pwm/pwmchip5/export"
# bash: echo: write error: Device or resource busy

sudo cat /sys/kernel/debug/pwm
```

```
3: platform/23000000.pwm, 2 PWM devices
 pwm-0   (sysfs      ): requested period: 20000000 ns   <- ArduPilot kanal 0 ✔
 pwm-1   (sysfs      ): requested period: 20000000 ns   <- ArduPilot kanal 1 ✔
4: platform/23010000.pwm, 2 PWM devices
 pwm-0   ((null)     ): boş
 pwm-1   ((null)     ): boş
5: platform/23020000.pwm, 2 PWM devices
 pwm-0   (cooling_fan): requested enabled period: 40000 ns duty: 19608 ns   <- ÇAKIŞMA
 pwm-1   ((null)     ): boş
```

Eski eşleme kanal 2–3'ü `pwmchip5`'e veriyordu; oradaki `pwm-0` device
tree'de **soğutma fanına** ayrılmıştır. Kanal 0–1 açılır, init 3.
kanalda ölür.

**Düzeltme** — kanal 2–3 boştaki `pwmchip4`'e taşınmış, fana
dokunulmamıştır:

```cpp
// libraries/AP_HAL_Linux/RCOutput_Sysfs.cpp
#elif CONFIG_HAL_BOARD_SUBTYPE == HAL_BOARD_SUBTYPE_LINUX_T3_GEM_O1
        if (i == 0 || i == 1) {
            // pwmchip3/pwm0, pwmchip3/pwm1
            _pwm_channels[i] = NEW_NOTHROW PWM_Sysfs(_chip+3, i);
        } else if (i == 2 || i == 3) {
            // pwmchip4/pwm0, pwmchip4/pwm1. pwmchip5/pwm0 is claimed by the
            // cooling_fan node in the device tree, so it can't be exported here
            _pwm_channels[i] = NEW_NOTHROW PWM_Sysfs(_chip+4, i-2);
        } else {
            // pwmchip0/pwm0, pwmchip1/pwm0, pwmchip2/pwm0
            _pwm_channels[i] = NEW_NOTHROW PWM_Sysfs(_chip+(i-4), 0);
        }
```

## Düzeltmelerden Sonra Yeniden Derleme

Her iki düzeltme de kaynak koda girdikten sonra binary yeniden
derlenmelidir ([4. bölüme](04-wsl-ve-derleme-ortami.md) bakınız):

```bash
./waf configure --board=t3-gem-o1 --static
./waf copter
```

> Bu düzeltmeler
> [github.com/t3gemstone/ardupilot](https://github.com/t3gemstone/ardupilot)
> fork'una girmiş olabilir. Klonladığınız sürümde `0xB4` desteğinin
> bulunup bulunmadığını hızlıca kontrol edin:
>
> ```bash
> grep -n "0xB4\|LPS22DF" libraries/AP_Baro/AP_Baro_LPS2XH.cpp
> ```

---

Devam etmek için
[08-device-tree-overlay-pwm.md](08-device-tree-overlay-pwm.md).
