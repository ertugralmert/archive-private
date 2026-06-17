# "Do Not Use the Same XmlType Name Across Multiple Classes" Bulgusu: Sebep, Mantık ve Çözüm

> WebSphere Application Server Traditional → WebSphere Liberty geçişinde AMA (Application Migration Accelerator) tarafından **Critical** olarak işaretlenen JAXB `@XmlType` çakışmasının uçtan uca açıklaması.

Bu hata ilk bakışta korkutucu görünür ama aslında çok mekaniktir: iki Java sınıfı, XML dünyasında *aynı isme* sahip oluyor ve JAXB "ben aynı ismi iki kez kaydedemem" diyerek patlıyor. Aşağıda önce hatanın *neden* oluştuğunu, sonra **WSDL'e ve client davranışına hiç dokunmadan, minimum kod değişikliğiyle** nasıl çözüleceğini anlatıyorum.

---

## 1. Belirti: Hata Nasıl Görünür?

Kod WAS Traditional'da yıllarca sorunsuz çalışmış olabilir. Ama uygulamayı Liberty'ye taşıyıp başlattığında, JAX-WS istemcisi/uç noktası ilk kez devreye girdiğinde şu tarz bir istisna görürsün:

```
com.sun.xml.bind.v2.runtime.IllegalAnnotationsException:
  Two classes have the same XML type name "evrakDTO".
  Use @XmlType.name and @XmlType.namespace to assign different names to them.
    this problem is related to the following location:
      at tr.gov.sgk.soap.ws.emekli4a.EvrakDTO
      at tr.gov.sgk.soap.ws.emekli4b.EvrakDTO
```

Mesajın kendisi zaten çözümü söylüyor: *"bu iki sınıfa farklı isim ver."* Gerisi bunu güvenli şekilde yapmak.

---

## 2. Önce Temel: JAXB ve `@XmlType` Ne İşe Yarar?

**JAXB**, Java nesnelerini XML'e (ve XML'i Java nesnelerine) çeviren standart mekanizmadır. SOAP web servisleri tam olarak bunu kullanır: bir istek nesnesini XML'e çevirip karşıya yollar, gelen XML'i nesneye geri çevirir.

`@XmlType` annotation'ı ise bir Java sınıfının XML şemasındaki **"complex type"** (karmaşık tip) karşılığını tanımlar. En kritik özelliği `name`:

```java
@XmlType(name = "evrakDTO")
public class EvrakDTO { ... }
```

**Püf nokta:** `name` boş bırakılırsa, JAXB onu **sınıfın adından türetir** (baş harfi küçültür). Yani `@XmlType` yazıp `name` vermezsen, `EvrakDTO` sınıfının XML tip adı otomatik `"evrakDTO"` olur. Bu davranış, çakışmanın asıl kaynağıdır.

---

## 3. Çakışma Neden Oluşuyor? — Mantık

JAXB, çalışırken bir **JAXBContext** kurar. Bu context'in içinde her tip, **(namespace, name)** çiftiyle benzersiz olarak kayıtlıdır. Tıpkı bir sözlükte aynı kelimenin iki kez tanımlanamaması gibi, aynı (namespace, name) çiftine iki farklı sınıf düşemez.

Şimdi senaryoyu kur:

- `tr.gov.sgk.soap.ws.emekli4a.EvrakDTO` → `@XmlType` ismi boş → etkin ad `"evrakDTO"`
- `tr.gov.sgk.soap.ws.emekli4b.EvrakDTO` → `@XmlType` ismi boş → etkin ad yine `"evrakDTO"`
- İkisinin de namespace'i aynı (ya hiç verilmemiş, yani boş namespace).

İki sınıf da aynı **(boş namespace, "evrakDTO")** çiftine düşüyor. Aynı context içine girdiklerinde JAXB "bu ismi iki kez kaydedemem" der ve `IllegalAnnotationsException` fırlatır. Sınıfların *farklı paketlerde* olması Java'da sorun değil, ama JAXB paketi umursamaz — sadece (namespace, name) çiftine bakar.

> Özetle: **Farklı paketlerde, aynı isimli, aynı namespace'te iki sınıf = çakışma.**

---

## 4. Neden WAS Traditional'da Patlamıyor da Liberty'de Patlıyor?

Kod hiç değişmediği hâlde sadece platform değişince hatanın ortaya çıkması kafa karıştırır. Sebep şu:

- **WAS Traditional** kendi web servis/JAXB altyapısını kullanır ve genellikle her servis için ayrı, daha dar kapsamlı bir context kurar ya da bu çakışmalara karşı daha gevşek davranır. Bu yüzden `emekli4a` ve `emekli4b` tipleri çoğu zaman hiç aynı context'te karşılaşmaz.
- **Liberty**, Apache CXF tabanlı JAX-WS implementasyonunu kullanır ve JAXB'yi standartlara daha sıkı uygular. Tipler aynı context'e girdiğinde çakışmayı anında yakalar.

