# 12. Karşılaşılan Hatalar ve Çözümleri

Bu bölüm, kurulum ve derleme süreci boyunca karşılaşılabilecek yaygın
hataları, olası sebeplerini ve çözümlerini özetler. Bir şey beklendiği
gibi çalışmazsa önce buraya bakın.

## Kurulum ve Derleme Hataları

| Hata / Belirti | Olası Sebep | Çözüm |
|---|---|---|
| `E: Unable to locate package t3-gem-ardupilot` | APT deposunda ArduPilot paketi henüz yayınlanmamış (sadece BSP/kernel paketleri mevcut) | Kaynak koddan cross-compile ile derleme yoluna gidilir |
| Ethernet'te IP alınamıyor, ping "Network is unreachable" | Windows ICS paylaşımı yanlış adaptöre yönlendirilmiş veya hiç açılmamış | Wi-Fi → Özellikler → Paylaşım sekmesinden doğru Ethernet adaptörü seçilip paylaşım açılır |
| DHCPDISCOVER gönderiliyor ama yanıt yok | ICS servisi geçici olarak durmuş / arayüz resetlenmesi gerekiyor | Paylaşım kapatılıp açılır, kablo çıkarılıp takılır, `sudo dhclient eth0` tekrar denenir |
| WSL Ubuntu terminali açılır açılmaz kapanıyor | WSL2 bileşenleri güncel değil | PowerShell'de `wsl --update` çalıştırılır |
| `Read-only file system` / `Input/output error` | Windows `C:` sürücüsünde disk alanı tükenmiş | Disk Temizleme ile yer açılır, `wsl --shutdown` ile WSL yeniden başlatılır |
| `Could not find the program ['aarch64-linux-gnu-ar']` | Cross-compile toolchain kurulmamış | `sudo apt install gcc-aarch64-linux-gnu g++-aarch64-linux-gnu binutils-aarch64-linux-gnu -y` |
| `you need to install empy` / `pexpect` | Python bağımlılıkları eksik | `python3 -m pip install empy==3.3.4 pexpect --break-system-packages` |
| `GLIBC_2.43 not found` hatası ile binary çalışmıyor | WSL'deki GLIBC sürümü karttakinden daha yeni | `./waf configure --board=t3-gem-o1 --static` ile statik link kullanılır |
| `scp` ile WSL'den karta bağlantı kurulamıyor (Connection closed / timeout) | WSL2'nin izole NAT ağı host'un yerel ağına (kart) doğrudan erişemiyor | Binary önce Windows dosya sistemine (`/mnt/c/...`) kopyalanır, sonra Windows PowerShell üzerinden `scp` ile gönderilir |
| Build `dronecangen` veya `AP_Networking` aşamasında kırılıyor | `modules/DroneCAN` / `modules/lwip` submodule'leri indirilmemiş | `git submodule update --init --recursive modules/DroneCAN` ve aynısını `modules/lwip` için çalıştırın |

## Çalışma Zamanı Hataları

Bu tablo, binary karta kurulduktan sonra ortaya çıkan sorunları
kapsar.

