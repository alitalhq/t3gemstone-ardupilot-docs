# 1. Genel Bakış ve Gerekli Malzemeler

Bu doküman, T3 Gemstone O1 geliştirme kartının sıfırdan işletim
sistemi imajı ile yazılmasını ve kart üzerinde ArduPilot'un
(ArduCopter) çalıştırılmasını anlatır. Anlatılan tüm adımlar,
Windows 11 işletim sistemli bir bilgisayar üzerinden yürütülmüştür.

## Gerekli Donanım

- T3 Gemstone O1 geliştirme kartı (model dizesi: `T3 Gemstone O1 Obsidian`)
- USB Type-C kablosu (DFU modda karta bağlanmak ve seri konsol/ağ için)
- Ethernet kablosu (internet paylaşımı için, opsiyonel)
- Windows 11 çalışan bir bilgisayar
- RC alıcı (iBUS/SBUS) — kumanda ile uçuş denemesi yapılacaksa

### Kartın Üzerindeki Sensörler

Bu rehberde kullanılan kartın doğrulanmış sensör dizilimi:

| Aygıt | Çip | Yol |
|---|---|---|
| IMU (ivme + jiro) | ICM-20948 | `/dev/spidev0.3` |
| Pusula | AK09916 (ICM-20948 içinde) | — |
| Barometre | LPS22DF | `/dev/spidev0.1` |
| Sıcaklık / nem | HDC2010 | I2C |
| CAN | m_can + TCAN1462-Q1 | `can0` |

Bu listenin nasıl doğrulandığı ve device tree'de yazandan nerede
ayrıldığı [6. bölümdedir](06-donanim-envanteri-ve-sensorler.md).

GPS:

...

## Gerekli Yazılımlar

- **Gem Imager** — T3 Gemstone'un resmi imaj yazma aracı
- **PuTTY** — SSH / seri port bağlantısı için
- **Git**, **WSL** (Windows Subsystem for Linux) — ArduPilot'u derlemek için

## Genel İş Akışı Özeti

1. Gem Imager indirilir ve kurulur.
2. Kart, BOOT switch'leri ayarlanarak ve buton yardımıyla DFU (Device
   Firmware Update) moduna alınır.
3. Gem Imager ile imaj karta yazılır.
4. Kart normal moda alınıp açılır, PuTTY ile SSH/seri bağlantı kurulur.
5. ArduPilot kaynak kodu indirilip WSL üzerinden cross-compile
   (çapraz derleme) ile derlenir.
6. Derlenen binary Gemstone'a aktarılıp çalıştırılır.
7. Kartta fiilen takılı sensörler tespit edilir; barometre ve PWM
   kanal haritası için kaynak kodu düzeltmeleri uygulanır.
8. PWM pinlerini başlığa çıkaran device tree overlay'leri kurulur.
9. MAVLink taşımaları ve RC girişi yapılandırılır, systemd servisi
   kurulur.
10. Sensörler doğrulanır, kalibrasyon yapılır ve uçuş öncesi riskler
    gözden geçirilir.

Bu adımlar 2'den 11'e kadar olan bölümlerde ayrıntılandırılmıştır;
12. bölüm ise sık karşılaşılan hataların ve çözümlerinin özet
tablosunu içerir.

> **Nerede biter, nerede başlar:** 5. bölümün sonunda binary kartta
> çalışır durumdadır ancak barometre bulunamaz, PWM kanalları açılmaz
> ve RC girişi görünmez. 6–11. bölümler bu üç engeli kaldırır.

## Bu Rehber ile T3'ün Resmi ArduPilot Sayfası Arasındaki Fark

T3'ün resmi dokümantasyonunda da bir
[ArduPilot sayfası](https://docs.t3gemstone.org/tr/projects/ardupilot)
bulunur; bu sayfa APT paket deposu üzerinden kurulumu, donanım
sensörlerini, systemd servis yönetimini ve GPIO pinout tablolarını
anlatır. Bu rehber ise, APT deposunda ArduPilot paketinin henüz
yayınlanmadığı durumlarda izlenecek olan **kaynaktan cross-compile ile
derleme** yolunu adım adım ele alır. Paket APT üzerinden
kurulabiliyorsa, doğrudan resmi sayfadaki yöntemi tercih etmeniz
yeterlidir.

---

Devam etmek için [02-imaj-yazma-gem-imager.md](02-imaj-yazma-gem-imager.md).