Yani bu bir "kod hatası" değil, **platformlar arası davranış farkı**. AMA da tam bu yüzden bu durumu önceden işaretliyor: WAS'ta sorun çıkarmayan ama Liberty'de patlayacak bir nokta.

---

## 5. Resmi Kaynak Ne Diyor?

AMA'nın bu kural için verdiği resmi açıklama özetle şöyledir: WAS Traditional'dan Liberty'ye geçişte JAX-WS sınıflarındaki **tekrar eden `@XmlType` isimleri runtime istisnasına yol açar**; `@XmlType`'ın `name` özelliği boşsa değer sınıf adına eşitlenir; bu da aynı sınıf adı farklı paketlerde göründüğünde çakışmaya neden olur. Kuralın önerisi nettir: **her `@XmlType` annotation'ı kullanıldığında benzersiz bir `name` belirtmeli.** AMA dokümanı örnek olarak `@XmlType(name="retryEventRequest")` → `@XmlType(name="retryEventRequest2")` dönüşümünü gösterir.

Resmi referans olarak da Java SE 8 JAXB `@XmlType` annotation dokümantasyonuna yönlendirir; orada `name` ve `namespace` özelliklerinin tip kimliğini birlikte belirlediği açıkça yazar.

Bu, çözümün "uydurma bir yama" değil, **doğrudan standardın ve IBM'in resmi tavsiyesinin** olduğu anlamına gelir.

---

## 6. Bu Projedeki Somut Durum

AMA detay raporu, çakışmanın yerini tam olarak veriyor:

