# 1. Genel Bakış ve Gerekli Malzemeler

Bu doküman, T3 Gemstone O1 geliştirme kartının sıfırdan işletim
sistemi imajı ile yazılmasını ve kart üzerinde ArduPilot'un
(ArduCopter) çalıştırılmasını anlatır. Anlatılan tüm adımlar,
Windows 11 işletim sistemli bir bilgisayar üzerinden yürütülmüştür.

## Gerekli Donanım

- T3 Gemstone O1 geliştirme kartı
- USB Type-C kablosu (DFU modda karta bağlanmak ve seri konsol/ağ için)
- Ethernet kablosu (internet paylaşımı için, opsiyonel)
- Windows 11 çalışan bir bilgisayar

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

Bu altı adım sırasıyla bu dokümanın 2'den 5'e kadar olan
bölümlerinde ayrıntılandırılmıştır; 6. bölüm ise sık karşılaşılan
hataların ve çözümlerinin özet tablosunu içerir.

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
