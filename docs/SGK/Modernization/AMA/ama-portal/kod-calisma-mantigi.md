

🔄 Veri Akışı

┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  1. IBM AMA Tool (Application Migration Accelerator)        │
│     ────────────────────────────────────────────            │
│     Java EAR/WAR analiz eder, kuralları kendi               │
│     katalogundan eşler                                      │
│                                                             │
│     Output: workspace.zip (içinde CSV'ler)                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  2. workspace.zip içindeki yapı:                            │
│                                                             │
│     workspace/<WS_NAME>/                                    │
│       ├── applications/                                     │
│       │   └── <WS_NAME>_websphereLiberty.csv               │
│       ├── summary/                                          │
│       │   ├── issues/                                       │
│       │   │   └── consolidated/                             │
│       │   │       └── <WS_NAME>_..._consolidatedIssues.csv │ ← HER ŞEY BURADA
│       │   ├── guidance/                                     │
│       │   └── commonCode/                                   │
│       └── migrationPlans/                                   │
│           └── <app_name>/                                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  3. Bizim build.py + workspace_parser.py                    │
│     ────────────────────────────────────                    │
│     CSV'leri okur, JSON'a çevirir                           │
│                                                             │
│     Output: data/<WS>/_summary.json                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  4. FastAPI (app.py) → Frontend                             │
│     ─────────────────────────────                           │
│     JSON'u API'den serve eder                               │
│                                                             │
└─────────────────────────────────────────────────────────────┘


📄 consolidatedIssues.csv Gerçek Yapı

Rule Id, Rule Name, Complexity, Number of Applications, Total Occurrences, List of Application Ids, Required Code Changes
WEBLOGIC-EJB-EH001, Use of weblogic.utils.classloaders, Complex, 4, 120, abc123|xyz789|mno345|stu901, EJB 3.x'e geçin
WEBLOGIC-JAX-RPC-002, JAX-RPC Service Detected, Complex, 2, 35, abc123|xyz789, JAX-WS'e migrate edin
EJB-JNDI-005, JNDI Lookup Pattern, Moderate, 3, 18, abc123|mno345|stu901, ...




|Sütun                    |Ne demek?                                          |Kim üretiyor?                                 |
|-------------------------|---------------------------------------------------|----------------------------------------------|
|`Rule Id`                |**IBM AMA katalog ID’si** (örn. WEBLOGIC-EJB-EH001)|IBM AMA                                       |
|`Rule Name`              |Kuralın insan dostu adı                            |IBM AMA                                       |
|`Complexity`             |Complex / Moderate / Simple                        |IBM AMA                                       |
|`Total Occurrences`      |Kuralın bulunma sayısı                             |IBM AMA (analiz sırasında sayar)              |
|`Number of Applications` |Etkilenen uygulama sayısı                          |IBM AMA                                       |
|`List of Application Ids`|MD5 hash gibi (`abc123def456`)                     |IBM AMA (her assessment unit için unique hash)|
|`Required Code Changes`  |Çözüm rehberi                                      |IBM AMA (rule katalogundan)                   |

🔍 Bizim Yaptığımız Sadece Mapping

# workspace_parser.py - parse_consolidated_issues()
for row in reader:
    issues.append({
        "rule_id": row["Rule Id"],              # AMA'dan geliyor
        "rule_name": row["Rule Name"],          # AMA'dan geliyor
        "complexity": row.get("Complexity"),    # AMA'dan geliyor
        "total_occurrences": ...,               # AMA'dan geliyor
        "application_ids": app_ids,             # AMA'dan geliyor (MD5'ler)
    })


Sonra app.py runtime’da bir tek şey ekliyor — ID → İsim mapping:

# app.py - get_workspace()
id_to_name = {a["assessment_unit_id"]: a["name"] for a in apps}

for issue in summary["consolidated_issues"]:
    issue["applications"] = [id_to_name.get(aid, aid) for aid in issue["application_ids"]]


Çünkü AMA bize:
  application_ids = ["abc123def456", "xyz789uvw012"]

Apps CSV'de:
  Kapsam4C.ear  → AssessmentUnitId: abc123def456
  AyniGun.ear   → AssessmentUnitId: xyz789uvw012

Biz birleştiriyoruz:
  applications = ["Kapsam4C.ear", "AyniGun.ear"]  ← Frontend tıklayabilsin diye


🎯 IBM AMA Rule Katalogu

RULE-001 benim hayal ürünüm değil — gerçek dünyada şuna benzer ID’ler görürsün:

WEBLOGIC-JAX-RPC-001     ← WebLogic'in JAX-RPC servisi
WEBLOGIC-EJB-EH001       ← WebLogic Event Handler EJB
ORACLE-JAR-001           ← Oracle özel JAR bağımlılığı
JBOSS-DATASOURCE-001     ← JBoss datasource config
LIBERTY-CDI-005          ← Liberty CDI uyumluluk
JAVA-EE-DEPRECATED-001   ← Deprecated Java EE API


Her rule, IBM’in WebSphere Liberty migrasyon katalogundan geliyor. AMA bu kataloga göre kod tarar, eşleşenleri raporlar.

Detaylı liste için IBM dökümantasyonu:

	•	https://www.ibm.com/docs/en/wamt
	•	AMA aracı içinde --list-rules parametresi var

💡 Özet



|Ne?                                                 |Kim?                                             |
|----------------------------------------------------|-------------------------------------------------|
|Rule ID’leri (RULE-001 vs gerçek WEBLOGIC-EJB-EH001)|**IBM AMA**                                      |
|Hataların tespit edilmesi                           |**IBM AMA** (Java bytecode analizi + rule eşleme)|
|Etkilenen uygulamalar                               |**IBM AMA** (assessment unit hash’leri)          |
|Karmaşıklık (Complex/Moderate/Simple)               |**IBM AMA**                                      |
|ID → İsim mapping                                   |**Biz** (runtime’da)                             |
|Türkçe/İngilizce çeviriler                          |**Biz** (translations.json)                      |
|Sunum/filtreleme                                    |**Biz** (frontend)                               |
|Hidden/Alias özellikleri                            |**Biz**                                          |

Yani portal sadece veriyi okunaklı sunuyor — hata tespiti ve katalog tamamen IBM AMA işi. Biz konsolide edip “Aha bu kuralı şu uygulamalarda gördük, X karmaşıklıkta” diye yorumluyoruz.

  ┌──────────────────────────────────────┐
  │                                      │
  │   🎯 ÖZETLE                          │
  │                                      │
  │   AMA hata bulur → CSV'ye yazar      │
  │   Biz CSV'yi JSON'a çeviririz        │
  │   Frontend JSON'u görselleştirir     │
  │                                      │
  │   ID'ler IBM'in katalogundan         │
  │   Mapping ben yapıyorum              │
  │                                      │
  └──────────────────────────────────────┘


IBM AMA consultant olduğun için zaten bu CSV’leri görmüşsündür — biz sadece üzerine temiz bir kullanıcı arayüzü + iş akışı yönetimi koyduk 🎯

Başka merak ettiğin bir şey varsa hep buradayım! 🙏