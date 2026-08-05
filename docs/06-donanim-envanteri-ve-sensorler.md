# 6. Donanım Envanteri ve Sensörlerin Doğrulanması

Binary'nin karta aktarılıp `--help` çıktısı vermesi, ArduPilot'un
**uçuşa hazır** olduğu anlamına gelmez. Bu noktadan sonra kartın
üzerindeki sensörlerin gerçekten okunabilmesi gerekir.

Bu bölümden itibaren anlatılanlar, **T3 Gemstone O1 Obsidian** kartının
güncel donanımı üzerinde ölçülerek doğrulanmıştır. Kartın yazılım
tanımları (device tree ve ArduPilot'un stok sürücüleri) ile üzerinde
fiilen takılı olan çipler arasında aşağıdaki farklar vardır:

| Konu | Yazılımın söylediği | Güncel kartta olan | Sonuç |
|---|---|---|---|
| Barometre çipi | Device tree `spi0.1`'i `bmp390-spidev` olarak tanıtır | ST **LPS22DF** (`WHO_AM_I = 0xB4`) | Barometre bulunamaz; sürücüye çip desteği eklenmesi gerekir → [7. bölüm](07-kaynak-kodu-duzeltmeleri.md) |
| Barometre sürücüsü | `AP_Baro_LPS2XH` yalnızca LPS22HB (`0xB1`) ve LPS25HB (`0xBD`) kabul eder | LPS22DF (`0xB4`) | `_check_whoami()` reddeder |
| PWM kanal 2–3 | Stok eşleme `pwmchip5`'e yönlendirir | `pwmchip5/pwm0` device tree'de soğutma fanına ayrılmıştır | ArduPilot başlarken çöker → [7. bölüm](07-kaynak-kodu-duzeltmeleri.md) |
| PWM pinleri | 40-pin başlığa çıktığı varsayılır | Fabrika ayarında çıkmaz | Overlay kurulmalıdır → [8. bölüm](08-device-tree-overlay-pwm.md) |
| RC girişi | `SERIALn_PROTOCOL=23` yeterli sanılır | Linux HAL bu kart için hiçbir alıcı aygıtı açmaz | UART elle verilmelidir → [9. bölüm](09-mavlink-ve-rc-girisi.md) |

Aşağıdaki ölçümler bu farkların nasıl tespit edildiğini gösterir.

## Kartta Hangi Aygıtlar Var?

Karta SSH (veya PuTTY seri konsol) ile bağlanıp aşağıdaki komutlar
çalıştırılır:

```bash
ls /dev/spidev*
ls /dev/i2c*
for d in /sys/bus/iio/devices/*/name; do echo "$d: $(cat $d)"; done
cat /proc/device-tree/model; echo
uname -a
```

Doğrulanmış çıktı:

```
=== spidev ===
/dev/spidev0.0  /dev/spidev0.1  /dev/spidev0.2  /dev/spidev0.3
=== i2c ===
/dev/i2c-1 ... /dev/i2c-6
=== iio ===
/sys/bus/iio/devices/iio:device0/name: hdc2010
=== model ===
T3 Gemstone O1 Obsidian
=== uname ===
Linux gemstone 6.12.24-ti #1 SMP PREEMPT_RT ... aarch64 GNU/Linux
```

## SPI Aygıtlarının Kimlikleri

Device tree'nin hangi çipi iddia ettiği şöyle görülür:

```bash
for d in /sys/bus/spi/devices/*; do
  echo "$d -> modalias=$(cat $d/modalias) driver=$(basename $(readlink $d/driver))"
done
```

```
spi0.0 -> modalias=spi:dh2228fv          driver=spidev
spi0.1 -> modalias=spi:bmp390-spidev     driver=spidev     <-- iddia: BMP390
spi0.2 -> modalias=spi:dh2228fv          driver=spidev
spi0.3 -> modalias=spi:icm20948-spidev   driver=spidev
```

Device tree `spi0.1` için "BMP390" der. Güncel kartta o soketteki çip
BMP390 değil, ST LPS22DF'tir; modalias satırı donanımla uyuşmaz.
Aşağıdaki ölçüm bunu kanıtlar.

## WHO_AM_I ile Fiziksel Doğrulama

Her sensör çipinin, üreticinin sabitlediği bir kimlik register'ı
vardır. Kartın imajında ne `gcc` ne de Python `spidev` modülü kurulu
olduğu için SPI ioctl çağrısı saf Python ile elle kurulmuştur.
Aşağıdaki betik **salt okuma** yapar, hiçbir register'a yazmaz:

```python
#!/usr/bin/env python3
"""Read-only SPI WHO_AM_I / CHIP_ID probe using raw spidev ioctls."""
import array
import fcntl
import struct
import sys

SPI_IOC_WR_MODE = 0x40016B01
SPI_IOC_MESSAGE_1 = 0x40206B00


def xfer(fd, txbytes, speed_hz=1000000):
    tx = array.array('B', txbytes)
    rx = array.array('B', [0] * len(txbytes))
    tx_addr, _ = tx.buffer_info()
    rx_addr, _ = rx.buffer_info()
    msg = struct.pack('QQIIHBBBBBB', tx_addr, rx_addr, len(txbytes),
                      speed_hz, 0, 8, 0, 0, 0, 0, 0)
    fcntl.ioctl(fd, SPI_IOC_MESSAGE_1, msg)
    return list(rx)


def probe(path, mode, regs, extra=0):
    with open(path, 'rb+', buffering=0) as f:
        fcntl.ioctl(f, SPI_IOC_WR_MODE, struct.pack('B', mode))
        out = {}
        for reg in regs:
            n = 2 + extra
            rx = xfer(f, [reg | 0x80] + [0x00] * (n - 1))
            out[reg] = rx[1:]
        return out


if __name__ == '__main__':
    dev = sys.argv[1]
    mode = int(sys.argv[2]) if len(sys.argv) > 2 else 3
    for reg in (0x00, 0x0F, 0x01):
        vals = probe(dev, mode, [reg], extra=1)[reg]
        print('%s mode%d reg 0x%02X -> %s' %
              (dev, mode, reg, ' '.join('0x%02X' % v for v in vals)))
```

<details>
<summary>ioctl ayrıntıları (bu <code>struct.pack</code> biçimi neden böyle)</summary>

`struct spi_ioc_transfer` (linux/spi/spidev.h), aarch64 üzerinde
**32 bayt**:

| Alan | Tip | struct formatı |
|---|---|---|
| `tx_buf` | `__u64` | `Q` |
| `rx_buf` | `__u64` | `Q` |
| `len` | `__u32` | `I` |
| `speed_hz` | `__u32` | `I` |
| `delay_usecs` | `__u16` | `H` |
| `bits_per_word` | `__u8` | `B` |
| `cs_change` | `__u8` | `B` |
| `tx_nbits` | `__u8` | `B` |
| `rx_nbits` | `__u8` | `B` |
| `word_delay_usecs` | `__u8` | `B` |
| `pad` | `__u8` | `B` |

→ `'QQIIHBBBBBB'` = 8+8+4+4+2+6 = **32 bayt** ✔

ioctl numaraları (`SPI_IOC_MAGIC = 'k' = 0x6B`):

```
SPI_IOC_MESSAGE(1) = _IOW(0x6B, 0, char[32])
                   = (1 << 30) | (32 << 16) | (0x6B << 8) | 0  = 0x40206B00
SPI_IOC_WR_MODE    = _IOW(0x6B, 1, __u8)
                   = (1 << 30) | (1 << 16)  | (0x6B << 8) | 1  = 0x40016B01
```

Her iki çip de register okumasını MSB=1 ile yapar → `tx[0] = reg | 0x80`.
</details>

Betik karta kopyalanıp çalıştırılır:

```bash
python3 /tmp/spi_probe.py /dev/spidev0.1 3
python3 /tmp/spi_probe.py /dev/spidev0.1 0
python3 /tmp/spi_probe.py /dev/spidev0.3 3
```

**Belirleyici çıktı:**

```
/dev/spidev0.1 mode3 reg 0x00 -> 0x00 0x00
/dev/spidev0.1 mode3 reg 0x0F -> 0xB4 0x00      <-- LPS22DF WHO_AM_I
/dev/spidev0.1 mode0 reg 0x0F -> 0xB4 0x00      <-- mode0'da da aynı, kararlı
/dev/spidev0.3 mode3 reg 0x00 -> 0xEA 0x00      <-- ICM-20948 WHO_AM_I
```

| Gözlem | Çıkarım |
|---|---|
| `spidev0.3` reg `0x00` = `0xEA` | ICM-20948 doğrulandı |
| `spidev0.1` reg `0x0F` = `0xB4` | ST **LPS22DF**. LPS22HB `0xB1`, LPS25H `0xBD` olurdu |
| `spidev0.1` reg `0x00` = `0x00` | BMP390 **değil** — BMP3 ailesi burada `0x60` döndürür |
| mode0 ve mode3 aynı sonucu veriyor | Okuma gürültü değil, gerçek |

İki bağımsız kanıt (pozitif `0xB4` + negatif `0x00`) barometrenin
BMP390 değil **LPS22DF** olduğunu kesinleştirir.

## Kartın Gerçek Sensör Dizilimi

| Aygıt | Çip | Yol | Not |
|---|---|---|---|
| IMU (ivme + jiro) | ICM-20948 | `/dev/spidev0.3` | 9-DOF, içinde pusula da var |
| Pusula | AK09916 | ICM-20948 içinden | Ayrı SPI aygıtı değil |
| Barometre | **LPS22DF** | `/dev/spidev0.1` | Device tree `bmp390-spidev` der, çip LPS22DF'tir |
| Sıcaklık / nem | HDC2010 | I2C, iio | ArduPilot kullanmıyor |
| CAN | m_can + TCAN1462-Q1 | `can0` | bkz. [11. bölüm](11-dogrulama-ve-kalibrasyon.md) |

## `hwdef.dat` — Kartın ArduPilot'a Tanıtıldığı Dosya

Kart ArduPilot'ta zaten tanımlıdır. İlgili dosyalar:

| Dosya | İçerik |
|---|---|
| `libraries/AP_HAL_Linux/hwdef/t3-gem-o1/hwdef.dat` | Kart tanımı |
| `libraries/AP_HAL/AP_HAL_Boards.h` | `HAL_BOARD_SUBTYPE_LINUX_T3_GEM_O1 1029` |
| `libraries/AP_HAL/board/linux.h` | `HAL_LINUX_GPIO_T3_GEM_O1_ENABLED` varsayılanı |
| `libraries/AP_HAL_Linux/GPIO_T3_GEM_O1.{h,cpp}` | GPIO pin haritası (LED yeşil 380, kırmızı 381) |

`hwdef.dat` içindeki kritik satırlar:

```
IMU     Invensensev2 SPI:icm20948 ROTATION_ROLL_180_YAW_90
COMPASS AK09916:probe_ICM20948 0 ROTATION_ROLL_180
BARO    LPS2XH SPI:lps22df

LINUX_SPIDEV "icm20948" 0   3      SPI_MODE_3 8   SPI_CS_KERNEL 4*MHZ  8*MHZ
LINUX_SPIDEV "lps22df"  0   1      SPI_MODE_3 8   SPI_CS_KERNEL 10*MHZ 10*MHZ
#                       ^bus ^subdev
```

`LINUX_SPIDEV` satırının okunuşu:

```
LINUX_SPIDEV "<isim>" <bus> <cs> <mod> <bit> <cs-tipi> <min-hız> <maks-hız>
                        0     3   →  /dev/spidev0.3
```

`ROTATION_*` değerleri çipin karta hangi yönde lehimlendiğini söyler —
yanlışsa araç havada ters tepki verir. Bu değerlerin kartta
doğrulanması için bkz. [11.6](11-dogrulama-ve-kalibrasyon.md).

### hwdef → C++ Kod Üretimi

Derleme sırasında
`libraries/AP_HAL_Linux/hwdef/scripts/linux_hwdef.py` bu dosyayı okuyup
C++ makroları üretir:

```c
#define HAL_SPI_DEVICE1 SPIDesc("icm20948", 0, 3, SPI_MODE_3, 8, SPI_CS_KERNEL, 4*MHZ, 8*MHZ)
#define HAL_SPI_DEVICE2 SPIDesc("lps22df" , 0, 1, SPI_MODE_3, 8, SPI_CS_KERNEL, 10*MHZ, 10*MHZ)
#define HAL_BARO_PROBE1 probe_spi_dev(AP_Baro_LPS2XH::probe, "lps22df"); RETURN_IF_NO_SPACE;
#define AP_BARO_LPS2XH_ENABLED 1
#define HAL_BARO_PROBE_LIST HAL_BARO_PROBE1
```

Yani `BARO LPS2XH SPI:lps22df` satırı `AP_Baro_LPS2XH::probe()`
çağrısına dönüşür. Sorun şudur ki o sürücü LPS22DF'i tanımaz. Bir
sonraki bölüm bunu düzeltir.

---

Devam etmek için
[07-kaynak-kodu-duzeltmeleri.md](07-kaynak-kodu-duzeltmeleri.md).
