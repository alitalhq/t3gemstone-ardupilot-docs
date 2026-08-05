# 5. GLIBC Uyumsuzluğu, Karta Aktarım ve Çalıştırma

Bir önceki bölümde ArduCopter binary'sini başarıyla derledik. Bu
bölümde, derlenen binary'nin kartta neden çalışmayabileceğini, buna
nasıl çözüm bulunacağını ve binary'nin karta aktarılıp çalıştırılma
adımlarını ele alıyoruz.

## GLIBC Uyumsuzluğu ve Statik Link Çözümü

WSL üzerindeki Ubuntu sürümü ile kart üzerindeki Ubuntu sürümünün
GLIBC (GNU C Kütüphanesi) sürümleri farklı olabilir. Bu durumda
derlenen binary kartta çalıştırıldığında aşağıdaki gibi bir hata
alınır:

```
/opt/ardupilot/bin/arducopter: /lib/aarch64-linux-gnu/libm.so.6: version `GLIBC_2.43' not found
```

<p align="center">
  <img src="images/05-karta-aktarim/01-glibc-hatasi.png" width="480" alt="GLIBC uyumsuzluğu hatası">
  <br>
  <em>GLIBC uyumsuzluğu hatası</em>
</p>

### Sürümlerin Kontrolü

Kartta ve WSL'de aşağıdaki komutla GLIBC sürümü görülebilir:

```bash
ldd --version
```

Derleme ortamındaki (WSL) GLIBC sürümü, kartınkinden daha yeni ise,
dinamik olarak derlenen binary kartta çalışmaz. Doğrulanmış durum:

| Ortam | GLIBC |
|---|---|
| Kart (Ubuntu 24.04) | **2.39** — `ldd (Ubuntu GLIBC 2.39-0ubuntu8) 2.39` |
| Derleme host'u | 2.43 |

### Çözüm: Statik Link (`--static`)

ArduPilot'un waf build sistemi, tüm bağımlılıkları (GLIBC dahil)
binary'nin içine gömen statik link seçeneğini desteklemektedir. Bu
sayede binary, hedef sistemin GLIBC sürümünden bağımsız olarak
çalışabilir hale gelir. Aşağıdaki komutlar önceki derleme çıktısını
temizler ve `--static` bayrağıyla yeniden derler:

```bash
./waf distclean
./waf configure --board=t3-gem-o1 --static
./waf copter
```

Statik link ile derlenen binary'nin dosya boyutu, dinamik link ile
derlenen binary'ye göre daha büyük olur; ölçülen değer **5.0 MB →
14.0 MB**'dır ve bu beklenen bir durumdur.

Sonucu doğrulayın:

```bash
file build/t3-gem-o1/bin/arducopter
# ELF 64-bit LSB executable, ARM aarch64, statically linked
```

> **Not:** Alternatif bir çözüm olarak, WSL üzerinde kartla birebir
> aynı Ubuntu sürümünü kullanan bir Docker container'ı içinde derleme
> yapmak da mümkündür. Ancak statik link yöntemi, ek bir ortam
> kurmadan hızlı bir çözüm sağlamaktadır.

## Binary'nin Karta Aktarılması ve Çalıştırılması

### Kartın Güncel IP Adresinin Bulunması

Kartta PuTTY üzerinden aşağıdaki komut çalıştırılarak kartın ağ
üzerindeki IP adresi öğrenilir:

```bash
hostname -I
```

### WSL'den Karta Doğrudan Aktarım Sınırlaması

> **Dikkat:** WSL2, kendi izole sanal ağında (NAT arkasında)
> çalıştığından, WSL içinden host bilgisayarın yerel ağındaki diğer
> cihazlara (kart gibi) doğrudan erişim genellikle mümkün olmaz. Bu
> nedenle WSL'den doğrudan `scp` ile karta dosya göndermek çoğunlukla
> başarısız olur (bağlantı zaman aşımına uğrar).

### Çözüm: Windows Üzerinden Aktarım

Derlenen binary önce WSL'den Windows dosya sistemine kopyalanır:

```bash
cp ~/ardupilot/build/t3-gem-o1/bin/arducopter /mnt/c/Users/<kullanici>/Desktop/arducopter
```

<p align="center">
  <img src="images/05-karta-aktarim/02-binary-windows-dosya-sistemi.png" width="260" alt="Binary Windows dosya sisteminde">
  <br>
  <em>Binary, Windows dosya sisteminde (arducopter dosyası)</em>
</p>

Ardından Windows PowerShell'den, OpenSSH istemcisi kullanılarak karta
gönderilir:

```powershell
scp C:\Users\<kullanici>\Desktop\arducopter gemstone@<KART_IP>:~/arducopter
```

<p align="center">
  <img src="images/05-karta-aktarim/03-scp-aktarim.png" width="520" alt="PowerShell üzerinden scp ile aktarım">
  <br>
  <em>PowerShell üzerinden scp ile aktarım</em>
</p>

Kartın `gemstone` kullanıcısının şifresi girilerek transfer
tamamlanır.

### Binary'nin Karta Kurulması

Kartta PuTTY üzerinden aşağıdaki komutlar sırayla çalıştırılır. Bu
komutlar binary'yi çalıştırılabilir hale getirir, gerekli klasör
yapısını (log, terrain, storage, parametre dosyası) oluşturur:

```bash
sudo mkdir -p /opt/ardupilot/bin
sudo cp ~/arducopter /opt/ardupilot/bin/arducopter
sudo chmod +x /opt/ardupilot/bin/arducopter
sudo mkdir -p /opt/ardupilot/var/log
sudo mkdir -p /opt/ardupilot/var/terrain
sudo mkdir -p /opt/ardupilot/var/storage
sudo mkdir -p /opt/ardupilot/etc
sudo touch /opt/ardupilot/etc/ardupilot.parm
```

### Çalıştırma Testi

Binary'nin doğru çalıştığını doğrulamak için yardım menüsü çağrılır:

```bash
sudo /opt/ardupilot/bin/arducopter --help
```

Yardım menüsünün hatasız görüntülenmesi, binary'nin karta doğru
şekilde derlenip aktarıldığını doğrular.

<p align="center">
  <img src="images/05-karta-aktarim/04-arducopter-help-ciktisi.png" width="480" alt="arducopter --help çıktısı">
  <br>
  <em>arducopter --help çıktısı</em>
</p>

Gerçek bir başlatma denemesi için:

```bash
sudo /opt/ardupilot/bin/arducopter \
  --serial0 udp:127.0.0.1:14550 \
  --log-directory /opt/ardupilot/var/log \
  --terrain-directory /opt/ardupilot/var/terrain \
  --storage-directory /opt/ardupilot/var/storage
