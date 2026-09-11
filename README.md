# Cisco WS-C3560-24PS — Iskilip Network Lab

Cisco Packet Tracer üzerinde yapılandırılmış C3560 layer-3 switch projesi.

## Kullanılan Cihaz

| Özellik | Değer |
|---|---|
| Model | WS-C3560-24PS-E |
| IOS | 12.2(37)SE1 — ADVIPSERVICESK9 |
| Port | 24x FastEthernet + 2x GigabitEthernet |
| Bellek | 122880K/8184K bytes |
| MAC | 0030.F2D4.CE7C |

## Proje İçeriği

- `Iskilip-Network.pkt` — Packet Tracer topoloji dosyası
- `configs/switch-running-config.txt` — Switch çalışan yapılandırması
- `commands.md` — Kullanılan tüm komutlar (adım adım)

## Yapılan İşlemler

### 1. VLAN 1 IP Yapılandırması
```
interface vlan 1
  no shutdown
  ip address 10.19.2.254 255.255.255.0
```

### 2. Default Gateway Tanımlama
```
ip default-gateway 10.19.2.1
```

### 3. TFTP'den Konfigürasyon Geri Yükleme
```
copy tftp: running-config
  Host: 10.6.2.2
  Dosya: Iskilip
```
→ 1531 byte başarıyla yüklendi, hostname **Iskilip** olarak değişti.

### 4. Ping Testi
```
ping 10.6.2.2   → %80 başarı (4/5)
```

## Dosyayı Çalıştırmak

1. [Cisco Packet Tracer](https://www.netacad.com/courses/packet-tracer) uygulamasını açın.
2. `Iskilip-Network.pkt` dosyasını açın.
3. Switch console'una bağlanarak komutları test edebilirsiniz.