- **Nerede:** `soap-webservice-client-4.0.4.jar` içinde (yani uygulamanın kendi `src/main/java`'sında **değil**, bağımlı olduğu kütüphanede).
- **Hangi paketler:** `tr.gov.sgk.soap.ws.emekli4a` ve `tr.gov.sgk.soap.ws.emekli4b`.
- **Hangi sınıflar (ikisinde de aynı isimle var):**
  `ArrayOfEvrakDTO`, `AylikGelirEvrakGoruntule`, `AylikGelirEvrakGoruntuleResponse`, `BarkodluBelgeOlustur`, `BarkodluBelgeOlusturResponse`, `EvrakDTO`, `KesintiBilgileri` — toplam **7 sınıf adı, iki pakette tekrar ediyor.**

Bunlar iki ayrı servisin (emekli4a ve emekli4b) WSDL'lerinden üretilmiş istemci sınıflarıdır. Aynı isimli DTO'lar her iki serviste de bulunduğu ve aynı (boş) namespace'e düştüğü için çakışıyorlar.

> Daha önce `src/main/java` içinde arama yaptığında neden boş geldiğini de bu açıklıyor: sınıflar senin elle yazdığın kodda değil, **kütüphanenin içindeki üretilmiş stub'larda**.

---

## 7. Kritik Soru: WSDL ve Client'a Dokunmadan Nasıl Çözülür?

İşin can alıcı kısmı burası. "İki sınıfa farklı isim ver" deyince akla "ama bu SOAP mesajını bozar mı, server'la uyum kalır mı?" sorusu gelir. Cevap için `@XmlType.name`'in tam olarak neyi etkilediğini bilmek gerekir:

**`@XmlType(name=...)` yalnızca şemadaki *tip adını* (complexType name) değiştirir. Tel üzerinde (SOAP mesajının içinde) görünen *eleman adlarını* değiştirmez.**

Bir SOAP isteği gönderildiğinde, mesajdaki etiketler (`<evrak>`, `<aylikGelir>` gibi) `@XmlElement` / `@XmlRootElement` annotation'larından gelir. `@XmlType.name` ise sadece şemanın `<xsd:complexType name="...">` kısmında geçer ve **örnek mesajda görünmez** (tek istisna: aşağıdaki `xsi:type` notu).

Bunun pratik sonucu şudur — bir istemci sınıfının `@XmlType` adını değiştirdiğinde:

- **WSDL değişmez.** Sen istemci tarafındasın; server'ın WSDL'i ayrı bir yerde duruyor ve ona dokunmuyorsun.
- **SOAP mesajı değişmez.** Gönderilen XML'deki eleman adları aynı kalır, dolayısıyla karşıdaki servis hiçbir farkı görmez. Client sözleşmesi korunur.
- **Sadece JAXB'nin kendi iç haritasındaki çakışma çözülür.**

Bu yüzden `@XmlType.name`'i tekilleştirmek, en **minimal, en güvenli ve resmi olarak önerilen** çözümdür.

> **`xsi:type` istisnası:** Eğer servis kalıtım/polimorfizm kullanıyorsa (DTO'larda nadirdir), tip adı mesaja `xsi:type` ile düşebilir. SGK tarzı düz DTO'larda bu beklenmez; yine de değişiklikten sonra bir uçtan uca çağrı testi yapmak en sağlamı.

---

## 8. Adım Adım Çözüm

**1. Doğru projeyi aç.** Sınıflar `soap-webservice-client` projesinde (emekli-maas-bordro'da değil). Düzeltme orada yapılır.

**2. Çakışan tarafı tekilleştir.** İki paketten **yalnızca birini** değiştir (diğeri olduğu gibi kalsın). Örneğin `emekli4b` paketindeki 7 sınıfta `@XmlType` adına servis sonekini ekle:

```java
// ÖNCE — tr.gov.sgk.soap.ws.emekli4b.EvrakDTO
@XmlType(name = "evrakDTO")          // veya hiç name yok (boş = "evrakDTO")
public class EvrakDTO { ... }

// SONRA
@XmlType(name = "evrakDTO4b")        // emekli4a tarafı "evrakDTO" kalır
public class EvrakDTO { ... }
```

Aynısını diğer 6 sınıf için yap: `arrayOfEvrakDTO4b`, `aylikGelirEvrakGoruntule4b`, `aylikGelirEvrakGoruntuleResponse4b`, `barkodluBelgeOlustur4b`, `barkodluBelgeOlusturResponse4b`, `kesintiBilgileri4b`.

> `2` yerine `4b` gibi anlamlı bir sonek kullanmak, kodu okuyan bir sonraki kişiye "bu hangi servisin tipi" bilgisini de verir — profesyonel ve self-documenting. Toplam değişiklik: 7 sınıfta birer satırlık annotation. Çağrı kodu, iş mantığı, WSDL, hiçbiri değişmez.

**3. Kütüphaneyi yeniden derleyip yerel depoya kur:**

```bash
# soap-webservice-client kökünde:
mvn clean install
```

Bu, düzeltilmiş JAR'ı `.m2`'ne koyar.

**4. Tüketen projeyi yeniden derle.** emekli-maas-bordro artık düzeltilmiş JAR'ı çeker. Yeni WAR'da çakışma yoktur.

---

## 9. Neden "namespace" Yöntemini Tercih Etmiyoruz?

Çakışmayı çözmenin ikinci bir yolu, her pakete `package-info.java` ile **ayrı bir namespace** vermektir (`@XmlSchema(namespace=...)`). Teknik olarak bu da çakışmayı bitirir. **Ama senin şartına uymaz:** namespace değiştirmek, gönderilen SOAP mesajının namespace'ini de değiştirir; karşıdaki servis beklediği namespace'i alamaz ve çağrı kırılır. Yani bu yöntem "WSDL/client davranışı değişmesin" kuralını ihlal eder.

> İstisna: Eğer iki paketin namespace'leri *zaten* farklı olmalıydı ve üreten araç bunu yanlış (boş) üretmişse, her servisin **gerçek WSDL targetNamespace'ini** vermek hem çözer hem de "doğru" olur. Ama bunu yapmak için iki WSDL'i de incelemen gerekir ve yanlış bir namespace çağrıyı bozar. Emin olamadığın sürece `@XmlType.name` yöntemi güvenli tarafta kalmanı sağlar.

---

## 10. Doğrulama

1. **Liberty'ye deploy et ve başlat.** Açılış loglarında (`messages.log`) `IllegalAnnotationsException` **görünmüyorsa**, çakışma çözülmüştür.
2. **Mümkünse bir SOAP çağrısı yap** (gerçek servise erişim yoksa mock ile). Eleman adları değişmediği için istek/yanıt aynen çalışmalı — bu, "client sözleşmesini bozmadık" kanıtıdır.

---

## 11. Önemli Uyarı: Üretilen (Generated) Sınıflar

Bu DTO'lar derleme sırasında WSDL'den **otomatik üretiliyorsa**, elle yaptığın `@XmlType` değişikliği bir sonraki build'de **silinir**. Bu durumda iki sağlıklı yol var:

- **a)** Üretimi tek seferlik yap, üretilen sınıfları repoya **commit'le** ve bir daha üretme (artık "elle bakımlı" kaynak hâline gelirler) → sonra madde 8'i uygula.
- **b)** Bir **JAXB binding dosyası** (`bindings.xml`) ile üretim sırasında tip adını değiştirt; böylece her üretimde isim otomatik tekil gelir.

Eğer sınıflar zaten repoda statik olarak duruyorsa (üretilmiyorsa), hiç dert değil — doğrudan düzenle.

---

## Özet

- Hata, iki sınıfın aynı **(namespace, `@XmlType` name)** çiftine düşmesinden çıkar; `name` boşsa sınıf adına eşitlendiği için farklı paketlerdeki aynı isimli sınıflar çakışır.
- Liberty (CXF/JAXB) bu çakışmayı WAS'tan daha sıkı yakalar; o yüzden kod değişmeden sadece platform değişince ortaya çıkar.
- **Çözüm:** çakışan tarafın `@XmlType` adını tekilleştir (örn. `...4b` soneki). Bu yalnızca şema tip adını değiştirir, SOAP mesajını ve WSDL'i etkilemez → client sözleşmesi korunur. Resmi ve minimal yol budur.
- Düzeltme `soap-webservice-client` kütüphanesinde yapılır → `mvn install` → tüketen proje düzeltilmiş JAR'ı alır.

---

*Hazırlayan: mertugral*
