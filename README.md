# Cisco WS-C3560-24PS — VLAN Yapılandırması, TFTP ile Konfigürasyon Yönetimi ve IOS Firmware Güncellemesi

> **Packet Tracer Lab Çalışması**
> Bu projede Cisco WS-C3560-24PS switch üzerinde sıfırdan başlanarak:
> VLAN 1 yönetim IP'si atandı, TFTP sunucusundan mevcut konfigürasyon geri yüklendi
> ve IOS firmware güncelleme süreci uygulandı.

---

## İçindekiler

1. [Kullanılan Cihaz](#1-kullanılan-cihaz)
2. [Ağ Topolojisi](#2-ağ-topolojisi)
3. [Açılış Süreci (Boot)](#3-açılış-süreci-boot)
4. [VLAN 1 IP Yapılandırması](#4-vlan-1-ip-yapılandırması)
5. [Default Gateway Tanımlama](#5-default-gateway-tanımlama)
6. [TFTP ile Konfigürasyon Geri Yükleme](#6-tftp-ile-konfigürasyon-geri-yükleme)
7. [IOS Firmware Güncellemesi](#7-ios-firmware-güncellemesi)
8. [Kullanılan Komutlar (Özet Tablo)](#8-kullanılan-komutlar-özet-tablo)

---

## 1. Kullanılan Cihaz

| Özellik | Değer |
|---|---|
| **Model** | WS-C3560-24PS-E |
| **Açılış IOS** | 12.2(25r)SEC (Boot Loader) |
| **Çalışan IOS** | 12.2(37)SE1 — C3560-ADVIPSERVICESK9-M |
| **Port** | 24x FastEthernet + 2x GigabitEthernet |
| **RAM** | 122880K / 8184K bytes |
| **Flash** | 64 MB (64.016.384 bytes) |
| **Kullanılan Flash** | ~8.9 MB (IOS image) |
| **MAC Adresi** | 0030.F2D4.CE7C |
| **Seri No** | CAT1037RJF7 |

---

## 2. Ağ Topolojisi

```
[Switch WS-C3560-24PS]
   VLAN 1 IP : 10.19.2.254 / 24
   Gateway   : 10.19.2.1
        |
        | GigabitEthernet0/2
        |
   [TFTP Sunucu]
   IP: 10.6.2.2
   Dosya: Iskilip (running-config)
         + IOS image (firmware)
```

---

## 3. Açılış Süreci (Boot)

Switch açıldığında Boot Loader devreye girer ve Flash'taki IOS imajını yükler.

```
Boot Loader: C3560-HBOOT-M Version 12.2(25r)SEC

Yüklenen imaj:
  flash:/c3560-advipservicesk9-mz.122-37.SE1.bin

POST (Power-On Self Test) sonuçları:
  ✓ CPU MIC register Tests       : Passed
  ✓ PortASIC Memory Tests        : Passed
  ✓ CPU MIC interface Loopback   : Passed
  ✓ PortASIC RingLoopback        : Passed
  ✓ Inline Power Controller      : Passed
  ✓ PortASIC CAM Subsystem       : Passed
  ✓ PortASIC Port Loopback       : Passed
```

> **POST nedir?**
> Switch açılırken donanım bileşenlerini test eden yerleşik tanılama sürecidir.
> Tüm testlerin "Passed" dönmesi cihazın sağlıklı olduğunu gösterir.

---

## 4. VLAN 1 IP Yapılandırması

Switch'e management (yönetim) IP adresi atamak için **VLAN 1** sanal arayüzü kullanılır.

```cisco
Switch> enable
Switch# configure terminal

Switch(config)# interface vlan 1
Switch(config-if)# no shutdown          ! Arayüzü aktif hale getir
Switch(config-if)# ip address 10.19.2.254 255.255.255.0
Switch(config-if)# exit
```

**Neden VLAN 1?**
C3560 bir Layer-3 switch olsa da, yönetim trafiği için doğrudan bir fiziksel port yerine SVI (Switched Virtual Interface) olan VLAN 1 kullanılır. Bu yaklaşım tüm portların aynı yönetim IP'sine erişmesini sağlar.

---

## 5. Default Gateway Tanımlama

Switch'in kendi ağı dışındaki hedeflere (ör. TFTP sunucusu) ulaşabilmesi için varsayılan çıkış yolu tanımlanır.

```cisco
Switch(config)# ip default-gateway 10.19.2.1
```

**Doğrulama:**
```cisco
Switch# ping 10.6.2.2
! Çıktı: Success rate is 80 percent (4/5)
```

> İlk paket ARP çözümlemesi nedeniyle kaybolur; bu normal davranıştır.

---

## 6. TFTP ile Konfigürasyon Geri Yükleme

TFTP (Trivial File Transfer Protocol), Cisco cihazlarında konfigürasyon ve imaj transferi için kullanılan hafif bir protokoldür.

### Adımlar

```cisco
Switch# copy tftp: running-config
Address or name of remote host []? 10.6.2.2
Source filename []? Iskilip
Destination filename [running-config]? (Enter — varsayılanı kabul et)

Accessing tftp://10.6.2.2/Iskilip...
Loading Iskilip from 10.6.2.2: !
[OK - 1531 bytes]

1531 bytes copied in 0 secs
```

### Sonuç

Konfigürasyon yüklendikten sonra switch **hostname** değişti:
```
Switch# → Iskilip#
```

Bu, TFTP'den gelen `Iskilip` dosyasındaki `hostname Iskilip` satırının uygulandığını gösterir.

### Neden TFTP?

| Yöntem | Avantaj | Dezavantaj |
|---|---|---|
| TFTP | Hızlı, basit, Cisco native | Şifreleme yok |
| SCP | Güvenli (SSH şifreli) | Ek yapılandırma gerekir |
| USB | Ağ bağlantısı gerekmez | Fiziksel erişim şart |

---

## 7. IOS Firmware Güncellemesi

IOS güncellemesi de aynı TFTP altyapısıyla yapılır. Yeni imaj Flash'a yüklenir, ardından boot parametresi güncellenerek cihaz yeniden başlatılır.

### Adım 1 — Mevcut durumu kontrol et

```cisco
Iskilip# show version
! Mevcut: Version 12.2(37)SE1

Iskilip# show flash:
! Kullanılabilir alan: ~55 MB
! Gerekli: yeni imaj boyutuna göre değişir (genellikle 10-20 MB)
```

### Adım 2 — Yeni IOS imajını TFTP'den indir

```cisco
Iskilip# copy tftp: flash:
Address or name of remote host []? 10.6.2.2
Source filename []? c3560-advipservicesk9-mz.122-55.SE.bin
Destination filename [c3560-advipservicesk9-mz.122-55.SE.bin]? (Enter)

Loading c3560-advipservicesk9-mz.122-55.SE.bin from 10.6.2.2:
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
[OK - 11.3 MB]
```

### Adım 3 — Boot parametresini güncelle

```cisco
Iskilip# configure terminal
Iskilip(config)# boot system flash:c3560-advipservicesk9-mz.122-55.SE.bin
Iskilip(config)# end
```

### Adım 4 — Konfigürasyonu kaydet

```cisco
Iskilip# copy running-config startup-config
! veya kısaca:
Iskilip# write memory
```

### Adım 5 — Cihazı yeniden başlat

```cisco
Iskilip# reload
Proceed with reload? [confirm] (Enter)
```

### Adım 6 — Güncellemeyi doğrula

```cisco
Iskilip# show version
! Yeni sürüm: 12.2(55)SE
```

> **⚠️ Dikkat:** Reload sırasında switch birkaç dakika erişilemez olur.
> Üretim ortamında bu işlemi maintenance window'da (bakım penceresi) yapın.

---

## 8. Kullanılan Komutlar (Özet Tablo)

| Komut | Açıklama |
|---|---|
| `enable` | Privileged EXEC moduna geç |
| `configure terminal` | Global konfigürasyon moduna gir |
| `interface vlan 1` | VLAN 1 SVI arayüzüne gir |
| `no shutdown` | Arayüzü aktif et |
| `ip address 10.19.2.254 255.255.255.0` | IP adresi ata |
| `ip default-gateway 10.19.2.1` | Varsayılan ağ geçidini tanımla |
| `ping 10.6.2.2` | Bağlantıyı test et |
| `copy tftp: running-config` | TFTP'den konfigürasyon yükle |
| `copy tftp: flash:` | TFTP'den IOS imajı indir |
| `boot system flash:<dosya>` | Boot imajını ayarla |
| `copy running-config startup-config` | Konfigürasyonu kaydet |
| `reload` | Cihazı yeniden başlat |
| `show version` | IOS sürümünü göster |
| `show flash:` | Flash içeriğini listele |

---

## Dosya Yapısı

```
cisco-c3560-iskilip/
├── Iskilip-Network.pkt          # Packet Tracer topoloji dosyası
├── configs/
│   └── switch-running-config.txt  # Switch konfigürasyonu
├── commands.md                  # Komutlar (detaylı)
└── README.md                    # Bu dosya
```

---

## Gereksinimler

- [Cisco Packet Tracer](https://www.netacad.com/courses/packet-tracer) (v8.x önerilir)
- Cisco NetAcad hesabı (ücretsiz)

---

*Cisco WS-C3560-24PS — IOS 12.2(37)SE1 — Packet Tracer Lab*
