# 10. Kalıcı Kurulum, Servis ve Yer İstasyonu Bağlantısı

Bu bölüm, binary'yi kartta kalıcı bir kuruluma dönüştürür: sabit dizin
yapısı, parametre varsayılanları, açılışta otomatik başlayan systemd
servisi ve yer istasyonu bağlantısı.

## Dizin Yapısı — Yollar Derleme Zamanında Sabittir

Aşağıdaki yollar `hwdef.dat` içinde derleme zamanında sabitlenir,
çalışma anında değiştirilemez:

```
define HAL_BOARD_LOG_DIRECTORY     "/opt/ardupilot/var/log"
define HAL_BOARD_TERRAIN_DIRECTORY "/opt/ardupilot/var/terrain"
define HAL_BOARD_STORAGE_DIRECTORY "/opt/ardupilot/var/storage"
define HAL_PARAM_DEFAULTS_PATH     "/opt/ardupilot/etc/ardupilot.parm"
```

Kartta dizinleri oluşturup dosyaları yerleştirin:

```bash
sudo mkdir -p /opt/ardupilot/bin /opt/ardupilot/etc \
              /opt/ardupilot/var/log /opt/ardupilot/var/terrain \
              /opt/ardupilot/var/storage
sudo install -m 0755 /tmp/arducopter        /opt/ardupilot/bin/arducopter
sudo install -m 0644 /tmp/ardupilot.parm    /opt/ardupilot/etc/ardupilot.parm
sudo install -m 0644 /tmp/ardupilot.service /etc/systemd/system/ardupilot.service
```

> Dosyaların karta nasıl aktarılacağı (WSL → Windows → `scp`) için
> [5. bölüme](05-karta-aktarim-ve-calistirma.md) bakınız.

## Parametre Varsayılanları — `ardupilot.parm`

`/opt/ardupilot/etc/ardupilot.parm`:

```
# --- MAVLink -----------------------------------------------------------
SERIAL0_PROTOCOL 2      # MAVLink2 — UDP taşıması üzerinde
SERIAL1_PROTOCOL 2      # MAVLink2 — TCP taşıması üzerinde

# --- RC girişi ---------------------------------------------------------
SERIAL2_PROTOCOL 23     # RCIN — iBUS/SBUS alıcı (/dev/ttyS3)

# --- Gövde -------------------------------------------------------------
FRAME_CLASS 1           # Quad
FRAME_TYPE  1           # X
```

> **`SERIAL2_PROTOCOL 23` tek başına yetmez** — `ardupilot.service`
> içinde `--serial2 /dev/ttyS3` de bulunmalıdır. Gerekçe:
> [9. bölüm — RC girişi](09-mavlink-ve-rc-girisi.md).

> **`FRAME_CLASS` neden gerekli:** Ayarlanmazsa araç arm olmayı
> reddeder (`PreArm: Motors: Check frame class and type`) ve hiç motor
> çıkışı üretmez. `1/1` (Quad/X) konvansiyonel bir başlangıç
> noktasıdır — **uçurmadan önce gerçek gövdenize göre değiştirin.**
> Kart 7 PWM kanalı sunduğu için hexa (`2`) ve octa (`3`) da
> mümkündür.

> Bu dosya **varsayılan** verir, değer dayatmaz. Yer istasyonundan
> yazılan parametreler `/opt/ardupilot/var/storage` içinde saklanır ve
> bu dosyanın önüne geçer. Yani bir parametreyi burada değiştirmek,
> storage'da zaten bir değeri varsa etkisiz kalır.

## systemd Servisi — `ardupilot.service`

`/etc/systemd/system/ardupilot.service`:

