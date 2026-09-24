# Doğrulama ve Test Yaklaşımı

## Neden Katmanlı Kabul?

Retivexa'nın kritik davranışları yalnızca unit veya integration test ile kanıtlanamaz. DPAPI, Windows Service hesabı, Named Pipe ACL'leri, gerçek reboot ve Windows Installer yaşam döngüsü gerçek işletim sistemi sınırlarına bağlıdır.

Bu nedenle kabul yaklaşımı üç katmandan oluşur:

1. Otomatik regresyon testleri
2. Uçtan uca runtime acceptance testleri
3. Kurulu Windows ortamında MSI lifecycle acceptance

## Otomatik ve Uçtan Uca Kapsam

| Alan | Doğrulanan davranış |
|---|---|
| Provisioning | Geçerli tek kullanımlık token tüketilir; installation müşteriyle eşleşir. |
| Authentication | Gerçek Agent credential kısa ömürlü bearer token üretir. |
| Entitlement | Merkezi yetki senkronize edilir; kaldırılan yetki başarılı sync sonrası yerelde de kalkar. |
| İlk artifact | Keşif, parse, validation, SHA-256, yetki kontrolü ve merkezi kayıt zinciri çalışır. |
| Değişen artifact | Yeni fingerprint yeni artifact sürümü oluşturur. |
| Değişmeyen artifact | Aynı içerik duplicate merkezi sürüm üretmez. |
| Kalıcı state | Restart doğruluğu yalnızca in-memory veriye dayanmaz. |
| Interrupted delivery | Yarım iş aynı attempt/correlation bağlamıyla toparlanır. |
| Release discovery | Authenticated Agent uygun release bilgisini alır. |

## Gerçek Windows Kabul Senaryoları

| Senaryo | Kabul ölçütü |
|---|---|
| Fresh install | MSI kurulur; Service `Automatic / LocalSystem` olarak çalışır. |
| İlk provisioning | Token Tray → Named Pipe → Service → Central zincirinde tüketilir; credential DPAPI ile korunur. |
| Tray reconnect | Service restart sonrası Tray'i yeniden başlatmadan bağlantı kurulur. |
| Service restart | Kalıcı identity, credential ve artifact state korunur. |
| Gerçek reboot | Windows yeniden başladıktan sonra Service otomatik kalkar ve recovery tamamlanır. |
| Offline startup | Son geçerli authority korunur; Agent degraded modda başlar. |
| Central recovery | Service restart olmadan yeni token ve güncel entitlement alınır. |
| Downgrade | Daha eski MSI kurma girişimi reddedilir. |
| Major upgrade | Yeni MSI kurulur; Service ve ProgramData continuity korunur. |
| Explicit uninstall | Service, binary ve startup registration kalkar; çalışan Tray orphan kalmaz. |
| ProgramData preservation | Normal uninstall operasyonel state'i silmez. |
| Reinstall | Aynı identity ve DPAPI credential yeniden kullanılır; yeni provisioning gerekmez. |

## Son Kayıtlı Sonuç

`v0.4.0` doğrulama kaydı:

```text
568 passed
0 failed
```

Release acceptance yalnızca test komutunun yeşil dönmesine bağlanmaz. Final MSI yeniden üretilir, SHA-256 hash'i kaydedilir ve kabul kanıtı kullanılan aynı artifact ile ilişkilendirilir.

## Kabulde Bilinçli Olarak Yasaklanan Kestirmeler

Aşağıdaki yollar başarı ölçütü sayılmaz:

- yetki vermek için yerel entitlement cache'i elle düzenlemek,
- normal toparlanma için ProgramData'yı silmek,
- her Central kesintisinde Service'i manuel yeniden başlatmak,
- gerçek Windows reboot yerine yalnızca Service restart yapmak,
- `msiexec` dönüş kodunu tek başına başarılı upgrade kanıtı kabul etmek.

Bu kurallar, test ortamındaki kolaylaştırmaların production davranışı gibi görünmesini engeller.
