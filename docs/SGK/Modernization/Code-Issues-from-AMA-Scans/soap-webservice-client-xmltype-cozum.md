# Uygulama Planı ve Doğrulama Rehberi (Runbook)

> emekli-maas-bordro uygulamasını WebSphere Liberty'ye taşırken yaptığımız işin **uçtan uca planı**. Her adımda üç şey var: *ne yapıyoruz*, *neden* (mantık), ve *nasıl doğrularız* (kontrol noktası). Detaylar için: `01-maven`, `02-xmltype`, `03-liberty` dokümanları.

Bu doküman, "şu an neredeyim, doğru mu gidiyorum?" sorusunu her adımda cevaplayabilmen için yazıldı. Komutu çalıştır, **beklenen sinyali** gör, bir sonrakine geç. Sinyali görmüyorsan o adımda takılmışsındır; en alttaki sorun-giderme tablosuna bak.

---

## Büyük Resim: Neyi, Neden Yapıyoruz?

Önce zihinsel modeli oturt, gerisi mekanik:

- Ortada **iki ayrı proje** var:
  - `soap-webservice-client` → bir **kütüphane**. Hatalı sınıflar (`@XmlType` çakışması) **burada**.
  - `emekli-maas-bordro` → asıl **uygulama**. Kütüphaneyi sadece **kullanıyor**.
- Hata uygulamada değil, kütüphanede. O yüzden düzeltmeyi kütüphanede yaparız.
- **Teslim noktası = local `.m2`.** Kütüphaneyi düzeltip `mvn install` ile `.m2`'ye koyarsın; uygulama da bağımlılığını oradan alır. İki proje birbirine işte bu `.m2` üzerinden bağlanır.

Akış tek bakışta:

```
1. Nexus (settings.xml)        → bağımlılıklar çözülebilir hâle gelir
2. soap-client: mvn install    → kütüphane derlenir, JAR .m2'ye düşer
3. DTO'lar kaynak mı? (kontrol)→ düzeltme yöntemini belirler
4. @XmlType düzelt (emekli4b)  → çakışmanın kaynağı temizlenir
5. soap-client: mvn install    → DÜZELTİLMİŞ JAR .m2'ye gider
6. emekli-maas-bordro: build   → WAR üretilir (düzeltilmiş JAR ile)
7. Liberty: deploy + start     → IllegalAnnotationsException YOK = başarı
```

---

## Adım 1 — Maven'ı Nexus'a Bağla

**Ne yapıyoruz:** `~/.m2/settings.xml`'e Nexus mirror + kimlik bilgisi ekliyoruz (bkz. `01-maven`).

**Neden:** İki proje de "Missing artifact" veriyor çünkü Maven, kuruma özel JAR'ları (soap-webservice-client, itext, visolog) ve hatta soap-client'ın kendi test/lombok bağımlılıklarını nereden alacağını bilmiyor. Nexus bunların hepsini tek adresten verir. Bu adım, sonraki her şeyin önkoşulu.

**Nasıl doğrularım:**
```powershell
mvn help:effective-settings        # çıktıda <mirror> bloğu görünmeli
mvn dependency:get -Dartifact=tr.gov.sgk:soap-webservice-client:4.0.4
```
- İkinci komut **BUILD SUCCESS** verip JAR'ı indiriyorsa → Nexus erişimin ve ayarın çalışıyor.
- Eclipse'te: User Settings'i göster → **Maven → Update Project (force)** → kırmızı "Missing artifact" hatalarının kaybolması gerekir.

> ✅ Geçtim sinyali: `dependency:get` BUILD SUCCESS **ve** Eclipse'te Missing artifact hataları gitti.

---

## Adım 2 — Kütüphaneyi (soap-webservice-client) Derle

**Ne yapıyoruz:** Kütüphane projesinin kökünde:
```powershell
mvn clean install -Dmaven.test.skip=true
```

**Neden:** Düzelteceğimiz sınıflar bu projede; ama önce projenin **temiz derlendiğini** görmeliyiz ki düzeltmeden sonra "acaba benim değişikliğim mi bozdu" diye kafa karışmasın. `-Dmaven.test.skip=true` test sınıflarını hiç derlemez — ekrandaki junit/spring-test hataları test tarafında ve bizi ilgilendirmiyor. WSProperty getter'ları Lombok'tan geliyorsa, **komut satırı Lombok'u çalıştırır**; Eclipse'in kırmızısı sadece editör (IDE agent) sorunudur, `mvn` build'ini durdurmaz.

**Nasıl doğrularım:**
```powershell
# Çıktının sonunda:  BUILD SUCCESS
# JAR gerçekten oluştu mu / tarihi yeni mi:
Get-Item "$env:USERPROFILE\.m2\repository\tr\gov\sgk\soap-webservice-client\4.0.4\soap-webservice-client-4.0.4.jar" |
  Select Name, LastWriteTime
```

