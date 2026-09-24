# Güvenlik ve Dayanıklılık

## Kimlik Zinciri

Retivexa'da bir kurulum kimliği doğrudan yetki olarak kullanılmaz. Güven zinciri ayrı aşamalardan oluşur:

1. Süreli ve tek kullanımlık provisioning token bootstrap yetkisi sağlar.
2. Başarılı provisioning, Agent kurulumunu belirli bir müşteri hesabına bağlar.
3. Uzun ömürlü Agent credential merkezi tarafta doğrulanabilir biçimde saklanır; yerelde DPAPI ile korunur.
4. Agent, credential ile kısa ömürlü JWT bearer token alır.
5. Korunan endpoint'ler sunucu tarafında müşteri ve şirket kapsamına göre yetkilendirilir.
6. Storage entitlement, artifact'in merkezi saklamaya girip giremeyeceğini ayrıca belirler.

Bu ayrım, ele geçirilmiş tek bir identifier'ın kalıcı ve sınırsız yetkiye dönüşmesini engelleyen defense-in-depth yaklaşımıdır.

## Yerel Güven Sınırı

Windows Service `LocalSystem` hesabında çalışır ve aşağıdaki hassas durumun sahibidir:

- installation identity,
- DPAPI korumalı Agent credential,
- policy ve entitlement cache,
- artifact ve delivery state,
- authentication ve delivery iş akışları,
- diagnostics üretimi.

ProgramData kökünde kalıtım kapatılır; explicit ACL'lerle LocalSystem ve Administrators yetkilendirilir. Tray bu dosyaları doğrudan okumaz. Destek amaçlı sanitize edilmiş diagnostics export dizini, interaktif kullanıcının çıktıyı kopyalayabilmesi için daha dar bir read sınırı sunar.

## Credential ve Token Yönetimi

- Agent credential güvenli biçimde oluşturulur.
- Merkezde credential'ın SHA-256 temsili/verifier'ı tutulur.
- Yerel credential, Windows DPAPI `LocalMachine` ile korunur.
- DPAPI koruması ACL korumasının yerine geçmez; iki kontrol birlikte uygulanır.
- Bearer token kısa ömürlüdür, process-local cache'te tutulur ve ProgramData'ya yazılmaz.
- Central revocation ve validation otoritesini korur.

## Yetkilendirme ve Fail-Closed Davranış

Filesystem visibility bir iş yetkisi değildir. Agent bir dosyayı görebilse bile aktif storage entitlement yoksa merkezi teslimata sokmaz.

Geçici Central kesintisinde son başarılı entitlement snapshot'ı korunabilir. Ancak Central'a yeniden bağlanıp daha yeni bir snapshot başarıyla alındığında — yetki kaldırılmış olsa dahi — bu yeni durum yerel cache'in yerini alır. Böylece kullanılabilirlik, eski yetkinin kalıcı olarak kazanmasına dönüşmez.

## IPC Sınırı

Tray ve Service sürümlü Windows Named Pipe üzerinden haberleşir. Pipe için explicit Windows ACL uygulanır. Bu tasarım:

- UI sürecini privileged state sahibi yapmaz,
- Service'in kullanıcı oturumu olmadan çalışmasını sürdürür,
- Tray'in Service restart'ından sonra yeniden bağlanabilmesini sağlar,
- provisioning ve diagnostics gibi desteklenen operasyonları kontrollü bir sözleşmeye bağlar.

## Kesintiden Toparlanma

Artifact ve teslimat state'i `%ProgramData%\Retivexa\Agent\state.json` altında dayanıklıdır. Başlangıç sırası, yeni iş almadan önce yarım kalmış teslimatları toparlayacak biçimde tasarlanmıştır.

Başlıca senaryolar:

| Senaryo | Beklenen davranış |
|---|---|
| Service restart | Kimlik, credential ve state yeniden yüklenir; teslimatlar devam eder. |
| Windows reboot | Automatic Service ayağa kalkar; ilk reconciliation ve recovery çalışır. |
| Central geçici olarak kapalı | Son geçerli authority korunur, Agent degraded/offline durumda çalışır. |
| Central geri geldi | Service restart gerekmeden token yenilenir, policy/entitlement yeniden senkronize edilir. |
| Teslimat yarıda kesildi | Attempt ve correlation bilgisi korunur; retry ile tamamlanır. |
| MSI major upgrade | Binary'ler güncellenir; ProgramData kimliği ve state korunur. |
| Normal uninstall + reinstall | Korunan state yeniden kullanılır; sırf reinstall nedeniyle reprovisioning yapılmaz. |

## Diagnostics Güvenliği

Diagnostics paketi ham ProgramData kopyası değildir. Seçili runtime ve sanitize edilmiş log bilgisi içerir.

Normal export'a özellikle dahil edilmeyen içerikler:

- Agent credential,
- bearer token,
- installation identity dosyası,
- entitlement cache,
- artifact state dosyası,
- artifact payload'ları,
- müşteri finansal içeriği.

Correlation ID operasyonları ilişkilendirmek için kullanılır; kimlik doğrulama veya yetkilendirme sağlamaz.
