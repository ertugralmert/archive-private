# Maven Bağımlılık Yönetimi: Çözümleme Mantığı ve Kurumsal Nexus Yapılandırması

> WebSphere Application Server Traditional → WebSphere Liberty geçiş çalışmaları için hazırlanmış pratik rehber.

Bir projeyi Git'ten çektin, açtın, ve Eclipse "Missing artifact" diye kırmızı kırmızı bağırıyor. Çoğu zaman sorun kodda değil; Maven'ın o JAR'ı **nereden** arayacağını bilmemesinde. Bu doküman, Maven'ın bir bağımlılığı nasıl bulduğunu adım adım açıklıyor, sonra "ben bunu Nexus'tan çektireyim" dediğinde tam olarak ne yapman gerektiğini gösteriyor.

---

## 1. Temel Soru: Maven Bir JAR'ı Nereden Buluyor?

`pom.xml`'de bir bağımlılık yazdığında, örneğin:

```xml
<dependency>
  <groupId>tr.gov.sgk</groupId>
  <artifactId>soap-webservice-client</artifactId>
  <version>4.0.4</version>
</dependency>
```

Maven sana bu satırın *fiziksel karşılığı* olan `soap-webservice-client-4.0.4.jar` dosyasını bulmak zorunda. Bunu hep aynı sırayla yapar:

1. **Önce yerel depoya bakar** (kendi bilgisayarındaki `.m2` klasörü). Orada varsa işi biter, internete hiç çıkmaz.
2. **Yoksa uzak depolara gider** (remote repository). Bulduğunu indirir ve yerel depoya kopyalar (cache).

Bu iki adımı kafana iyice oturt, çünkü "build neden offline çalıştı", "neden bir makinede çalışıp diğerinde çalışmadı" gibi soruların cevabı hep burada.

---

## 2. Yerel Depo (`.m2/repository`)

Yerel depo, Maven'ın indirdiği her şeyi sakladığı klasördür. Yeri standarttır:

- **Windows:** `C:\Users\<kullanıcı>\.m2\repository`
- **Linux/Mac:** `~/.m2/repository`

İçinde artifact'lar klasör yapısına açılır: `tr/gov/sgk/soap-webservice-client/4.0.4/soap-webservice-client-4.0.4.jar`.

Bir artifact bir kere indiyse, sonraki build'lerde tekrar indirilmez. Bu yüzden bir dependency'i bir defa çözdükten sonra internet olmadan da build alabilirsin. **Tersi de doğru:** eğer bir JAR `.m2`'de yoksa ve Maven onu hiçbir uzak depoda bulamıyorsa, "Missing artifact" hatası alırsın. Hata genelde "kod bozuk" değil, "bu JAR'ı nereden alacağımı bilmiyorum" demektir.

---

## 3. Uzak Depolar Nerede Tanımlı?

Maven uzak depoları üç yerden öğrenir:

