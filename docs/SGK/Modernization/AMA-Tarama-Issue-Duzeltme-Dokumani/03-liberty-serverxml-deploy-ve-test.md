# WebSphere Liberty: Sunucu Yapılandırması, Deploy ve Test

> WAS Traditional'dan taşınan bir Java EE 7 (javax) uygulamasını Liberty'de ayağa kaldırıp test etmek için adım adım rehber.

WAS Traditional'a alışkınsan Liberty ilk başta tuhaf gelir: dev gibi bir konsol yok, her şey küçük bir `server.xml` dosyasından dönüyor. Aslında işin güzeli de bu — neyi açtığını bilerek, sadece ihtiyacın olan kadar sunucu kuruyorsun. Bu doküman, sıfırdan bir sunucu oluşturmaktan uygulamayı deploy edip test etmeye kadar tüm akışı anlatıyor.

---

## 1. Liberty Mimarisini Anlamak: `wlp`, `server`, `feature`

Üç kavramı oturt, gerisi kolay:

- **`wlp`** — Liberty'nin kök klasörü (zip'ten çıkan dizin). İçinde `bin/` (komutlar), `usr/` (senin sunucuların), `lib/` (runtime) var.
- **server** — bir sunucu örneği. Her sunucunun kendi `server.xml`'i, kendi logları, kendi uygulamaları vardır. Aynı `wlp` altında birden çok server olabilir. Komut: `server.bat` (Windows) / `server` (Linux).
- **feature** — Liberty modülerdir. WAS gibi "her şeyi yükle" demez; sen `server.xml`'de sadece ihtiyacın olan yetenekleri (servlet, jsp, jaxws, jdbc...) açarsın. Bu yüzden Liberty hafif ve hızlı başlar.

---

## 2. Sunucu Oluşturma

`wlp/bin` altında (Windows PowerShell'de o klasöre girip `.\server.bat` ile çalıştırmak en garantisidir):

```powershell
cd <senin-yolun>\wlp\bin
.\server.bat create sgkServer
```

Bu, `wlp/usr/servers/sgkServer/` klasörünü oluşturur. İçinde:

- `server.xml` — yapılandırma (asıl uğraşacağın dosya)
- `apps/` — kontrollü deploy için uygulama koyduğun yer
- `dropins/` — bırak-çalışsın (otomatik) deploy klasörü
- `logs/` — `messages.log`, `console.log`

---

## 3. `server.xml`'in Anatomisi

Dört temel blok vardır:

- **`<featureManager>`** — hangi yeteneklerin açık olduğu.
- **`<httpEndpoint>`** — hangi port(lar)dan dinlediği.
- **`<webApplication>` / `<application>`** — hangi uygulamanın yüklendiği (dropins kullanırsan bu bloğa gerek yok).
- **`<dataSource>`** — veritabanı bağlantısı (gerekiyorsa).

---

## 4. Doğru Feature Seçimi

Bu uygulamanın profili: **javax tabanlı (Java EE 7)**, içinde SOAP istemcisi (JAX-WS), web arayüzü (Servlet + JSP) ve veritabanı erişimi (JDBC) var. İki yaklaşım mümkün:

**a) Şemsiye feature (en pratik):** `javaee-7.0`. Open Liberty dokümantasyonuna göre bu tek feature, Java EE 7 Full Platform'u oluşturan alt feature'ları bir arada getirir — `servlet-3.1`, `jsp` (webProfile içinden), `jaxws-2.2`, `jaxb-2.2`, `jdbc-4.x`, `jms`, `javaMail` ve diğerleri — ve **Java 8 (JavaSE 1.8) üzerinde çalışır.** Bu sürüm Liberty 26.x dahil destekleniyor. Yani `JAVA_HOME` olarak bir Java 8 JDK kullanman yeterli.

**b) Tek tek minimal feature'lar (daha sıkı kontrol):** sadece gerekenleri açarsın:

```xml
<feature>servlet-3.1</feature>
<feature>jsp-2.3</feature>
<feature>jaxws-2.2</feature>
<feature>jdbc-4.1</feature>
```

Hangisini seçersen seç, **SOAP için `jaxws-2.2` şarttır** — ve dikkat: Liberty'nin JAX-WS'i (CXF tabanlı) JAXB'yi sıkı uyguladığı için, "duplicate XmlType" hatası tam olarak bu feature aktifken ortaya çıkar. (Bkz. XmlType dokümanı.) Open Liberty dokümantasyonu da JAX-WS web uygulamaları için `servlet` ve `jaxws-2.2` feature'larının birlikte açık olması gerektiğini söyler.

Başlangıç için `javaee-7.0`'ı öneririm: tek satır, her şey gelir, sonra istersen daraltırsın.

---

## 5. Tam `server.xml` Örneği

