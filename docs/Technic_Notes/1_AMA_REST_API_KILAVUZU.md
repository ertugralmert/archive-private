# IBM AMA REST API — Kullanım Kılavuzu

> IBM **Application Migration Accelerator (AMA)** REST API'sine CLI ile bağlanma, endpoint'ler ve örnek komutlar.
> SGK Modernizasyon Projesi · Mert Ertuğral · IBM Expert Labs

---

## 1. AMA Nedir?

**AMA = Application Migration Accelerator** — IBM'in WebSphere uygulamalarını (Traditional → Liberty) analiz edip geçiş eforu, uyumluluk sorunları ve önerileri çıkaran aracı. Tüm analiz çıktısı bir REST API üzerinden erişilebilir.

---

## 2. Bağlantı Bilgileri

| Öğe | Değer |
|-----|-------|
| **Base URL** | `https://openapi.ibm-ama-ama.apps.ocpdev.sgk.intra/lands_advisor` |
| **API versiyonu** | `/advisor/v2` |
| **Auth** | OpenShift bearer token, **iki header'da birden** |
| **TLS** | İç `.intra` endpoint, self-signed CA → `-k` (insecure) gerekir |

### Kimlik Doğrulama — Çift Header (önemli!)

AMA gateway, token'ı **iki header'da birden** ister. Biri eksikse istek reddedilir:

```
Authorization: Bearer <token>
apiKey: <token>
```

Token, OpenShift oturum token'ıdır:
```bash
oc whoami -t
```

---

## 3. Hazırlık — Token ve Değişkenler

Her şeyden önce OCP'ye login olun ve değişkenleri ayarlayın:

```bash
# OCP login (token Console'dan: kullanıcı adı → Copy login command)
oc login --token=sha256~XXXX --server=https://api.ocpdev.sgk.intra:6443

# Değişkenler
export TOKEN=$(oc whoami -t)
export SRV=https://openapi.ibm-ama-ama.apps.ocpdev.sgk.intra/lands_advisor
```

> `-sk` = sessiz + insecure TLS. `-s` çıktıyı sadeleştirir, `-k` self-signed sertifikayı kabul eder.

---

## 4. Endpoint'ler ve Örnek Komutlar

### 4.1. Workspace Listesi

Tüm workspace'leri ve hedef tercihlerini (seçili Java SE/EE) listeler.

```bash
curl -sk -H "Authorization: Bearer $TOKEN" -H "apiKey: $TOKEN" \
  "$SRV/advisor/v2/workspaces" | python3 -m json.tool
```

Workspace ID'sini buradan alın:
```bash
curl -sk -H "Authorization: Bearer $TOKEN" -H "apiKey: $TOKEN" \
  "$SRV/advisor/v2/workspaces" | python3 -c "
import sys, json
for w in json.load(sys.stdin).get('workspaces', []):
    print(w.get('id'), '→', w.get('name'))
"
```

Bir workspace ID'sini değişkene koyun:
```bash
export WS=<workspace-id>
```

### 4.2. Cost Details — Efor ve JAR Envanteri

En zengin endpoint. Üç katman içerir: workspace özeti, per-app efor, versiyonlu JAR listesi.

```bash
curl -sk -H "Authorization: Bearer $TOKEN" -H "apiKey: $TOKEN" \
  "$SRV/advisor/v2/workspaces/$WS/costDetails" | python3 -m json.tool
```

**Workspace özeti** (hedef başına toplam efor):
```bash
curl -sk -H "Authorization: Bearer $TOKEN" -H "apiKey: $TOKEN" \
  "$SRV/advisor/v2/workspaces/$WS/costDetails" | python3 -c "
import sys, json
d = json.load(sys.stdin)
for dom in d.get('domains', []):
    for t in dom['recommendations']['summary']['targets']:
        print(f\"{t['targetId']}: total={t['totalCost']} (common={t['commonCodeCost']}, unique={t['uniqueCodeCost']})\")
"
```

**Per-app efor** (her uygulama için gün, complexity, kritik):
```bash
curl -sk -H "Authorization: Bearer $TOKEN" -H "apiKey: $TOKEN" \
  "$SRV/advisor/v2/workspaces/$WS/costDetails" | python3 -c "
import sys, json
d = json.load(sys.stdin)
for dom in d.get('domains', []):
    for au in dom['recommendations'].get('assessmentUnits', []):
        app = au.get('displayProperties', {}).get('application', '?')
        for t in au.get('targets', []):
            s = t.get('summary', {})
            iss = s.get('issues', {})
            print(f\"{app}: {t['totalCost']}d, {s.get('complexity',{}).get('score','')}, critical={iss.get('critical',0)}, {s.get('requiredCodeChanges','')}\")
"
```

### 4.3. Assessment Units — Uygulama Listesi

Workspace'teki uygulamalar (ad, host, middleware, ear/war).

```bash
curl -sk -H "Authorization: Bearer $TOKEN" -H "apiKey: $TOKEN" \
  "$SRV/advisor/v2/workspaces/$WS/assessmentUnits" | python3 -m json.tool
```

Sadece isim + tip:
```bash
curl -sk -H "Authorization: Bearer $TOKEN" -H "apiKey: $TOKEN" \
  "$SRV/advisor/v2/workspaces/$WS/assessmentUnits" | python3 -c "
import sys, json
for a in json.load(sys.stdin).get('assessmentUnits', []):
    print(a.get('applicationName'), '|', a.get('host'), '|', a.get('middleware'))
"
```

### 4.4. Issues — Hedef Başına Migration Kuralları

Belirli bir hedef için tetiklenen kurallar. **Hedef parametresi zorunlu** — her hedef farklı sonuç verir.