1. **Projenin `pom.xml`'indeki `<repositories>` bloğu.** Çoğu kurumsal projede bu blok *yoktur* — bilerek. Çünkü repo adresini her projeye yazmak yerine merkezi bir yerden vermek isterler (bkz. madde 4).
2. **Parent POM.** Senin pom bir parent'tan (örn. `spring-boot-starter-parent`) miras alıyorsa, onun repo tanımları da geçerli olur.
3. **`settings.xml`.** Asıl kurumsal ayar burada yapılır. İki kopyası olabilir:
   - Global: `<MAVEN_KURULUM>/conf/settings.xml`
   - Kullanıcıya özel: `~/.m2/settings.xml` (Windows'ta `C:\Users\<kullanıcı>\.m2\settings.xml`)
   - Kullanıcıya özel olan, global olanı ezer.

**Önemli kural:** Eğer hiçbir yerde uzak depo tanımlı değilse, Maven sadece **tek bir** depoyu bilir: Maven Central. Ve Maven Central'da `tr.gov.sgk.*` gibi kuruma özel artifact'lar **yoktur**. "Missing artifact tr.gov.sgk:..." hatasının en sık sebebi tam budur — Maven sadece Central'ı biliyor, oysa o JAR kurumun Nexus'unda duruyor.

---

## 4. Maven Central ve "Mirror" Mantığı

Varsayılan uzak depo Maven Central'dır. Ama kurumsal ortamlarda (özellikle kapalı ağlarda) durum şöyle işler:

- Geliştirici makineleri çoğu zaman doğrudan internete / Maven Central'a çıkamaz.
- Bunun yerine kurum kendi **Nexus** (veya Artifactory) sunucusunu koyar.
- Nexus iki işi birden yapar: (1) Maven Central'ı **proxy**'ler (yani senin yerine indirir, cache'ler), (2) kuruma özel artifact'ları (`tr.gov.sgk:soap-webservice-client` gibi) **barındırır** (host).
- `settings.xml`'e bir **mirror** koyarsın ve "tüm istekler Nexus'tan geçsin" dersin. İşte sihirli ifade:

```xml
<mirrorOf>*</mirrorOf>
```

Bu, "Maven hangi depoyu isterse istesin, hepsini Nexus'a yönlendir" demektir. Böylece tek bir adresten hem açık kaynak hem kurum-içi JAR'lara erişirsin.

---

## 5. "Benim Projem Şu An Nereden Çekiyor?" — Teşhis

Bir projeyi devraldığında ilk işin, mevcut ayarın ne olduğunu görmek olmalı. Tahmin etme, **gör**. Şu komutlar işini görür (projenin kökünde, `pom.xml`'in olduğu yerde çalıştır):

```bash
# Aktif ayarları (mirror, server kimlikleri, profiller) göster:
mvn help:effective-settings

# Parent ve profiller birleştikten SONRA geçerli olan <repositories>:
mvn help:effective-pom

# Bağımlılık ağacını gör (neyin neyi çektiğini):
mvn dependency:tree

# En ayrıntılısı: her artifact'ın HANGİ depodan çözüldüğünü satır satır yazar:
mvn -X clean compile
```

Bir de gözünle bak: `C:\Users\<kullanıcı>\.m2\settings.xml` dosyasını aç. **Dosya yoksa**, demek ki hiç mirror tanımlı değil — Maven sadece Central'a (ve pom'daki repo'lara) güveniyor. Bu, kapalı ağda neredeyse kesin "Missing artifact" demektir.

---

## 6. Sık Karşılaşılan Hatalar ve Gerçek Anlamları

- **"Missing artifact X" / "Could not find artifact X":** Maven bu JAR'ı ne yerel depoda ne de bildiği uzak depolarda bulamadı. Çözüm: ya doğru uzak depoyu (Nexus) tanıt, ya JAR'ı elle `.m2`'ye kur.
- **"The container 'Maven Dependencies' references non existing library 'C:\...\.m2\...'":** Bu bir Eclipse hatasıdır. Eclipse'in derleme yolu (classpath), `.m2`'de artık var olmayan bir JAR'a işaret ediyor (sen bir artifact'ı sildin/değiştirdin, ama Eclipse eski hâli hatırlıyor). Çözüm: projeye sağ tık → **Maven → Update Project** (force update).
- **"Referenced file contains errors" (örn. bir XSD):** Bu Maven değil, XML doğrulama uyarısıdır; kapalı ağda şema URL'sine erişemediğinden olur. Genelde build'i durdurmaz.

Kısacası, bu hataların büyük kısmı "kod yanlış" değil, "altyapı/erişim ayarı eksik" demektir. Paniğe gerek yok.

---

## 7. Nexus'a Yönlendirme: `settings.xml`

Nexus erişimin varsa, yapman gereken tek şey doğru bir `settings.xml` yazmak. Dosyayı şuraya koy: `C:\Users\<kullanıcı>\.m2\settings.xml`.

```xml
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0">

  <!-- Nexus kimlik bilgileri -->
  <servers>
    <server>
      <id>kurumsal-nexus</id>
      <username>NEXUS_KULLANICI_ADIN</username>
      <password>NEXUS_SIFREN</password>
    </server>
  </servers>

  <!-- Tüm istekleri Nexus'a yönlendir -->
  <mirrors>
    <mirror>
      <id>kurumsal-nexus</id>
      <name>Kurumsal Nexus</name>
      <url>https://NEXUS_HOST/repository/maven-public/</url>
      <mirrorOf>*</mirrorOf>
    </mirror>
  </mirrors>

</settings>
```

Üç noktaya dikkat et, çünkü en sık buralarda hata yapılır:

1. **`mirrorOf>*`** — yıldız, "her şey Nexus'tan geçsin" demek. Nexus zaten Maven Central'ı proxy'lediği için açık kaynak JAR'lar da buradan gelir.
2. **`<server>` içindeki `<id>` ile `<mirror>` içindeki `<id>` AYNI olmalı.** Maven kimlik bilgisini bu id eşleşmesiyle bağlar. Farklı yazarsan kullanıcı/şifre uygulanmaz, "401 Unauthorized" alırsın.
3. **URL'yi tahmin etme, Nexus arayüzünden al.** Nexus'a gir → *Browse* → genelde `maven-public` adında bir **group** repo vardır (proxy + hosted hepsini birleştiren). Onun "Repository Path"/"URL" değeri tam olarak buraya yazılacak adrestir.

Yazdıktan sonra çözümü zorla:

```bash
mvn -U clean install
```

`-U` (update), Maven'ın uzak depoyu yeniden kontrol etmesini zorlar (önbelleğe takılıp kalmaz).

---

## 8. Tek Bir Artifact'ı Test Etmek

Tüm projeyi build etmeden önce, "bu kritik JAR Nexus'ta var mı?" sorusunu tek komutla cevaplayabilirsin:

```bash
mvn dependency:get -Dartifact=tr.gov.sgk:soap-webservice-client:4.0.4
```

- **İniyorsa:** Nexus'ta var, `settings.xml`'in doğru çalışıyor. İşin kolaylaştı.
- **"Could not find artifact" diyorsa:** ya o artifact Nexus'ta hiç yok, ya `settings.xml` Nexus'a bakmıyor. Önce settings'i (madde 7) doğrula; yine bulamıyorsa artifact gerçekten Nexus'ta yok demektir → madde 9.

---

## 9. Artifact Nexus'ta Yoksa Ne Yaparım?

Kuruma özel bazı JAR'lar Nexus'a hiç yüklenmemiş olabilir. O zaman üç seçeneğin var:

**a) Tekil JAR elinde varsa, yerel depoya kur:**

```bash
mvn install:install-file ^
  -Dfile=itext-4.2.1.jar ^
  -DgroupId=tr.gov.sgk ^
  -DartifactId=itext ^
  -Dversion=4.2.1 ^
  -Dpackaging=jar
```

(Aynısını `visolog`, `soap-webservice-client` için tekrarla. Windows'ta satır devamı `^`, Linux/Mac'te `\`.)

**b) JAR'ın kaynak projesi elinde varsa (ayrı bir Maven projesi olarak), onu derleyip kur:**

```bash
# soap-webservice-client kaynağının kökünde:
mvn clean install
```

Bu, projeyi derler ve ürettiği JAR'ı senin yerel `.m2`'ne koyar. Artık ona bağımlı olan diğer proje (örn. emekli-maas-bordro) bu JAR'ı bulur. **Bu yöntem, JAR'ın içindeki bir şeyi düzeltmen gerekiyorsa (örneğin bir annotation hatası) tek yoldur** — çünkü hazır JAR'ı düzeltmek mümkün değil, kaynağı düzeltip yeniden derlemen gerekir.

**c) Kalıcı çözüm:** O artifact'ın sahibi ekipten Nexus'a deploy etmesini iste. Böylece bütün ekip aynı JAR'ı çeker, herkes elle kurmakla uğraşmaz.

---

## 10. Eclipse Tarafı

Komut satırında ayarladığın `settings.xml`'i Eclipse'in de görmesi gerekir:

1. **Window → Preferences → Maven → User Settings** → `settings.xml` yolunu göster → **Reindex / Update Settings**.
2. Projeye sağ tık → **Maven → Update Project** (Alt+F5) → "Force Update of Snapshots/Releases" işaretli → OK.
3. `.m2`'de bir şeyi elle değiştirdiysen (madde 9a/9b), classpath bozulabilir; yine **Update Project (force)** düzeltir.

Eclipse'in "offline mode" ayarının kapalı olduğundan emin ol (Preferences → Maven → "Offline" işaretli olmamalı), yoksa hiçbir indirme yapmaz.

---

## 11. Adım Adım Kontrol Listesi

1. `mvn help:effective-settings` ve `~/.m2/settings.xml` ile mevcut durumu **gör**.
2. Eksik artifact'ı `mvn dependency:get` ile **test et**.
3. Nexus'a bakmıyorsa → `settings.xml`'e **mirror + server** ekle (id'ler aynı, URL Nexus arayüzünden).
4. `mvn -U clean install` ile **zorla**.
5. Hâlâ eksik olan kuruma özel JAR'lar varsa → kaynağını `mvn install` et veya `install:install-file` ile **elle kur**.
6. Eclipse'te **User Settings**'i göster + **Update Project (force)**.
7. Build yeşile dönene kadar 1–6'yı daralt.

---

*Hazırlayan: mertugral*
