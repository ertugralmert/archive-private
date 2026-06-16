# IBM InfoSphere Information Server — Reboot Sonrası DB2/DataStage Otomatik Başlamama Sorunu

**Sunucu:** miatsdst06.migros.local
**Ortam:** RHEL 9.6 — Information Server 11.7.1.6 — DB2 v11.5.9.0 (Production)
**Sorun tipi:** Reboot sonrası servislerin otomatik başlamaması
**Kök neden:** SAN multipath mount timing — DB2 binary'leri mount gelmeden çalıştırılmaya çalışılıyor

---

## 1. Sorunun Özeti

Sunucu reboot edildikten sonra DataStage client'ları sunucuya bağlanamıyordu. İlk bakışta "autostart çalışmıyor" gibi görünen sorun, aslında **bir mount-timing zinciri** problemiydi:

> `/datastage` (SAN multipath disk) boot sırasında geç geliyor → DB2 binary'leri bu mount üzerinde olduğu için DB2 başlatıcısı (`db2fmcd`) mount gelmeden çalışıp `203/EXEC` hatasıyla ölüyor → DB2 instance hiç başlamıyor → XMETA repository erişilemez → WebSphere/DataStage yarım kalıyor.

Çözüm: DB2 ve WebSphere servislerine **systemd drop-in override** ile mount bağımlılığı eklemek. IBM'in orijinal dosyalarına dokunmadan.

---

## 2. Teşhis Süreci (Adım Adım Mantık)

Bu bölüm sadece "ne yaptık" değil, **"neden o komutu çalıştırdık ve çıktıdan ne anladık"** mantığını anlatır. Asıl öğrenilecek kısım budur.

### 2.1 — İlk gözlem: WebSphere durumu

```sh
ps -ef | grep server1 | grep -v kafka
```

**Neden:** WebSphere'in (server1 prosesi) çalışıp çalışmadığını görmek için.
**Mantık:** `grep -v kafka` ekledik çünkü Kafka komut satırında "server1" geçebiliyor ve yanlış eşleşme yaratıyor. Çıktıda sadece `grep` komutunun kendisi görünüyorsa (yani gerçek bir server1 prosesi yoksa), WebSphere durmuş demektir.

> **Öğreti:** `ps -ef | grep X` çıktısında her zaman `grep`'in kendisi de görünür. `grep -v grep` veya spesifik filtrelerle bunu eler, gerçek prosesi ayırt ederiz.

### 2.2 — Mount yapısını anlamak

```sh
mount | grep datastage
grep datastage /etc/fstab
findmnt /datastage
lsblk
pvs
```

**Neden:** `/datastage` dizininin ne tür bir disk olduğunu anlamak. Çünkü autostart scriptleri bu dizini (`/datastage/IBM/InformationServer/...`) kullanıyor ve daha önce buraya `sleep` eklenmişti.

**Çıktıdan öğrendiklerimiz:**
- `/datastage` = `/dev/mapper/data_vg-data_lv` (xfs)
- `data_vg` → `/dev/mapper/mpatha` (**multipath**) → fiziksel `sdb/sdc/sdd/sde` (4 yol = klasik **SAN multipath**)
- fstab'da `_netdev` option'ı var

> **Kritik öğreti:** `/datastage` lokal disk değil, **SAN üzerinden gelen bir LUN**. SAN diskleri ve multipath, boot'un erken aşamasında (`local-fs.target`) hazır DEĞİLDİR — network/SAN katmanı gelene kadar beklenir. `_netdev` option'ı tam da bu yüzden doğru: mount'u `remote-fs.target`'a bağlar, böylece SAN hazır olmadan mount denemesi yapılmaz.

> **Neden önemli:** SAN mount geç geldiği için, bu mount üzerindeki binary'leri kullanan her servis, mount'tan önce çalışırsa "dosya yok" hatası alır.

### 2.3 — Sürüm ve DataStage Engine autostart kontrolü

