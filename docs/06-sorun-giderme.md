# 6. Karşılaşılan Hatalar ve Çözümleri

Bu tablo, kurulum ve derleme süreci boyunca karşılaşılabilecek yaygın
hataları, olası sebeplerini ve çözümlerini özetler. Bir şey beklendiği
gibi çalışmazsa önce buraya bakın.

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

## Ek Kaynaklar

Bu tablo, yalnızca cross-compile kurulum sürecinde karşılaşılan
hataları kapsar. Aşağıdaki durumlar için T3'ün resmi dokümantasyonuna
bakınız:

- **Gem Imager açılmıyor / ".dll bulunamadı" hatası veriyor:** T3'ün
  resmi sorun giderme kılavuzu:
  [docs.t3gemstone.org/tr/troubleshoting](https://docs.t3gemstone.org/tr/troubleshoting).
- **ArduPilot çalışıyor ama uçuş davranışı, RC girişi veya telemetri
  ile ilgili bir sorun var:** T3'ün resmi ArduPilot sayfasındaki
  "Sorun Giderme" bölümü:
  [docs.t3gemstone.org/tr/projects/ardupilot](https://docs.t3gemstone.org/tr/projects/ardupilot).

---

Başa dönmek için [README.md](../README.md).