```xml
<server description="sgkServer">

    <featureManager>
        <feature>javaee-7.0</feature>
        <feature>localConnector-1.0</feature>  <!-- server komutlarıyla yönetim için -->
    </featureManager>

    <!-- HTTP / HTTPS dinleme noktaları -->
    <httpEndpoint id="defaultHttpEndpoint"
                  host="*"
                  httpPort="9080"
                  httpsPort="9443" />

    <!-- DB gerekiyorsa (Oracle örneği) — gerekmiyorsa bu bloğu sil -->
    <!--
    <library id="oracleLib">
        <fileset dir="${server.config.dir}/lib" includes="ojdbc8.jar"/>
    </library>
    <dataSource id="sgkDS" jndiName="jdbc/sgkDS">
        <jdbcDriver libraryRef="oracleLib"/>
        <properties.oracle URL="jdbc:oracle:thin:@HOST:1521:SID"
                           user="DB_KULLANICI" password="DB_SIFRE"/>
    </dataSource>
    -->

    <!-- dropins kullanacaksan bu satıra gerek yok -->
    <webApplication location="EmekliMaasBordro.war" contextRoot="/emekli" />

</server>
```

Notlar:
- `host="*"` → sadece localhost değil, dışarıdan da erişilebilir (test için pratik).
- `contextRoot` → uygulamaya hangi yoldan erişeceğin (`/emekli` → `http://localhost:9080/emekli`).
- DB bağlantısı PoC aşamasında genelde gerekmez; gerçek DB'ye erişim yoksa o bloğu kapalı bırak.

---

## 6. Uygulamayı Deploy Etme — İki Yol

**Yol 1 — dropins (en kolay):** Derlenmiş WAR'ı şuraya kopyala:

```
wlp\usr\servers\sgkServer\dropins\EmekliMaasBordro.war
```

Liberty bunu otomatik algılar ve yükler. `server.xml`'de `<webApplication>` satırına gerek kalmaz. Hızlı deneme için ideal.

**Yol 2 — apps + server.xml (kontrollü):** WAR'ı `apps/` altına koy ve `server.xml`'de `<webApplication location="EmekliMaasBordro.war" contextRoot="/emekli"/>` ile tanımla. Context root, güvenlik vb. ayarları açıkça yönetmek istediğinde bunu kullan.

---

## 7. Başlatma ve Logları Okuma

```powershell
cd <senin-yolun>\wlp\bin
.\server.bat start sgkServer
.\server.bat status sgkServer

# Logları canlı izle:
Get-Content ..\usr\servers\sgkServer\logs\messages.log -Wait -Tail 50
```

Loglarda aradığın mesaj kodları:

- **`CWWKF0011I`** — sunucu hazır ("server is ready to run a smarter planet").
- **`CWWKF0012I`** — yüklenen feature'lar listelenir (beklediğin feature'lar var mı kontrol et).
- **`CWWKZ0001I`** — uygulama başladı ("Application ... started").
- **`CWWKT0016I`** — uygulamanın erişilebilir URL'sini yazar.
- **Hata olarak `IllegalAnnotationsException`** görürsen → bu, XmlType çakışmasıdır; uygulama açılamaz. (Çözümü XmlType dokümanında.) Yani Liberty başlatması, aynı zamanda XmlType düzeltmenin **doğrulama testidir.**

Durdurmak için: `.\server.bat stop sgkServer`.

---

## 8. Test Etme

**Web arayüzü:** Tarayıcıda `http://localhost:9080/emekli` (contextRoot neyse). Açılış sayfası geliyorsa servlet/JSP katmanı çalışıyor demektir.

**SOAP tarafı:** Bu uygulama dış servisleri (emekli4a / emekli4b) **çağıran** bir istemci. Dolayısıyla:
- SOAP çağrısını tetikleyen bir ekrana/işleme gir.
- Gerçek servislere ağ erişimi varsa, çağrı baştan sona çalışmalı.
- Erişim yoksa çağrı bir bağlantı hatası verir — **ama bu normaldir**; önemli olan, hatanın `IllegalAnnotationsException` (XmlType) değil, "servise ulaşamadım" tarzı bir hata olması. Yani marshalling/JAXB tarafı düzelmiştir.
- Uygulama kendi bir SOAP uç noktası da sunuyorsa, WSDL'ine `http://localhost:9080/emekli/<servis>?wsdl` ile bak; SoapUI ile örnek istek gönder.

---

## 9. Sık Karşılaşılan Sorunlar

- **"feature ... is not recognized":** O feature bu Liberty sürümünde yok ya da adı yanlış. `javaee-7.0`'ı dene veya feature'ları tek tek aç. Logda hangi feature'ın tanınmadığı yazar.
- **Port çakışması ("port 9080 in use"):** `httpEndpoint`'te portu değiştir (örn. 9081).
- **Veritabanı hatası (NamingException / connection):** `jdbc/sgkDS` tanımlı değilse veya DB'ye erişim yoksa olur. PoC'de DB'ye erişim yoksa bu beklenen bir durumdur.
- **WAR yüklenmiyor:** `messages.log`'a bak; çoğu zaman eksik feature, eksik bağımlılık ya da XmlType gibi bir başlatma istisnasıdır.

---

## 10. Çekirdek Akış — Kontrol Listesi

1. `server create sgkServer`
2. `server.xml`'i düzenle → `javaee-7.0` + `httpEndpoint` (+ gerekiyorsa `dataSource`)
3. WAR'ı `dropins/`'e at (veya `apps/` + `<webApplication>`)
4. `server start sgkServer` → `messages.log`'u izle
5. `CWWKT0016I` ile gelen URL'den eriş, web arayüzünü test et
6. SOAP çağrısını tetikle; `IllegalAnnotationsException` yoksa XmlType düzeltmen de doğrulanmış olur
7. `server stop sgkServer`

---

*Hazırlayan: mertugral*