```sh
grep -i version /datastage/IBM/InformationServer/Version.xml
su - dsadm -c '. /datastage/IBM/InformationServer/Server/DSEngine/dsenv; \
  /datastage/IBM/InformationServer/Server/DSEngine/bin/uv -admin -info'
```

**Neden:** IBM'in "DSEngine RHEL'de otomatik başlamıyor" dokümanı (Doc 538445) sürüme göre farklı çözüm öneriyor. Önce sürümü ve DSEngine'in durumunu bilmemiz gerekti.

**Çıktıdan öğrendiklerimiz:**
- Sürüm: **11.7.1.6** → IBM dokümanına göre "11.7.1.x değişiklik gerektirmez"
- DSEngine: `Autostart mode: enabled`, native systemd servisi (`ds.rc.service`), çalışıyor

> **Öğreti:** DSEngine zaten sorunsuz çalışıyordu. Sorun DSEngine'de değildi. İlk sandığımız yer (DSEngine autostart) yanlış çıktı — bu yüzden teşhise devam ettik. **Doğru teşhis, yanlış yolları elemekten geçer.**

### 2.4 — Asıl sorunu bulmak: failed servisler

```sh
systemctl --failed
```

**Neden:** Boot sonrası hangi servisin fail ettiğini tek komutla görmek.
**Çıktı:** `db2fmcd.service — DB2 v11.5.9.0 — failed`

> **Dönüm noktası:** Asıl fail eden servis DB2 fault monitor'üydü. DataStage/WebSphere değil. Sorunun kökü DB2 katmanındaydı.

### 2.5 — db2fmcd neden fail etti?

```sh
systemctl status db2fmcd.service --no-pager -l
journalctl -b -1 -u db2fmcd.service --no-pager
```

**Çıktının kritik satırı:**
```
ExecStart=/datastage/IBM/DB2/bin/db2fmcd (code=exited, status=203/EXEC)
db2fmcd.service: Start request repeated too quickly.
```

> **Kök neden netleşti:** `status=203/EXEC` systemd'de çok spesifik bir hatadır: **"belirtilen executable çalıştırılamadı"** — dosya yok, erişilemiyor veya çalıştırma izni yok.
>
> Yol dikkat: `/datastage/IBM/DB2/bin/db2fmcd` — yani **DB2 da SAN mount üzerinde!**
>
> Zincir: boot'ta `db2fmcd` çok erken çalıştı (14:29:55) → `/datastage` henüz mount edilmemişti → binary fiziksel olarak yoktu → `203/EXEC` → systemd 5 kez denedi, hepsinde aynı → "repeated too quickly" deyip pes etti. Mount ~23 saniye sonra geldi ama systemd çoktan vazgeçmişti.

### 2.6 — Dosyanın gerçekten var olduğunu ve izinlerini doğrulama

```sh
ls -l /datastage/IBM/DB2/bin/db2fmcd
mount | grep noexec
```

**Çıktı:** `-r-xr-xr-x 1 bin bin 79280 ... db2fmcd` (izinler tamam) + `/datastage` `noexec` listesinde YOK.

> **Öğreti:** Bu kontrol `203/EXEC`'in alternatif sebeplerini eledi. İzin sorunu değil (`r-xr-xr-x` var), `noexec` mount sorunu değil. Geriye tek sebep kaldı: **mount-timing**. Dosya o saniyede fiziksel olarak yoktu. Teşhis kesinleşti.

### 2.7 — DB2 instance gerçekten çalışıyor mu?

```sh
ps -ef | grep db2sysc | grep -v grep
```

**Çıktı:** BOŞ.

> **Kritik ayrım:** `db2fmcd` (Fault Monitor Coordinator) ≠ DB2 instance (`db2sysc`).
> - `db2fmcd` = DB2'yi izleyen/başlatan daemon
> - `db2sysc` = asıl veritabanı motoru
>
> `db2sysc` boş olması = **veritabanı hiç çalışmıyor**. `db2fmcd` boot'ta ölünce, onun tetiklemesi gereken instance da hiç başlamadı. XMETA erişilemez durumda.