> ✅ Geçtim sinyali: **BUILD SUCCESS** ve `.m2`'deki JAR'ın `LastWriteTime`'ı az önceki saat.
>
> ⚠️ BUILD FAILURE alıyorsan: WSProperty Lombok kullanmıyorsa gerçek bir kaynak eksiği olabilir — hatayı paylaş, beraber bakarız.

---

## Adım 3 — Düzeltilecek Sınıflar Kaynak mı, Üretilmiş mi?

**Ne yapıyoruz:** soap-webservice-client içinde şu dosyaya bakıyoruz:
`src/main/java/tr/gov/sgk/soap/ws/emekli4b/EvrakDTO.java` (ve diğer 6 DTO).

**Neden:** Bu, düzeltme **yöntemini** belirleyen kritik kontrol. Sınıflar elle yazılmış/commit'lenmiş kaynaksa doğrudan düzenleriz. Ama build sırasında WSDL'den **otomatik üretiliyorlarsa**, elle yaptığımız değişiklik bir sonraki derlemede silinir — o yüzden önce hangisi olduğunu bilmemiz şart.

**Nasıl doğrularım:**
- `emekli4b` paketinde 7 DTO `.java` dosyası olarak **açılıyorsa** → kaynak. Adım 4'e geç.
- Pakette sadece `Emekli4BClient.java` var, DTO'lar **yoksa** (yalnızca `target/` altında üretiliyorsa) → üretilmiş. Bu durumda binding yöntemine geçeriz (`02-xmltype` §11).

> ✅ Geçtim sinyali: 7 DTO dosyasını Eclipse'te açıp içindeki `@XmlType`'ı görebiliyorum.

---

## Adım 4 — XmlType Çakışmasını Düzelt

**Ne yapıyoruz:** `emekli4b` paketindeki **7 sınıfta** `@XmlType` adını tekilleştiriyoruz (emekli4a tarafı **olduğu gibi kalır**):
```java
// tr.gov.sgk.soap.ws.emekli4b.EvrakDTO
@XmlType(name = "evrakDTO4b")   // önceden: name yok veya "evrakDTO"
public class EvrakDTO { ... }
```
Aynısını `arrayOfEvrakDTO4b`, `aylikGelirEvrakGoruntule4b`, `aylikGelirEvrakGoruntuleResponse4b`, `barkodluBelgeOlustur4b`, `barkodluBelgeOlusturResponse4b`, `kesintiBilgileri4b` için yap.

**Neden:** İki sınıf (emekli4a ve emekli4b'deki aynı isimliler) aynı **(namespace, XML tip adı)** çiftine düşüyor; Liberty'nin sıkı JAXB'si bunu çakışma sayıp patlıyor. Tek tarafa farklı `name` verince çift benzersizleşir, çakışma biter. `@XmlType.name` yalnızca **şema tip adını** değiştirir, SOAP mesajındaki **eleman adlarını değil** → karşıdaki servise giden istek değişmez, WSDL'e dokunulmaz (bkz. `02-xmltype` §7).

**Nasıl doğrularım:**
- Her 7 sınıfta `@XmlType(name="...4b")` **açıkça** yazıyor.
- `import javax.xml.bind.annotation.XmlType;` mevcut.
- Eclipse'te bu dosyalarda yeni derleme hatası **yok**.

> ✅ Geçtim sinyali: 7 sınıf da tekil `name` aldı, derleme temiz.

---

## Adım 5 — Kütüphaneyi Yeniden Derle ve Kur

**Ne yapıyoruz:** Tekrar:
```powershell
mvn clean install -Dmaven.test.skip=true
```

**Neden:** Düzeltilmiş **kaynaktan** yeni bir JAR üretip `.m2`'ye koymak. Uygulamanın çekeceği yer burası. Bu adım atlanırsa, sen kodu düzeltsen bile `.m2`'deki JAR hâlâ eski/bozuk olur ve uygulama bozuk olanı alır.

**Nasıl doğrularım:**
- **BUILD SUCCESS.**
- `.m2`'deki JAR'ın `LastWriteTime`'ı **yine güncellendi** (Adım 2'deki komutla bak).
- (Opsiyonel/ileri seviye) JAR'ın içindeki düzeltmeyi kanıtlamak istersen:
  ```powershell
  # JAR'ı geçici bir yere aç, sonra:
  javap -v tr.gov.sgk.soap.ws.emekli4b.EvrakDTO | Select-String "XmlType|evrakDTO4b"
  ```
  `name=evrakDTO4b` görünüyorsa düzeltme JAR'a işlemiş demektir.

> ✅ Geçtim sinyali: BUILD SUCCESS + JAR tarihi yeni (+ istersen javap teyidi).

---

## Adım 6 — Uygulamayı (emekli-maas-bordro) Derle

