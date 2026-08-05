# 8. Device Tree Overlay Kurulumu (PWM Pinleri)

Kod tarafındaki düzeltmeler yapıldıktan sonra bile, kartın fabrika
ayarında **PWM pinleri 40-pinli başlığa çıkmaz**. Overlay kurulmadan
`/sys/class/pwm` altında ArduPilot'un beklediği çipler görünmez ve
motor çıkışı üretilemez.

## Overlay Nedir, Nasıl Uygulanır?

Device tree, Linux'ta donanımın tarif edildiği yapıdır. Ana `.dtb`
dosyası kartın temel donanımını tanımlar; `.dtbo` **overlay**'leri
bunun üzerine yama gibi eklenir. Bir pinin PWM çıkışı mı yoksa normal
GPIO mu olacağı burada belirlenir.

Kart U-Boot ile açılır. `/boot/uEnv.txt` içindeki `overlays=` satırı,
`/boot/overlays/` altındaki `.dtbo` dosyalarını sırayla ana device
tree'ye uygular:

```
get_overlays=for o in ${overlays}; do
               load mmc ${bootpart} ${fdtoverlayaddr} overlays/${o};
               fdt apply ${fdtoverlayaddr};
             done
```

## Mevcut Overlay Dosyaları

```bash
ls /boot/overlays/ | grep pwm
```

```
k3-am67a-t3-gem-o1-pwm-ecap0-gpio12.dtbo
k3-am67a-t3-gem-o1-pwm-ecap1-gpio16.dtbo
k3-am67a-t3-gem-o1-pwm-ecap2-gpio18.dtbo
k3-am67a-t3-gem-o1-pwm-epwm0-gpio5.dtbo
k3-am67a-t3-gem-o1-pwm-epwm0-gpio14.dtbo
k3-am67a-t3-gem-o1-pwm-epwm0-gpio5-gpio14.dtbo     <-- ikisini birden açan
k3-am67a-t3-gem-o1-pwm-epwm1-gpio6.dtbo
k3-am67a-t3-gem-o1-pwm-epwm1-gpio13.dtbo
k3-am67a-t3-gem-o1-pwm-epwm1-gpio6-gpio13.dtbo     <-- ikisini birden açan
```

`epwm0` / `epwm1` çipleri iki kanal taşır; `-gpio5-gpio14` gibi
birleşik overlay'ler ikisini birden açar. Tek tek olanlarla birleşik
olanı **aynı anda kullanmayın** — aynı düğümü iki kez yamalamış
olursunuz.

## Kurulum

> ### ⚠ Önce yedek alın
>
> `/boot/uEnv.txt` bozulursa kart açılmaz ve düzeltmek için microSD
> kartı çıkarıp başka bir bilgisayara takmanız gerekir.
>
> ```bash
> sudo cp /boot/uEnv.txt /boot/uEnv.txt.bak
> ```

`overlays=` satırını 7 PWM kanalını da açacak şekilde güncelleyin —
**tek satır**, boşlukla ayrılmış:

```
overlays=k3-am67a-t3-gem-o1-i2c1-400000.dtbo k3-am67a-t3-gem-o1-spidev0-2cs.dtbo k3-am67a-t3-gem-o1-pwm-epwm0-gpio5-gpio14.dtbo k3-am67a-t3-gem-o1-pwm-epwm1-gpio6-gpio13.dtbo k3-am67a-t3-gem-o1-pwm-ecap0-gpio12.dtbo k3-am67a-t3-gem-o1-pwm-ecap1-gpio16.dtbo k3-am67a-t3-gem-o1-pwm-ecap2-gpio18.dtbo
```

> İlk iki overlay (`i2c1-400000`, `spidev0-2cs`) zaten mevcuttur ve
> **korunmalıdır** — `spidev0-2cs`, sensörlerin bağlı olduğu SPI
> aygıtlarını oluşturur. Silinirse IMU ve barometre tamamen kaybolur.