```

> Bu komut yalnızca binary'nin ayağa kalktığını görmek içindir; bu
> aşamada barometre bulunamayacak ve PWM kanalları açılmayacaktır.
> Yer istasyonuna gerçekten bağlanan yapılandırma
> [9.](09-mavlink-ve-rc-girisi.md) ve [10. bölümdedir](10-servis-ve-yer-istasyonu.md).
> Dizin yolları `hwdef.dat` içinde zaten sabitlenmiş olduğu için
> `--log-directory` gibi bayraklar aslında gerekli değildir.

## Sıradaki Adım: Donanımın Çalışır Hâle Getirilmesi

Buradaki adımlar, derlenen binary'nin karta doğru şekilde
aktarıldığını doğrulamak içindir. `--help` çıktısının görünmesi
ArduPilot'un **uçuşa hazır** olduğu anlamına gelmez: bu noktada
barometre henüz bulunamaz, PWM kanalları açılmaz ve RC girişi
görünmez.

Bundan sonraki bölümler bu engelleri sırayla kaldırır:

| Bölüm | Konu |
|---|---|
| [6](06-donanim-envanteri-ve-sensorler.md) | Kartta fiilen takılı sensörlerin tespiti ve `hwdef.dat` |
| [7](07-kaynak-kodu-duzeltmeleri.md) | LPS22DF barometre desteği ve PWM kanal haritası düzeltmesi |
| [8](08-device-tree-overlay-pwm.md) | PWM pinlerini başlığa çıkaran device tree overlay'leri |
| [9](09-mavlink-ve-rc-girisi.md) | MAVLink taşımaları ve RC (iBUS/SBUS) girişi |
| [10](10-servis-ve-yer-istasyonu.md) | systemd servisi, parametreler, yer istasyonu |
| [11](11-dogrulama-ve-kalibrasyon.md) | Doğrulama, kalibrasyon ve uçuş öncesi riskler |

> APT deposundan kurulum ve T3'ün resmi yapılandırma önerileri için:
> [docs.t3gemstone.org/tr/projects/ardupilot](https://docs.t3gemstone.org/tr/projects/ardupilot).

---

Devam etmek için
[06-donanim-envanteri-ve-sensorler.md](06-donanim-envanteri-ve-sensorler.md).
