<p align="center">
    <picture>
        <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/t3gemstone/docs/main/logo/dark.png" width="40%" />
        <source media="(prefers-color-scheme: light)" srcset="https://raw.githubusercontent.com/t3gemstone/docs/main/logo/light.png" width="40%" />
        <img alt="T3 Foundation" src="https://raw.githubusercontent.com/t3gemstone/docs/main/logo/light.png" width="40%" />
    </picture>
</p>

# T3 Gemstone O1 — ArduPilot Kurulum ve Cross-Compile Rehberi

<p align="center">
  <a href="https://github.com/t3gemstone/ardupilot"><img alt="ArduPilot fork" src="https://img.shields.io/badge/T3_Gemstone-ardupilot_fork-black.svg"></a>
  <a href="https://docs.t3gemstone.org/tr/projects/ardupilot"><img alt="Resmi ArduPilot sayfası" src="https://img.shields.io/badge/T3_Gemstone-ardupilot_docs-green.svg"></a>
  <a href="https://docs.t3gemstone.org"><img alt="T3 Gemstone Docs" src="https://img.shields.io/badge/T3_Gemstone-docs-blue.svg"></a>
</p>

> T3 Gemstone O1 geliştirme kartının sıfırdan işletim sistemi imajı ile
> yazılmasını, kart üzerinde ArduPilot'un (ArduCopter) cross-compile
> yöntemiyle derlenmesini ve sensörleri okuyup MAVLink yayınlayan,
> açılışta otomatik başlayan çalışır bir kuruluma dönüştürülmesini
> anlatan adım adım bir rehberdir. İmaj yazma ve derleme akışı
> Windows 11 üzerinden yürütülmüştür; kart tarafındaki tüm komutlar
> gerçek donanımda çalıştırılarak doğrulanmıştır.

## Hızlı Başlangıç

- [1. Genel Bakış ve Gerekli Malzemeler](docs/01-genel-bakis-ve-gereksinimler.md)
- [Tüm dokümanlar](docs)

## Kapsam

- Gem Imager ile işletim sistemi imajının karta yazılması
- Kartın DFU moduna alınması ve boot mode DIP switch ayarları
- PuTTY üzerinden seri port ile ilk bağlantı
- WSL (Windows Subsystem for Linux) kurulumu ve ArduPilot kaynak kodunun hazırlanması
- ARM64 cross-compile toolchain kurulumu ve ArduCopter'ın derlenmesi
- GLIBC uyumsuzluğu, statik link çözümü ve binary'nin karta aktarılıp çalıştırılması
- Kartta fiilen takılı sensörlerin SPI üzerinden tespiti (ICM-20948 + **LPS22DF**)
- LPS22DF barometre desteği ve PWM kanal haritası için kaynak kodu düzeltmeleri
- 7 PWM kanalını 40-pin başlığa çıkaran device tree overlay yapılandırması
- MAVLink taşımaları (UDP yayın + TCP) ve iBUS/SBUS RC girişi
- systemd servisi, parametre varsayılanları ve yer istasyonu bağlantısı
- Sensör doğrulaması, kalibrasyon sırası ve uçuş öncesi bilinen riskler
- Sık karşılaşılan hataların özet çözüm tablosu

### Bu Kartta Yazılım Tanımlarından Ayrılan Noktalar

Aşağıdaki farklar T3 Gemstone O1 Obsidian üzerinde ölçülerek
saptanmıştır; her biri ilgili bölümde ele alınır:

| Konu | Yazılımın söylediği | Kartta olan |
|---|---|---|
| Barometre | Device tree `spi0.1`'i `bmp390-spidev` olarak tanıtır | ST **LPS22DF** (`WHO_AM_I = 0xB4`) |
| Baro sürücüsü | `AP_Baro_LPS2XH` yalnızca `0xB1` / `0xBD` kabul eder | `0xB4` desteği eklenmelidir |
| PWM kanal 2–3 | Stok eşleme `pwmchip5`'e yönlendirir | `pwmchip5/pwm0` soğutma fanına ayrılmıştır |
| PWM pinleri | Başlığa çıktığı varsayılır | Overlay kurulmadan çıkmaz |
| RC girişi | `SERIALn_PROTOCOL=23` yeterli sanılır | Linux HAL alıcı aygıtı açmaz; UART elle verilmelidir |