**Ne yapıyoruz:** emekli-maas-bordro'da **Update Project (force)** yapıp derliyoruz (Eclipse build ya da `mvn clean package`).

**Neden:** Artık local `.m2`'de düzeltilmiş soap-client var. Maven önce `.m2`'ye baktığı için uygulama bu düzeltilmiş JAR'ı çeker (Nexus'taki eski/bozuk sürümü ezer). Sonuç: içinde düzeltilmiş kütüphane olan bir WAR.

**Nasıl doğrularım:**
- Eclipse'te "Missing artifact soap-webservice-client" hatası **gitti**.
- WAR üretildi:
  ```powershell
  Get-ChildItem target\*.war
  ```

> ✅ Geçtim sinyali: Missing artifact yok + `target/` altında `.war` dosyası var.

---

## Adım 7 — Liberty'ye Deploy Et ve Asıl Testi Yap

**Ne yapıyoruz:** WAR'ı `dropins/`'e atıp sunucuyu başlatıyoruz, logları izliyoruz (bkz. `03-liberty`).
```powershell
copy target\EmekliMaasBordro*.war <wlp>\usr\servers\sgkServer\dropins\
cd <wlp>\bin
.\server.bat start sgkServer
Get-Content ..\usr\servers\sgkServer\logs\messages.log -Wait -Tail 50
```

**Neden:** Bu, tüm işin **asıl doğrulaması**. Liberty'nin sıkı JAXB'si, düzeltme tutmamışsa açılışta çakışmayı yakalar; yakalamıyorsa düzeltme başarılı demektir. Yani Liberty başlatması, XmlType fix'inin gerçek sınavıdır.

**Nasıl doğrularım:** Loglarda —
- ❌ `IllegalAnnotationsException` / "same XML type name" **GÖRÜNMEMELİ**.
- ✅ `CWWKZ0001I` → uygulama başladı.
- ✅ `CWWKT0016I` → uygulamanın URL'sini yazar; o URL'den (`http://localhost:9080/<contextRoot>`) uygulama açılıyor.

> ✅ Geçtim sinyali: `IllegalAnnotationsException` yok, `CWWKZ0001I` var, URL açılıyor. **İş tamam.**

---

## Doğrulama Özeti (Hangi Adımda Ne Görmeliyim)

| Adım | Yaptığım | Doğru gittiğinin sinyali |
|------|----------|--------------------------|
| 1 | settings.xml → Nexus | `dependency:get` BUILD SUCCESS; Eclipse'te Missing artifact gitti |
| 2 | soap-client `mvn install` | BUILD SUCCESS; `.m2`'de JAR oluştu |
| 3 | DTO kontrolü | 7 DTO `.java` olarak açılıyor (veya: üretiliyor) |
| 4 | `@XmlType` düzelt | 7 sınıfta tekil `name`; derleme temiz |
| 5 | soap-client tekrar `install` | BUILD SUCCESS; JAR tarihi yeni |
| 6 | emekli-maas-bordro build | Missing artifact yok; `target/*.war` var |
| 7 | Liberty deploy + start | `IllegalAnnotationsException` yok; `CWWKZ0001I` + URL |

---

## Bir Şey Ters Giderse (Sorun → Muhtemel Adım → Çözüm)

| Belirti | Hangi adım | Ne yap |
|---------|-----------|--------|
| "Missing artifact" sürüyor | 1 | settings.xml mirror/id yanlış ya da Update Project yapılmadı |
| soap-client BUILD FAILURE | 2 | `-Dmaven.test.skip=true` eksik; ya da WSProperty gerçekten Lombok değil |
| emekli4b'de DTO yok | 3 | Sınıflar üretiliyor → binding yöntemi (`02-xmltype` §11) |
| WAR'da hâlâ Missing soap-client | 6 | Adım 5 (`mvn install`) atlandı veya Update Project yapılmadı |
| Liberty'de `IllegalAnnotationsException` | 7 | Düzeltme JAR'a yansımadı (Adım 5) ya da eski WAR yüklü — dropins'i temizleyip yeni WAR'ı at |
| Liberty'de "feature not recognized" | 7 | server.xml'de `javaee-7.0` kullan ya da feature'ı düzelt (`03-liberty` §9) |

---

## Mantığın Özü (Aklında Kalsın)

1. Hata **kütüphanede** (soap-webservice-client), uygulamada değil. Düzeltme orada yapılır.
2. İki projeyi birbirine **local `.m2`** bağlar: kütüphaneyi `mvn install` ile oraya koyarsın, uygulama oradan alır.
3. Çakışma, iki sınıfın aynı XML tip adına düşmesinden çıkar; **tek tarafa farklı `@XmlType name`** verince biter ve **SOAP/WSDL hiç değişmez**.
4. Her şeyin **gerçek kanıtı**, Liberty'nin `IllegalAnnotationsException` vermeden açılmasıdır.

---

*Hazırlayan: mertugral*
