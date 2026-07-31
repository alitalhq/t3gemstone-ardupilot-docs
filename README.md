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
> yazılmasını ve kart üzerinde ArduPilot'un (ArduCopter) cross-compile
> yöntemiyle derlenip çalıştırılmasını anlatan adım adım bir kurulum
> rehberidir. Windows 11 üzerinden yürütülen bir kurulum akışı esas
> alınmıştır.

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
- Sık karşılaşılan hataların özet çözüm tablosu

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
- [6. Karşılaşılan Hatalar ve Çözümleri](docs/06-sorun-giderme.md)

## Nasıl Okunmalı?

Dokümanlar sırayla okunacak şekilde tasarlanmıştır; her dosyanın
sonunda bir sonraki bölüme yönlendiren bir bağlantı bulunur. Kurulumu
ilk kez yapacaklar için 1'den 5'e kadar sırayla okumak, bir hatayla
karşılaşanlar için ise doğrudan
[6. bölüme](docs/06-sorun-giderme.md) geçmek önerilir.

## Katkı

Bir hata fark ederseniz veya eksik bir konu eklemek isterseniz, ilgili
`docs/*.md` dosyasını düzenleyip bir pull request açabilirsiniz.

## Kaynakça

T3 Gemstone resmi site: https://www.t3gemstone.org/software  
T3 Gemstone dokümantasyon: https://docs.t3gemstone.org/tr/quickstart  
ArduPilot T3 Gemstone fork: https://github.com/t3gemstone/ardupilot
