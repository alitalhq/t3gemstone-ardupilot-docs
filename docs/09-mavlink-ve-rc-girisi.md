# 9. MAVLink Bağlantısı ve RC Girişi

Bu bölüm, ArduPilot'un yer istasyonuyla nasıl haberleştiğini ve bu
kartta RC alıcısının nasıl tanıtıldığını anlatır. İkisi de bu kartta
varsayılan davranıştan ayrılır; sırasıyla ele alınmıştır.

## Ağ Adresleri

Kart USB-C ile bağlandığında bilgisayarda bir ethernet arayüzü belirir:

| | Adres |
|---|---|
| Kart (`usb0`) | `192.168.7.2` — **sabit** |
| Host PC | DHCP ile alınır (örn. `192.168.7.59`) — **dinamik, değişebilir** |
| Yayın (broadcast) | `192.168.7.255` |

> **Sık yapılan hata:** Host adresinin `192.168.7.1` olduğunu
> varsaymak. Host adresi DHCP ile atanır ve değişir. Bu yüzden karta
> host IP'si gömen bir yapılandırma kırılgandır.

## Taşıma Parametreyle Değil, Komut Satırıyla Seçilir

Bu, ArduPilot'a alışkın olanların bile takıldığı noktadır. Linux HAL'de
her `SERIAL<n>` portuna bir "cihaz yolu" verilir; bu yol bir seri port
da olabilir, TCP/UDP soketi de.

`libraries/AP_HAL_Linux/HAL_Linux_Class.cpp` içindeki kullanım örnekleri:

```
--serial0 /dev/ttyO4
--serial0 tcp:11.0.0.2:5678
--serial0 udp:11.0.0.2:14550
--serial0 udp:11.0.0.255:14550:bcast
--serial0 udpin:0.0.0.0:14550
```

Kabul edilen biçim `<protokol>:<ip>:<port>[:<bayrak>]`
(ayrıştırma: `UARTDriver.cpp`):

| Biçim | Davranış |
|---|---|
| `/dev/ttyX` | Karakter aygıtıysa doğrudan seri port |
| `udp:IP:PORT` | Kart **IP'ye gönderir** (push). Yer istasyonu sadece dinler. |
| `udp:IP:PORT:bcast` | Yayın adresine gönderir |
| `udpin:IP:PORT` | Kart **dinler** (bind). Yer istasyonu bağlantıyı başlatır. |
| `tcp:IP:PORT[:wait]` | TCP sunucu; `wait` bağlantı gelene kadar bekletir |

Kaynak koddan iki ayrıntı:

- `udpin` ile `bcast` birlikte kullanılamaz →
  `AP_HAL::panic("Can't combine udpin with bcast")`
- UDP'de `_packetise = true` olur — MAVLink mesajları datagram
  sınırlarına hizalanır, bir pakette yarım mesaj gitmez.

`SERIAL0_PROTOCOL` **taşımayı değil**, o taşıma üzerinde konuşulan
protokolü seçer. Varsayılanı zaten MAVLink2'dir
(`AP_SerialManager.cpp`):

```c
#ifndef DEFAULT_SERIAL0_PROTOCOL
#define DEFAULT_SERIAL0_PROTOCOL SerialProtocol_MAVLink2
#endif
```

<details>
<summary>Eski <code>-A/-B/-C</code> bayraklarının karşılıkları</summary>

| Eski bayrak | Yeni karşılığı |
|---|---|
| `-A` / `--uartA` | SERIAL0 |
| `-C` / `--uartC` | SERIAL1 |
| `-D` / `--uartD` | SERIAL2 |
| `-B` / `--uartB` | SERIAL3 |
| `-E` … `-J` | SERIAL4 … SERIAL9 |

`-B`'nin SERIAL3'e gitmesi tarihsel bir kalıntıdır. Yeni `--serial<n>`
biçimini kullanın.
</details>

## Seçilen Yapılandırma: UDP Yayın + TCP Aynı Anda

Her `--serial<n>` bağımsız bir taşıma alır, dolayısıyla ikisi birden
açık olabilir:

```bash
/opt/ardupilot/bin/arducopter \
    --serial0 udp:192.168.7.255:14550:bcast \
    --serial1 tcp:0.0.0.0:5760
```

**Neden bu ikili:**

| Taşıma | Avantaj | Bedel |
|---|---|---|
| UDP bcast (SERIAL0) | Yer istasyonu sadece portu bilir, IP gerekmez; kartı otomatik bulur | Host'ta güvenlik duvarı kuralı gerekir |
| TCP (SERIAL1) | Bağlantıyı yer istasyonu başlatır → güvenlik duvarı sorunsuz; kopup yeniden bağlanmak serbest | IP yazmak gerekir |

Yayın adresi (`192.168.7.255`) kullanıldığı için **host IP'si koda
gömülmez** — host DHCP ile adres değiştirse bile telemetri kesilmez.

### Kaçınılması Gereken Yapılandırmalar

| Yapılandırma | Sorun |
|---|---|
| `udp:192.168.7.59:14550` | Host IP'si dinamiktir. Adres değişince telemetri **sessizce** kesilir. |
| `udpin:0.0.0.0:14550` | Soket **ilk paketi gönderen istemciye kilitlenir** — aşağıya bakın |

