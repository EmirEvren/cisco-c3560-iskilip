# Kullanılan Cisco Komutları — C3560 Iskilip

## 1. Privileged EXEC moda geçiş
```
Switch> enable
```

## 2. Bağlantı testi (başarısız — IP henüz yok)
```
Switch# ping 10.6.2.2
! → %0 başarı
```

## 3. Global konfigürasyona giriş
```
Switch# configure terminal
```

## 4. VLAN 1 arayüz yapılandırması
```
Switch(config)# interface vlan 1
Switch(config-if)# no shutdown
Switch(config-if)# ip address 10.19.2.254 255.255.255.0
Switch(config-if)# exit
```

## 5. Default gateway tanımlama
```
Switch(config)# ip default-gateway 10.19.2.1
Switch(config)# exit
```

## 6. Bağlantı testi (başarılı)
```
Switch# ping 10.6.2.2
! → %80 başarı (4/5)
```

## 7. TFTP'den running-config geri yükleme
```
Switch# copy tftp: running-config
Address or name of remote host []? 10.6.2.2
Source filename []? Iskilip
Destination filename [running-config]? (Enter)
! → 1531 bytes kopyalandı, hostname Iskilip oldu
```