```ini
[Unit]
Description=ArduPilot ArduCopter (T3 Gemstone O1)
Documentation=https://docs.t3gemstone.org/tr/projects/ardupilot
# SPI/GPIO yereldir, ancak MAVLink UDP soketi ağ yığınının ayakta olmasını ister
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
WorkingDirectory=/opt/ardupilot

ExecStart=/opt/ardupilot/bin/arducopter \
    --serial0 udp:192.168.7.255:14550:bcast \
    --serial1 tcp:0.0.0.0:5760 \
    --serial2 /dev/ttyS3

# hızlı döngü SCHED_FIFO ve kilitli sayfalar ister
LimitRTPRIO=99
LimitMEMLOCK=infinity

Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Üç bağlantının anlamı:

| Argüman | İşlev |
|---|---|
| `--serial0 udp:192.168.7.255:14550:bcast` | Alt ağa UDP yayını. Yer istasyonu yalnızca 14550'i dinler, IP yazmaz. |
| `--serial1 tcp:0.0.0.0:5760` | TCP sunucu. Yer istasyonu `192.168.7.2:5760`'a bağlanır; en güvenilir bağlantı. |
| `--serial2 /dev/ttyS3` | UART-MAIN1 RX — RC girişi (iBUS/SBUS) |

Etkinleştirme:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now ardupilot
sudo systemctl status ardupilot
journalctl -u ardupilot -f
```

### Elle Çalıştırma (Test İçin)

```bash
sudo systemctl stop ardupilot
sudo /opt/ardupilot/bin/arducopter \
    --serial0 udp:192.168.7.255:14550:bcast \
    --serial1 tcp:0.0.0.0:5760 \
    --serial2 /dev/ttyS3
```

Arka planda çalıştırmak ve durdurmak:

```bash
sudo nohup /opt/ardupilot/bin/arducopter \
    --serial0 udp:192.168.7.255:14550:bcast \
    --serial1 tcp:0.0.0.0:5760 > /tmp/ap.log 2>&1 &

sudo pkill -f arducopter
```

## Yer İstasyonu Bağlantısı

### Mission Planner — TCP (önerilen)

1. Sağ üstteki açılır kutudan **`TCP`** seçin
2. **Connect**
3. Sorulan kutulara: *host* → `192.168.7.2`, *port* → `5760`

Bağlantıyı Mission Planner başlattığı için güvenlik duvarına kural
eklemek gerekmez. Kapatıp açmak da sorunsuzdur.

### Mission Planner — düz UDP

Linux host'ta önce güvenlik duvarı kuralı:

```bash
sudo ufw allow 14550/udp
```

Sonra **`UDP`** seçip **Connect**; yalnızca port sorar → `14550`. IP
yazmanız gerekmez, kart yayın yaptığı için kendisi bulunur.

> ### ⚠ Güvenlik duvarı UDP yayınını engeller
>
> Doğrulanmış test sonucu:
>
> | Bağlantı | Sonuç |
> |---|---|
> | `tcp:192.168.7.2:5760` | ✔ heartbeat alındı |
> | UDP broadcast 14550 | ✘ 25 sn boyunca hiç paket gelmedi |
>
> Sebep: TCP'de bağlantıyı host başlattığı için conntrack dönüş
> paketlerine izin verir. Broadcast ise **talep edilmemiş gelen
> trafiktir** ve düşürülür. Belirti tam olarak *"portu seçiyorum ama
> bulamıyor"* şeklindedir.

> **Geçmişteki bir tuzak:** Kart `udpin:0.0.0.0:14550` ile
> çalıştırıldığında kart da *dinler*, Mission Planner'ın `UDP` seçeneği
> de *dinler* — iki taraf da beklediği için bağlantı hiç kurulmaz. O
> modda `UDPCl` (istemci) seçilip `192.168.7.2:14550` yazmak gerekir.
> Bu bölümdeki broadcast + TCP yapılandırmasında bu sorun yoktur.

### QGroundControl

*Application Settings → Comm Links → Add* → Type `TCP`,
Host `192.168.7.2`, Port `5760`.

### MAVProxy / pymavlink

```bash
mavproxy.py --master=tcp:192.168.7.2:5760
```

```python
from pymavlink import mavutil
m = mavutil.mavlink_connection('tcp:192.168.7.2:5760')
m.wait_heartbeat()
print('heartbeat: sysid=%d compid=%d' % (m.target_system, m.target_component))
```

---

Devam etmek için
[11-dogrulama-ve-kalibrasyon.md](11-dogrulama-ve-kalibrasyon.md).