```bash
# websphereLiberty hedefi için
curl -sk -H "Authorization: Bearer $TOKEN" -H "apiKey: $TOKEN" \
  "$SRV/advisor/v2/workspaces/$WS/issues?targetPlatform=websphereLiberty" | python3 -m json.tool
```

Geçerli hedefler: `websphereLiberty`, `websphereTraditional`, `managedLiberty`, `openLiberty`, `enterpriseApplicationService`

Kural listesi (ruleId + kaç app):
```bash
curl -sk -H "Authorization: Bearer $TOKEN" -H "apiKey: $TOKEN" \
  "$SRV/advisor/v2/workspaces/$WS/issues?targetPlatform=websphereLiberty" | python3 -c "
import sys, json
for i in json.load(sys.stdin).get('issues', []):
    print(f\"{i['totalApps']:3} app | {i['ruleId']} — {i['ruleName']}\")
"
```

### 4.5. Target Compatibility — Hedef Java Desteği

Her hedefin desteklediği Java SE / Java EE matrisi.

```bash
curl -sk -H "Authorization: Bearer $TOKEN" -H "apiKey: $TOKEN" \
  "$SRV/advisor/v2/targetCompatibility" | python3 -m json.tool
```

> Örnek fark: `websphereLiberty` Java 8/11/17/21 + EE 6–10 desteklerken, `websphereTraditional` sadece Java 8 + EE 7 destekler.

### 4.6. Bulk Export — Tam Workspace ZIP'i

Workspace'in tüm analiz verisini ZIP olarak indirir (`application/octet-stream`).

```bash
curl -sk -H "Authorization: Bearer $TOKEN" -H "apiKey: $TOKEN" \
  "$SRV/advisor/v2/workspaces/$WS/collectionArchives/bulkExport" \
  -o bulkExport.zip
```

Header'ları görmek için (dosya adı vs.):
```bash
curl -skI -H "Authorization: Bearer $TOKEN" -H "apiKey: $TOKEN" \
  "$SRV/advisor/v2/workspaces/$WS/collectionArchives/bulkExport"
# content-disposition: attachment;filename=bulkExport_<workspace>_<ts>.zip
```

---

## 5. Tüm Endpoint Listesini Keşfetme

API'nin OpenAPI spec'ini çekip tüm endpoint'leri görebilirsiniz:

```bash
curl -sk -H "Authorization: Bearer $TOKEN" -H "apiKey: $TOKEN" \
  "$SRV/openapi/" -o ama_spec.yaml

# Tüm path'leri listele (PyYAML gerekir: pip3 install pyyaml)
python3 -c "
import yaml
d = yaml.safe_load(open('ama_spec.yaml'))
for p in sorted(d.get('paths', {})):
    methods = [m.upper() for m in d['paths'][p] if m in ('get','post','put','delete')]
    print(f\"{','.join(methods):10} {p}\")
"
```

---

## 6. Endpoint Özet Tablosu

| Endpoint | Açıklama |
|----------|----------|
| `GET /advisor/v2/workspaces` | Workspace listesi + hedef tercihleri |
| `GET /advisor/v2/workspaces/{id}/costDetails` | Efor, complexity, JAR envanteri |
| `GET /advisor/v2/workspaces/{id}/assessmentUnits` | Uygulama listesi (ad, host, ear/war) |
| `GET /advisor/v2/workspaces/{id}/issues?targetPlatform=X` | Hedef başına migration kuralları |
| `GET /advisor/v2/targetCompatibility` | Hedef → Java SE/EE matrisi |
| `GET /advisor/v2/workspaces/{id}/collectionArchives/bulkExport` | Tam workspace ZIP'i |
| `GET /openapi/` | OpenAPI spec (tüm endpoint'ler) |

---

## 7. Sorun Giderme

| Belirti | Sebep | Çözüm |
|---------|-------|-------|
| `401 Unauthorized` | Token eksik/yanlış veya tek header | İki header'ı da gönder; `oc whoami -t` ile taze token al |
| `SSL certificate problem` | Self-signed CA | `curl -k` kullan |
| Boş yanıt / `404` | Yanlış workspace ID | `/workspaces` ile doğru ID'yi al |
| `Could not resolve host` | `.intra` ağına erişim yok | OCP iç ağında / VPN'de olduğundan emin ol |
| Token süresi doldu | OCP token zaman aşımı | `oc login` ile yeniden al |

---

## 8. Hızlı Başlangıç (Kopyala-Yapıştır)

```bash
# 1. Login + değişkenler
oc login --token=<TOKEN> --server=https://api.ocpdev.sgk.intra:6443
export TOKEN=$(oc whoami -t)
export SRV=https://openapi.ibm-ama-ama.apps.ocpdev.sgk.intra/lands_advisor

# 2. Workspace'leri listele, birini seç
curl -sk -H "Authorization: Bearer $TOKEN" -H "apiKey: $TOKEN" "$SRV/advisor/v2/workspaces" | python3 -m json.tool
export WS=<workspace-id>

# 3. Efor özeti
curl -sk -H "Authorization: Bearer $TOKEN" -H "apiKey: $TOKEN" "$SRV/advisor/v2/workspaces/$WS/costDetails" | python3 -m json.tool

# 4. Uygulamalar
curl -sk -H "Authorization: Bearer $TOKEN" -H "apiKey: $TOKEN" "$SRV/advisor/v2/workspaces/$WS/assessmentUnits" | python3 -m json.tool

# 5. Kurallar (Liberty)
curl -sk -H "Authorization: Bearer $TOKEN" -H "apiKey: $TOKEN" "$SRV/advisor/v2/workspaces/$WS/issues?targetPlatform=websphereLiberty" | python3 -m json.tool
```

---

*Designed & developed by Mert Ertuğral · IBM Expert Labs*