### 2.8 — Instance autostart yapılandırması

```sh
/datastage/IBM/DB2/bin/db2greg -getinstrec instancename='db2inst1'
```

**Çıktının kritik satırları:**
```
InstanceName = db2inst1
InstancePath = /home/db2inst1/sqllib   ← lokal disk
StartAtBoot  = 1                        ← autostart ZATEN açık
InstallPath  = /datastage/IBM/DB2       ← binary'ler SAN'da
```

> **Son parça yerine oturdu:** `StartAtBoot=1` yani autostart zaten doğru yapılandırılmış. Sorun autostart'ın kapalı olması DEĞİL. Sorun, `InstallPath`'in (`/datastage/IBM/DB2`) SAN mount üzerinde olması ve mount gelmeden başlatıcının çalışması. Yani **doğru yapılandırma + yanlış zamanlama**.

---

## 3. Acil Müdahale — Ortamı Ayağa Kaldırma

Teşhis bitti. Önce çalışmayan DB2'yi manuel başlatıp ortamı normale döndürdük.

### 3.1 — DB2 instance'ı başlat

```sh
su - db2inst1 -c "db2start"
ps -ef | grep db2sysc | grep -v grep    # doğrula: db2sysc görünmeli
```

### 3.2 — XMETA erişimini test et

```sh
su - db2inst1 -c "db2 connect to xmeta && db2 connect reset"
```

> **Çıktı yorumu:** `Database server = DB2/LINUXX8664 11.5.9.0`, `Local database alias = XMETA` görünürse bağlantı başarılı.
> Sondaki `SQL1024N A database connection does not exist` **hata değildir** — bu `db2 connect reset` komutunun normal çıktısıdır (bağlantıyı kapattı, "artık bağlantı yok" diyor). Önce bağlandı, sonra temizce çıktı.

### 3.3 — Fault monitor'ü temizle ve başlat

```sh
systemctl reset-failed db2fmcd.service
systemctl start db2fmcd.service
systemctl status db2fmcd.service --no-pager
```

> **Neden `reset-failed`:** systemd, bir servis çok kez fail edince "failed" state'ine kilitler ve "start request repeated too quickly" der. `reset-failed` bu kilidi sıfırlar, böylece tekrar başlatabiliriz. Mount artık geldiği için bu sefer başarılı oldu.

---

## 4. Kalıcı Çözüm — systemd Drop-in Override

### 4.1 — Neden drop-in override?

DB2'nin orijinal unit dosyası (`/etc/systemd/system/db2fmcd.service`) şu uyarıyı içeriyor:

```
# Note: any customizations to this file will be lost the next time this is updated
```

Yani dosyayı doğrudan düzenlersek, DB2 patch/upgrade sırasında değişikliğimiz silinir.

> **Çözüm: Drop-in override.** `/etc/systemd/system/<servis>.service.d/override.conf` dosyası, orijinal unit'i **ezmeden** ek ayar tanımlar. Orijinal dosyaya dokunulmaz, override ayrı durur, upgrade'de orijinal güncellense bile override kalır. Geri alması da temizdir (sadece klasörü sil).

### 4.2 — Mount unit adını teyit et

```sh
systemctl list-units --type=mount | grep -i datastage
# Çıktı: datastage.mount  loaded active mounted /datastage
```

> **Neden:** Override'da mount'a bağımlılık vereceğiz. systemd'nin bu mount'u hangi isimle tanıdığını (`datastage.mount`) kesin bilmemiz gerekti. `/datastage` yolu otomatik olarak `datastage.mount` unit adına çevrilir.

### 4.3 — DB2 fault monitor override'ı

```sh
mkdir -p /etc/systemd/system/db2fmcd.service.d
vi /etc/systemd/system/db2fmcd.service.d/override.conf
```

İçerik:

```ini
[Unit]
After=datastage.mount remote-fs.target
Wants=datastage.mount

[Service]
ExecStartPre=/bin/sh -c 'until [ -x /datastage/IBM/DB2/bin/db2fmcd ]; do sleep 2; done'
```