| Hata / Belirti | Olası Sebep | Çözüm |
|---|---|---|
| `LinuxPWM_Sysfs:Unable to open file /sys/class/pwm/pwmchipN/pwm0/duty_cycle` | Overlay kurulmamış **veya** ilgili PWM çipi başka bir sürücüde (soğutma fanı) | Önce [8. bölüm](08-device-tree-overlay-pwm.md); sonra `sudo cat /sys/kernel/debug/pwm` ile `(cooling_fan)` etiketli kanalı kontrol edin |
| ArduPilot "no barometer" ile takılıyor | `AP_Baro_LPS2XH` sürücüsü `0xB4` WHO_AM_I'yi kabul etmiyor | [7.1 bölümündeki](07-kaynak-kodu-duzeltmeleri.md) düzeltmenin derlemeye girdiğini doğrulayın; `/dev/spidev0.1`'den `0xB4` gelmiyorsa sorun kablolamadadır |
| `PreArm: Motors: Check frame class and type` | `FRAME_CLASS` ayarlanmamış | `ardupilot.parm` içinde `FRAME_CLASS`/`FRAME_TYPE` verin, ArduPilot'u yeniden başlatın — [10. bölüm](10-servis-ve-yer-istasyonu.md) |
| `PreArm: RC not found` — alıcı kumandayı görüyor ama kanallar boş | `--serial2 /dev/ttyS3` veya `SERIAL2_PROTOCOL 23` eksik | İkisini de sağlayın — [9. bölüm](09-mavlink-ve-rc-girisi.md). Alıcının LED'inin sabit olması ArduPilot'un sinyali okuduğunu göstermez |
| Yer istasyonu UDP'de "portu seçiyorum ama bulamıyor" | Güvenlik duvarı, talep edilmemiş broadcast trafiğini düşürüyor | `sudo ufw allow 14550/udp` veya TCP'ye (`192.168.7.2:5760`) geçin |
| Bir kere bağlandı, sonra hiç heartbeat gelmiyor | `udpin` soketi eski istemci portuna kilitlenmiş | `udp:`/`bcast` ya da TCP kullanın; geçici çözüm ArduPilot'u yeniden başlatmaktır |
| TCP'de "connection refused" | ArduPilot çalışmıyor | `pgrep -a -f arducopter` ve `ss -ltn \| grep 5760` |
| Overlay değişikliğinden sonra kart açılmıyor / SSH yok | `/boot/uEnv.txt` bozulmuş | microSD'yi başka bilgisayara takıp yedeği geri koyun: `sudo cp /mnt/uEnv.txt.bak /mnt/uEnv.txt` |

> **Kart "açılmıyor" kararını erken vermeyin.** Normal açılış 30–60
> saniye sürer ve USB gadget arayüzü bu sırada host'ta kaybolup geri
> gelir. Bu çalışma sırasında tam olarak bu yanılgıya düşülmüştür;
> kart aslında 7 overlay ile sorunsuz açılıyordu.

### RC Girişini İki Adımda Ayırt Etme

`PreArm: RC not found` görüyorsanız hangi yarının eksik olduğunu şöyle
bulun:

```bash
# 1) UART ArduPilot'a veriliyor mu? Cikti "--serial2 /dev/ttyS3" icermeli
pgrep -a -f arducopter
```

```python
# 2) Port RCIN olarak etiketli mi?
m.mav.param_request_read_send(m.target_system, m.target_component, b'SERIAL2_PROTOCOL', -1)
print(m.recv_match(type='PARAM_VALUE', blocking=True).param_value)   # 23.0 olmali
```

İkisi de doğruysa sorun sinyal tarafındadır;
[9. bölümdeki](09-mavlink-ve-rc-girisi.md) ham bayt dinleme testiyle
alıcının gerçekten veri gönderip göndermediğini ayırt edin. Sık
sebepler: sinyal kablosu TX'e takılmış (RX olmalı), ortak GND yok,
alıcı beslemesi yetersiz.

## Ek Kaynaklar

Aşağıdaki durumlar için T3'ün resmi dokümantasyonuna bakınız:

- **Gem Imager açılmıyor / ".dll bulunamadı" hatası veriyor:** T3'ün
  resmi sorun giderme kılavuzu:
  [docs.t3gemstone.org/tr/troubleshoting](https://docs.t3gemstone.org/tr/troubleshoting).
- **ArduPilot çalışıyor ama uçuş davranışı, RC girişi veya telemetri
  ile ilgili bir sorun var:** T3'ün resmi ArduPilot sayfasındaki
  "Sorun Giderme" bölümü:
  [docs.t3gemstone.org/tr/projects/ardupilot](https://docs.t3gemstone.org/tr/projects/ardupilot).

---

Başa dönmek için [README.md](../README.md).