T3 Gemstone kartının donanımı ve genel yazılım ekosistemi hakkında
genel bilgi için T3'ün resmi dokümantasyonuna bakınız:
[docs.t3gemstone.org](https://docs.t3gemstone.org). T3'ün resmi
[ArduPilot sayfası](https://docs.t3gemstone.org/tr/projects/ardupilot)
ise APT paket deposu üzerinden kurulumu ve kalıcı (systemd) yapılandırmayı
anlatır; bu doküman ise APT paketinin henüz yayınlanmadığı durumlarda
izlenecek **kaynaktan cross-compile ile derleme** yoluna odaklanır.

## İçindekiler

- [1. Genel Bakış ve Gerekli Malzemeler](docs/01-genel-bakis-ve-gereksinimler.md)
- [2. İmajın Karta Yazılması (Gem Imager)](docs/02-imaj-yazma-gem-imager.md)
- [3. İlk Açılış ve SSH Bağlantısı (PuTTY)](docs/03-ilk-baglanti-ssh-putty.md)
- [4. WSL Kurulumu ve Derleme Ortamı](docs/04-wsl-ve-derleme-ortami.md)
- [5. GLIBC Uyumsuzluğu, Karta Aktarım ve Çalıştırma](docs/05-karta-aktarim-ve-calistirma.md)
- [6. Donanım Envanteri ve Sensörlerin Doğrulanması](docs/06-donanim-envanteri-ve-sensorler.md)
- [7. Gereken Kaynak Kodu Düzeltmeleri](docs/07-kaynak-kodu-duzeltmeleri.md)
- [8. Device Tree Overlay Kurulumu (PWM Pinleri)](docs/08-device-tree-overlay-pwm.md)
- [9. MAVLink Bağlantısı ve RC Girişi](docs/09-mavlink-ve-rc-girisi.md)
- [10. Kalıcı Kurulum, Servis ve Yer İstasyonu](docs/10-servis-ve-yer-istasyonu.md)
- [11. Doğrulama, Kalibrasyon ve Uçuş Öncesi Riskler](docs/11-dogrulama-ve-kalibrasyon.md)
- [12. Karşılaşılan Hatalar ve Çözümleri](docs/12-sorun-giderme.md)

## Nasıl Okunmalı?

Dokümanlar sırayla okunacak şekilde tasarlanmıştır; her dosyanın
sonunda bir sonraki bölüme yönlendiren bir bağlantı bulunur.

- **İlk kez kuranlar:** 1'den 11'e kadar sırayla okuyun. 1–5 imajı
  yazıp binary'yi derler; 6–11 kartı gerçekten uçuşa hazır hâle
  getirir.
- **Kartı zaten kurulu olanlar:** doğrudan 6. bölümden başlayabilir.
- **Bir hatayla karşılaşanlar:** [12. bölüme](docs/12-sorun-giderme.md)
  geçin.

> ### ⚠ Uçurmadan önce
>
> ESC bağlamadan ve uçuş denemesi yapmadan önce sensör
> oryantasyonunun ve PWM pinlerinin doğrulanması için
> [11.6 bölümünü](docs/11-dogrulama-ve-kalibrasyon.md) okuyun.

## Katkı

Bir hata fark ederseniz veya eksik bir konu eklemek isterseniz, ilgili
`docs/*.md` dosyasını düzenleyip bir pull request açabilirsiniz.

## Kaynakça

T3 Gemstone resmi site: https://www.t3gemstone.org/software  
T3 Gemstone dokümantasyon: https://docs.t3gemstone.org/tr/quickstart  
ArduPilot T3 Gemstone fork: https://github.com/t3gemstone/ardupilot