Kartı yeniden başlatın:

```bash
sudo reboot
```

> Kart ~30–60 saniyede geri döner. USB gadget arayüzü bilgisayarda
> kaybolup yeniden belirir; bu normaldir. `ping` ile bekleyin, bu süre
> dolmadan "kart açılmıyor" kararı vermeyin.

## Doğrulama

```bash
ls /proc/device-tree/chosen/overlays/
```

Yedi overlay ve pin takma adları görünmelidir:

```
hat-08.23000000.pwm   hat-12.23120000.pwm   hat-29.23000000.pwm
hat-31.23010000.pwm   hat-32.23100000.pwm   hat-33.23010000.pwm
hat-36.23110000.pwm
k3-am67a-t3-gem-o1-i2c1-400000.kernel
k3-am67a-t3-gem-o1-pwm-ecap0-gpio12.kernel
k3-am67a-t3-gem-o1-pwm-ecap1-gpio16.kernel
k3-am67a-t3-gem-o1-pwm-ecap2-gpio18.kernel
k3-am67a-t3-gem-o1-pwm-epwm0-gpio5-gpio14.kernel
k3-am67a-t3-gem-o1-pwm-epwm1-gpio6-gpio13.kernel
k3-am67a-t3-gem-o1-spidev0-2cs.kernel
```

## `pwmchip` Numaralarını Sabit Varsaymayın

`pwmchipN` numaraları çekirdeğin kayıt sırasına göre atanır ve
değişebilir. Doğrusunu **donanım adresinden** teyit edin:

```bash
for c in /sys/class/pwm/pwmchip*; do
  echo "$(basename $c) -> $(basename $(readlink -f $c/device))  npwm=$(cat $c/npwm)"
done
```

```
pwmchip0 -> 23100000.pwm  npwm=1
pwmchip1 -> 23110000.pwm  npwm=1
pwmchip2 -> 23120000.pwm  npwm=1
pwmchip3 -> 23000000.pwm  npwm=2
pwmchip4 -> 23010000.pwm  npwm=2
pwmchip5 -> 23020000.pwm  npwm=2     <-- cooling_fan, dokunulmuyor
```

## Kanal → Pin Haritası

| ArduPilot kanalı | pwmchip/pwm | Donanım | Overlay | 40-pin başlık |
|---|---|---|---|---|
| 0 | pwmchip3/pwm0 | 23000000 epwm0 | `epwm0-gpio5-gpio14` | **pin 29** |
| 1 | pwmchip3/pwm1 | 23000000 epwm0 | `epwm0-gpio5-gpio14` | **pin 08** |
| 2 | pwmchip4/pwm0 | 23010000 epwm1 | `epwm1-gpio6-gpio13` | **pin 31** |
| 3 | pwmchip4/pwm1 | 23010000 epwm1 | `epwm1-gpio6-gpio13` | **pin 33** |
| 4 | pwmchip0/pwm0 | 23100000 ecap0 | `ecap0-gpio12` | **pin 32** |
| 5 | pwmchip1/pwm0 | 23110000 ecap1 | `ecap1-gpio16` | **pin 36** |
| 6 | pwmchip2/pwm0 | 23120000 ecap2 | `ecap2-gpio18` | **pin 12** |
| — | pwmchip5/pwm0 | 23020000 epwm2 | (yok) | soğutma fanı — kullanılmıyor |

> **Yan etki:** `epwm0-gpio5-gpio14` overlay'i `main_uart1`'i RX-only
> hâle getirir. O UART'ı seri telemetri için kullanmayı planlıyorsanız
> bunu hesaba katın. RC girişi (iBUS/SBUS) tek yönlü olduğu için bu
> durum sorun çıkarmaz — bkz. [9. bölüm](09-mavlink-ve-rc-girisi.md).

---

Devam etmek için
[09-mavlink-ve-rc-girisi.md](09-mavlink-ve-rc-girisi.md).