**Her satırın anlamı:**

| Direktif | Anlamı |
|----------|--------|
| `After=datastage.mount remote-fs.target` | Bu servis, mount ve remote-fs hedefi hazır olduktan **sonra** başlasın (sıralama) |
| `Wants=datastage.mount` | Mount'u **iste** ama zorunlu kılma (yumuşak bağımlılık) |
| `ExecStartPre=...until...` | Asıl `ExecStart`'tan önce çalışır: binary çalıştırılabilir olana kadar bekler (multipath gecikmesine ekstra güvence) |

### 4.4 — WebSphere/ISFServer override'ı

```sh
mkdir -p /etc/systemd/system/ISFServer.service.d
vi /etc/systemd/system/ISFServer.service.d/override.conf
```

İçerik:

```ini
[Unit]
After=datastage.mount db2fmcd.service
Wants=datastage.mount
```

> **Neden:** WebSphere XMETA için DB2'ye muhtaç. `After=db2fmcd.service` ile WebSphere'in DB2 katmanından SONRA başlamasını garantiledik. Böylece WebSphere ayağa kalkarken XMETA erişilebilir olur.

### 4.5 — `Wants` vs `Requires` kararı (Production Güvenliği)

İlk taslakta `Requires=datastage.mount` (sert bağımlılık) düşünüldü, ama production için `Wants` (yumuşak) tercih edildi.

| | `Requires=datastage.mount` | `Wants=datastage.mount` |
|---|---|---|
| Mount başarısız olursa | Servis **HİÇ başlamaz** | Servis yine de başlamayı **dener** |
| Risk | SAN sorununda sistem tamamen kilitlenir | Daha esnek, `ExecStartPre` zaten binary'yi bekler |
| Production önerisi | — | ✅ Tercih edildi |

> **Öğreti:** Production'da sert bağımlılıklar (`Requires`) sistemi beklenmedik şekilde kilitleyebilir. `Wants` + `After` + `ExecStartPre` kombinasyonu hem sıralamayı garantiler hem de tek bir mount sorununda her şeyi durdurmaz.

### 4.6 — Değişiklikleri yükle ve doğrula

```sh
systemctl daemon-reload

systemctl show db2fmcd.service -p After -p Wants | grep -i datastage
systemctl show ISFServer.service -p After -p Wants | grep -i -E "datastage|db2fmcd"
```

> **Neden `daemon-reload`:** systemd, unit dosyalarını bellekte tutar. Yeni override'ı diske yazmak yeterli değil; `daemon-reload` ile systemd'nin yapılandırmayı yeniden okumasını sağlarız.
>
> **Neden `systemctl show`:** Dosyanın var olması, systemd'nin onu doğru okuduğu anlamına gelmez. `show` komutu, systemd'nin **gerçekte ne anladığını** gösterir. Çıktıda `datastage.mount` ve `db2fmcd.service` bağımlılık olarak görünüyorsa, override başarıyla devreye girmiş demektir.

---

## 5. Sonuç — Doğru Boot Zinciri

Override'lardan sonra boot sırası şu hale geldi:

```
datastage.mount (SAN multipath hazır)
        ↓
db2fmcd (DB2 fault monitor) → db2sysc (DB2 instance, StartAtBoot=1)
        ↓
Shared OSS (Zookeeper → Kafka → Solr)
        ↓
ISFServer (WebSphere / server1) → XMETA'ya bağlanır
        ↓
ISFAgents (ASB Agent)
```

Bu, IBM'in resmi başlatma sırasıyla birebir uyumlu: **veritabanı önce hazır olmalı, sonra application server, sonra agent'lar.**

---

## 6. Rollback (Geri Alma) Planı

Bir sorun olursa, override'lar tamamen kaldırılarak sistem eski haline döndürülür. IBM'in orijinal dosyalarına hiç dokunmadığımız için bu temiz bir geri dönüştür:

```sh
rm -rf /etc/systemd/system/db2fmcd.service.d
rm -rf /etc/systemd/system/ISFServer.service.d
systemctl daemon-reload
```