> **`udpin` kilitlenmesi (testte yaşandı):** Yer istasyonu kapanıp
> farklı bir kaynak porttan yeniden bağlanırsa, kart hâlâ eski (artık
> kapalı) porta göndermeye devam eder. Testte ilk oturum sorunsuz
> bağlandı; oturum bitip ikincisi başlayınca 45 saniye boyunca hiç
> heartbeat gelmedi. Çözüm ArduPilot'u yeniden başlatmak oldu.
>
> Yer istasyonunu sık açıp kapatacaksanız `udp:`/`bcast` veya TCP
> kullanın. Tek ve kalıcı bir bağlantı için `udpin` sorunsuzdur.

### Doğrulama

```bash
ss -ltn | grep 5760
# LISTEN 0  0  0.0.0.0:5760  0.0.0.0:*
```

> `udp:` (push/bcast) modunda dinleyen bir soket **görünmez** — kart
> giden yönde geçici bir kaynak portu kullanır. `ss -lun | grep 14550`
> çıktısının boş olması normaldir.

## RC Girişi (iBUS / SBUS) — SERIAL2

> ### ⚠ Bu kartta RC girişi kendiliğinden çalışmaz
>
> Linux HAL, T3 Gemstone O1 için **hiçbir alıcı aygıtı açmaz.**
> `HAL_Linux_Class.cpp`:
>
> ```cpp
> #elif ... CONFIG_HAL_BOARD_SUBTYPE == HAL_BOARD_SUBTYPE_LINUX_T3_GEM_O1
> // this is needed to allow for RC input using SERIALn_PROTOCOL=23. No fd is opened
> // in the linux driver and instead user needs to provide a uart via SERIALn_PROTOCOL
> static RCInput_RCProtocol rcinDriver{nullptr, nullptr};
> ```
>
> `nullptr, nullptr` — yani UART'ı **siz vermek zorundasınız**. İki
> parça birden gereklidir:
>
> 1. Servis/komut satırında fiziksel UART: `--serial2 /dev/ttyS3`
> 2. Parametre dosyasında protokol etiketi: `SERIAL2_PROTOCOL 23` (RCIN)
>
> İkisinden biri eksikse RC girişi hiç görünmez ve prearm `RC not
> found` der. Alıcının kumandayı görüyor olması (LED sabit) bu durumu
> değiştirmez — sinyal kartın UART'ına ulaşır, ancak ArduPilot o portu
> dinlemez.

### Hangi UART?

RC giriş pini 40-pin başlıkta **GPIO-15 = UART-MAIN1 RX**'tir. Kartta
karşılığı:

```bash
for d in /sys/class/tty/ttyS*; do
  dev=$(readlink -f $d/device 2>/dev/null)
  [ -n "$dev" ] && echo "$(basename $d) -> $(basename $dev)"
done
```

```
ttyS2 -> 2800000.serial      <-- konsol (serial-getty calisiyor), KULLANMAYIN
ttyS3 -> 2810000.serial      <-- UART-MAIN1, RC girisi
```

Device tree bunu doğrular — `serial@2810000` düğümünün `symlink`
özelliği `ttyAMA0`, `status` = `okay`. Portun boşta olduğunu teyit edin
(getty yalnızca `ttyS2` ve `ttyGS0` üzerinde olmalıdır):

```bash
systemctl list-units --type=service --state=running | grep -i getty
```

**Baud ayarı gerekmez.** `AP_RCProtocol::check_added_uart()`
baud/parite/inversiyon kombinasyonlarını protokol kilitlenene kadar
sırayla dener, dolayısıyla iBUS (115200 8N1) otomatik yakalanır.
`RC_PROTOCOLS` varsayılanı `1` = hepsi etkin, dokunmaya gerek yoktur.

> **Overlay yan etkisi:** `epwm0-gpio5-gpio14` overlay'i `main_uart1`'i
> RX-only yapar. iBUS/SBUS tek yönlü (alıcı → kart) olduğu için bu
> **sorun değildir**. Ancak iBUS *telemetri* (çift yönlü) kullanmayı
> planlıyorsanız o overlay'i kaldırmanız gerekir — bu durumda PWM
> kanal 0 ve 1'i kaybedersiniz.

### Doğrulama (Alıcı Takılıyken)

```python
from pymavlink import mavutil
m = mavutil.mavlink_connection('tcp:192.168.7.2:5760')
m.wait_heartbeat(timeout=25)
m.mav.request_data_stream_send(m.target_system, m.target_component,
                               mavutil.mavlink.MAV_DATA_STREAM_RC_CHANNELS, 5, 1)
rc = m.recv_match(type='RC_CHANNELS', blocking=True, timeout=20)
print('chancount=%d' % rc.chancount,
      [getattr(rc, 'chan%d_raw' % i) for i in range(1, 9)])
```

`chancount` > 0 ve kanal değerleri ~1000–2000 aralığında olmalıdır.
`chancount=0` ve tüm kanallar sıfırsa sinyal ArduPilot'a ulaşmıyor
demektir.

Alıcının gerçekten veri gönderdiğini bağımsız olarak sınamak için —
önce ArduPilot'u durdurun, aksi hâlde portu o tutar:

```bash
sudo systemctl stop ardupilot
sudo stty -F /dev/ttyS3 115200 raw -echo cs8 -parenb -cstopb
sudo timeout 3 dd if=/dev/ttyS3 bs=1 count=200 2>/dev/null | od -An -tx1 | head
```

iBUS çerçeveleri `20 40` ile başlayan 32 baytlık bloklar hâlinde
görünür. Hiç bayt gelmiyorsa sorun kablolamadadır (RX pini, ortak GND,
alıcı beslemesi) — yazılımda değil. Test bittiğinde servisi geri
başlatın:

```bash
sudo systemctl start ardupilot
```

---

Devam etmek için
[10-servis-ve-yer-istasyonu.md](10-servis-ve-yer-istasyonu.md).
