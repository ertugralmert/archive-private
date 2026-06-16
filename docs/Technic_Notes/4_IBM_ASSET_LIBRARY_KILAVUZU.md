# IBM Technical Asset Collection — Form Doldurma Kılavuzu

> IBM topluluğuna varlık (asset) eklerken her alana ne yazılacağı.
> Hem **AMA Live Console** hem **AMA Portal** için hazır içerik.
> Mert Ertuğral · IBM Expert Labs

---

## Genel: Form Bölümleri

IBM "Add Technical Asset Collection" formu dört bölümden oluşur:

1. **Basic Information** — Başlık + açıklama
2. **Asset Resources** — Kaynak tipi + URL (repo linkleri)
3. **Taxonomy Classification** — Kategori + etiketler
4. (Gönder)

Kurallar:
- **Title:** en az 3 kelime (tireli "Ama-Live-Console" tek kelime sayılır!)
- **Description:** 10–100 kelime
- **Resource URL:** IBM domain olmalı (`github.ibm.com`, `ibm.com`, `box.ibm.com`)

---

# 1) AMA LIVE CONSOLE

## Basic Information

### Asset Title (3+ kelime)
```
AMA Live Console for Liberty Migration
```

### Asset Description (10–100 kelime)
```
A web console over the IBM Application Migration Accelerator (AMA) REST API, built for large-scale WebSphere Application Server Traditional to Liberty modernization on OpenShift. It aggregates AMA's deeply-nested analysis (cost, effort, issues, target compatibility) into a navigable interface with KPI dashboards, a searchable findings explorer, and a Migration Workbench for per-application fix planning. Runs in two modes from a single image: live (real-time AMA REST API) or offline (baked-in snapshot, no token). Ships as a ready-to-run container image.
```

## Asset Resources

İki kaynak ekle (Add Resource ile):

**Kaynak 1 — Release imajı:**
| Alan | Değer |
|------|-------|
| Resource Type | Code Asset |
| Description | Ready-to-run container image |
| Resource URL | `https://github.ibm.com/MERTUGRAL/ama-live-console-releases.git` |

**Kaynak 2 — Kaynak kod:**
| Alan | Değer |
|------|-------|
| Resource Type | Code Asset |
| Description | Source code and documentation |
| Resource URL | `https://github.ibm.com/MERTUGRAL/ama-live-console.git` |

## Taxonomy Classification

| Alan | Değer |
|------|-------|
| Primary | Application Development |
| Secondary | Cloud / Containers / Migration (listede hangisi varsa) |
| Custom Tags | `websphere`, `liberty`, `openshift`, `migration`, `ama` |

---

# 2) AMA PORTAL

## Basic Information

### Asset Title (3+ kelime)
```
AMA Portal for WebSphere Modernization
```

### Asset Description (10–100 kelime)
```
A web portal that serves IBM Application Migration Accelerator (AMA) workspace outputs for large-scale WebSphere Application Server Traditional to Liberty modernization on OpenShift. It ingests workspace exports, architecture documents, and migration plans, then presents per-application assessments, estimated development cost, dependency analysis, and technical reports through a single interface. Includes an admin panel and a master observability view (visitor tracking, unique IPs). Packaged as a container image with persistent storage on OpenShift, supporting iterative versioned releases.
```

## Asset Resources

| Alan | Değer |
|------|-------|
| Resource Type | Code Asset |
| Description | Release images and documentation |
| Resource URL | `https://github.ibm.com/ertugralmert/ama-portal-releases.git` |

> Not: ama-portal-releases adresini kendi gerçek repo URL'inle değiştir (yukarıdaki örnek). Eğer `github.com` (public) ise ve form IBM domain istiyorsa, IBM içi bir ayna repo açman gerekebilir.

## Taxonomy Classification

| Alan | Değer |
|------|-------|
| Primary | Application Development |
| Secondary | Cloud / Containers / Migration |
| Custom Tags | `websphere`, `liberty`, `openshift`, `modernization`, `ama` |

---

## Genel İpuçları

1. **Başlık 3 kelime kuralı:** Tireli isimler ("Ama-Live-Console") tek kelime sayılır. Boşluklu yaz: "AMA Live Console for Liberty Migration".

2. **Açıklama kelime sınırı:** 10–100 kelime. Yukarıdaki açıklamalar ~70–80 kelime, güvenli aralıkta.

3. **IBM URL zorunluluğu:** Resource URL `github.ibm.com` gibi IBM domain olmalı. `github.com` (public) kabul edilmeyebilir — o durumda IBM içi repo aç.

4. **Birden fazla kaynak:** "Add Resource" ile hem release (imaj) hem source (kaynak kod) ekleyebilirsin. İkisi de değerli.

5. **Etiketler küçük harf:** Sistem etiketleri otomatik küçük harfe çevirir, sen de küçük yaz.

6. **Taxonomy:** Primary zorunlu (Application Development uygun). Secondary opsiyonel ama bulunabilirliği artırır.

---

## Doldurma Sırası (Checklist)

- [ ] Title — 3+ kelime, boşluklu
- [ ] Description — 10–100 kelime (yukarıdan kopyala)
- [ ] Resource 1 — Code Asset + releases repo URL
- [ ] Resource 2 — Code Asset + source repo URL (opsiyonel)
- [ ] Primary Taxonomy — Application Development
- [ ] Secondary Taxonomy — (opsiyonel)
- [ ] Custom Tags — 5 etiket
- [ ] Gönder

---

*Designed & developed by Mert Ertuğral · IBM Expert Labs*