---

## 7. Reboot Sonrası Doğrulama Checklist

Sistem geri geldikten ~2-3 dakika sonra (mount + DB2 + WebSphere zinciri için):

```sh
# 1. Hiçbir servis fail etmemeli — db2fmcd artık burada OLMAMALI
systemctl --failed

# 2. DB2 instance gerçekten geldi mi
ps -ef | grep db2sysc | grep -v grep

# 3. db2fmcd mount sonrası temiz başladı mı
systemctl status db2fmcd.service --no-pager
journalctl -b -u db2fmcd.service --no-pager | tail -15

# 4. WebSphere gerçekten ayakta mı (systemd state'ine değil, prosese bak)
ps -ef | grep server1 | grep -v -e grep -e kafka

# 5. XMETA erişilebilir mi
su - db2inst1 -c "db2 connect to xmeta && db2 connect reset"
```

**Başarı kriteri:** `systemctl --failed` boş + `db2sysc` var + `server1` var + XMETA connect oluyor.

---

## 8. Boot Süresini Ölçme

```sh
systemd-analyze                  # toplam OS boot süresi
systemd-analyze blame | head -20 # hangi servis ne kadar sürdü
```

> **Dikkat — SysV servisleri yanıltıcı olabilir:** `ISFServer` gibi SysV-tabanlı servislerde `systemd-analyze` "10 saniyede bitti" diyebilir, çünkü script `exited` deyip çıkar. Ama arkadaki WebSphere JVM'i dakikalarca daha açılmaya devam eder. systemd o JVM'i takip etmez.
>
> **Gerçek "kullanılabilir" anını ölçmek için:** reboot'tan sonra `ps -ef | grep server1` ile server1 prosesinin göründüğü anı izle. O an = ortamın fiilen kullanılabilir olduğu an.

Tipik beklenen toplam süre (OS boot → WebSphere tam hazır): **~6-12 dakika** (en uzun kalem WebSphere JVM'idir). Bu donanıma, SAN hızına ve XMETA boyutuna göre değişir — kesin değer için yukarıdaki komutlarla ölçülmelidir.

---

## 9. Genel Öğretiler (Özet)

1. **"Autostart çalışmıyor" her zaman autostart sorunu değildir.** Burada autostart (`StartAtBoot=1`) doğruydu; sorun mount-timing'di.

2. **SAN/multipath mount'lar boot'ta geç gelir.** Bu mount üzerindeki binary'leri kullanan servisler, mount bağımlılığı tanımlanmazsa erken çalışıp `203/EXEC` alır.

3. **`status=203/EXEC` = executable çalıştırılamadı.** Sebep: dosya yok / izin yok / noexec. Hepsini tek tek elemek gerekir.

4. **`db2fmcd` ≠ `db2sysc`.** Fault monitor ile instance farklı şeylerdir; doğru olanı kontrol etmek önemli.

5. **Drop-in override > dosyayı düzenlemek.** IBM/vendor dosyalarına dokunmadan, upgrade-güvenli ek ayar yapmanın doğru yolu.

6. **Production'da `Wants` genelde `Requires`'tan güvenlidir.** Sert bağımlılık tüm sistemi kilitleyebilir.

7. **systemd state'ine değil, gerçek prosese güven.** SysV servisleri "active (exited)" görünebilir ama arkadaki gerçek proses ölmüş olabilir. `ps` ile doğrula.

8. **Teşhiste her komutun bir amacı vardır.** "Şunu çalıştır" değil, "şunu görmek için şunu çalıştır, çıktıdan şunu anla" mantığıyla ilerle.

---

*Doküman miatsdst06 üzerinde yapılan gerçek troubleshooting oturumuna dayanmaktadır. Komutlar bu ortama (RHEL 9.6, IS 11.7.1.6, DB2 11.5.9.0, SAN multipath /datastage) özgüdür; başka ortamlara uyarlanırken yol ve isimler kontrol edilmelidir.*
