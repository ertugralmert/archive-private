# Container İmaj Oluşturma, Podman ve Release — Kılavuz

> Kod yazdıktan sonra: imaj nasıl yapılır, podman ne işe yarar, OCP'ye nasıl deploy edilir, yeni versiyon nasıl çıkarılır.
> Mert Ertuğral · IBM Expert Labs

---

## 1. Temel Kavramlar

### Container İmajı Nedir?

Bir **container imajı**, uygulamanın çalışması için gereken her şeyi (kod, Python runtime, bağımlılıklar, veri) tek bir paket halinde içeren dosyadır. Bir kez yaparsın, her yerde aynı şekilde çalışır — "benim makinemde çalışıyordu" sorununu ortadan kaldırır.

### Podman Nedir?

**Podman**, container imajı oluşturmak ve çalıştırmak için kullanılan bir araçtır (Docker'a benzer, ama daemon'suz ve root gerektirmez). IBM/Red Hat ortamlarında standart araçtır.

```
Kod + Containerfile  ──podman build──►  İmaj  ──podman push──►  OCP Registry  ──►  Pod (çalışan uygulama)
```

### Containerfile Nedir?

**Containerfile** (Docker'da `Dockerfile`), imajın nasıl oluşturulacağını tarif eden talimat dosyasıdır: hangi base image, hangi dosyalar kopyalanacak, hangi komut çalışacak.

---

## 2. Containerfile Anatomisi

AMA Live Console'un Containerfile'ı, satır satır ne yapıyor:

```dockerfile
# Base image: Red Hat UBI9 + Python 3.11 (kurumsal standart)
FROM registry.access.redhat.com/ubi9/python-311:latest

USER 0                          # root olarak başla (kurulum için)
WORKDIR /app                    # çalışma dizini

COPY requirements.txt .         # bağımlılık listesini kopyala
RUN pip install --no-cache-dir -r requirements.txt   # kur

COPY backend/ ./backend/        # kaynak kodu kopyala
COPY frontend/ ./frontend/      # arayüzü kopyala

# KAYNAK GİZLEME: .py → .pyc derle, .py sil
RUN python3 -m compileall -b -q backend/ \
    && find backend/ -name "*.py" -delete

# OpenShift'in rastgele UID'si için izinler
RUN chgrp -R 0 /app && chmod -R g=u /app

USER 1001                       # root olmayan kullanıcıya geç (güvenlik)

ENV AMA_OFFLINE=1 AMA_PORT=8090 # varsayılan ortam değişkenleri
EXPOSE 8090                     # bu portu aç

WORKDIR /app/backend
CMD ["python3", "-m", "uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8090"]
```

| Talimat | Ne yapar |
|---------|----------|
| `FROM` | Hangi temel imajdan başlanacak |
| `WORKDIR` | Komutların çalışacağı dizin |
| `COPY` | Dosyaları imaja kopyalar |
| `RUN` | Build sırasında komut çalıştırır |
| `ENV` | Ortam değişkeni tanımlar |
| `EXPOSE` | Hangi portun açılacağını belgeler |
| `USER` | Hangi kullanıcıyla çalışılacağı |
| `CMD` | Container başlayınca çalışacak komut |

---

## 3. İmaj Oluşturma (Build)

### Temel Build

```bash
podman build --platform linux/amd64 -t ama-live-console:v1.0 -f Containerfile .
```

| Parça | Anlamı |
|-------|--------|
| `--platform linux/amd64` | **OCP için ZORUNLU.** Mac (arm64) varsayılan arm64 üretir, OCP amd64 ister. Unutursan pod CrashLoopBackOff yer. |
| `-t ama-live-console:v1.0` | İmaja isim:versiyon etiketi verir |
| `-f Containerfile` | Hangi Containerfile kullanılacak |
| `.` | Build context (mevcut dizin) |

### Build Adımlarını Okuma

```
STEP 1/16: FROM ...              ← base image indiriliyor
STEP 6/16: RUN pip install ...   ← bağımlılıklar kuruluyor
STEP 10/16: compileall + .py sil ← kaynak gizleniyor
STEP 14/16: EXPOSE 8090          ← port açılıyor
STEP 16/16: CMD ...              ← çalışma komutu
COMMIT ama-live-console:v1.0     ← imaj tamamlandı ✓
```

### Mimari Doğrulama (push'tan önce ŞART)

```bash
podman inspect ama-live-console:v1.0 | grep -i architecture
# → "Architecture": "amd64"   olmalı
```

arm64 görürsen:
```bash
podman rmi ama-live-console:v1.0
podman build --platform linux/amd64 -t ama-live-console:v1.0 -f Containerfile .
```

---

## 4. İmajı Yerel Çalıştırma (Test)

OCP'ye göndermeden önce yerelde test:

```bash
# Offline modda çalıştır
podman run --rm -p 8090:8090 ama-live-console:v1.0
# → http://127.0.0.1:8090

# Canlı modda çalıştır (ortam değişkeniyle)
podman run --rm -p 8090:8090 \
  -e AMA_OFFLINE=0 \
  -e AMA_SERVER=https://openapi.ibm-ama-ama.apps.ocpdev.sgk.intra/lands_advisor \
  ama-live-console:v1.0
```

| Parça | Anlamı |
|-------|--------|
| `--rm` | Container durunca otomatik sil |
| `-p 8090:8090` | Host portu : container portu (sol Mac, sağ container) |
| `-e VAR=value` | Ortam değişkeni geç |

> **Port mantığı:** `-p 8090:8090` → "Mac'imdeki 8090'ı container'ın 8090'ına bağla". Mac'te başka şey 8090'ı kullanıyorsa sol tarafı değiştir: `-p 9000:8090` → `http://127.0.0.1:9000`.

---

## 5. OCP'ye Push (İç Registry)

```bash
# 1. OCP login
oc login --token=<TOKEN> --server=https://api.ocpdev.sgk.intra:6443
oc project ama-portal

# 2. Registry adresini al
REGISTRY=$(oc get route default-route -n openshift-image-registry --template='{{ .spec.host }}')
echo $REGISTRY

# 3. Registry'ye login
podman login -u $(oc whoami) -p $(oc whoami -t) $REGISTRY --tls-verify=false

# 4. Tag (registry adresiyle etiketle)
podman tag ama-live-console:v1.0 $REGISTRY/ama-portal/ama-live-console:v1.0

# 5. Push
podman push $REGISTRY/ama-portal/ama-live-console:v1.0 --tls-verify=false
```

> **İki registry URL'si farkı:**
> - **Push** → external route: `default-route-openshift-image-registry.apps...` (Mac'ten erişilen)
> - **Deployment image** → internal: `image-registry.openshift-image-registry.svc:5000` (pod cluster içinden çeker)

Registry route kapalıysa (cluster-admin gerekir):
```bash
oc patch configs.imageregistry.operator.openshift.io/cluster \
  --type merge --patch '{"spec":{"managementState":"Managed","defaultRoute":true}}'
sleep 10
REGISTRY=$(oc get route default-route -n openshift-image-registry --template='{{ .spec.host }}')
```

---

## 6. Deploy (OCP'de Çalıştırma)

```bash
# Deployment + Service + Route oluştur
oc apply -f openshift-deploy.yaml

# Image'ı internal registry'den çek
oc set image deployment/ama-live-console \
  ama-live-console=image-registry.openshift-image-registry.svc:5000/ama-portal/ama-live-console:v1.0

# Rollout'u izle
oc rollout status deployment/ama-live-console --timeout=5m

# URL'i al
oc get route ama-live-console -o jsonpath='{.spec.host}'; echo ""
```

### Yararlı Kontrol Komutları

```bash
oc get pods | grep ama-live-console          # pod durumu (1/1 Running olmalı)
oc logs deployment/ama-live-console --tail=30 # loglar
oc exec deployment/ama-live-console -- curl -s localhost:8090/api/health  # health
oc describe pod <pod-adı>                      # detaylı durum (hata ayıklama)
```

---

## 7. Yeni Versiyon Çıkarma

Kod değişti, yeni versiyon (v1.1) çıkaracaksın. Sadece tag'i değiştir:

```bash
# 1. Build (yeni tag)
podman build --platform linux/amd64 -t ama-live-console:v1.1 -f Containerfile .

# 2. Tag + push
podman tag ama-live-console:v1.1 $REGISTRY/ama-portal/ama-live-console:v1.1
podman push $REGISTRY/ama-portal/ama-live-console:v1.1 --tls-verify=false

# 3. Deployment'ı yeni imaja yönlendir
oc set image deployment/ama-live-console \
  ama-live-console=image-registry.openshift-image-registry.svc:5000/ama-portal/ama-live-console:v1.1

# 4. Rollout izle
oc rollout status deployment/ama-live-console
```

### Rollback (Geri Dönüş)

```bash
# Bir önceki versiyona dön
oc rollout undo deployment/ama-live-console

# Belirli bir versiyona dön
oc set image deployment/ama-live-console \
  ama-live-console=image-registry.openshift-image-registry.svc:5000/ama-portal/ama-live-console:v1.0
```

---

## 8. GitHub Release (İmajı Paylaşma)

İmajı `.tar.gz` olarak paketleyip GitHub release'e koymak (kullanıcı build etmeden çalıştırır):

```bash
# 1. İmajı tar'a kaydet
podman save ama-live-console:v1.0 -o ama-live-console-v1.0.tar

# 2. Sıkıştır
gzip ama-live-console-v1.0.tar
# → ama-live-console-v1.0.tar.gz

# 3. Boyutu gör
ls -lh ama-live-console-v1.0.tar.gz
```

Sonra GitHub'da:
```
<repo>/releases/new
  → tag: v1.0
  → title: AMA Live Console v1.0
  → Attach binaries: tar.gz sürükle
  → Publish release
```

Kullanıcı tarafında imajı yükleme:
```bash
gunzip ama-live-console-v1.0.tar.gz
podman load -i ama-live-console-v1.0.tar
podman run -p 8090:8090 ama-live-console:v1.0
```

---

## 9. Faydalı Podman Komutları

```bash
podman images                      # yerel imajları listele
podman ps                          # çalışan container'lar
podman ps -a                       # tüm container'lar (durmuş dahil)
podman logs <container>            # container loglarını gör
podman stop <container>            # durdur
podman rm <container>              # container'ı sil
podman rmi <imaj>                  # imajı sil
podman inspect <imaj>              # imaj detayları (mimari, env, vs.)
podman save <imaj> -o dosya.tar    # imajı dosyaya kaydet
podman load -i dosya.tar           # dosyadan imaj yükle
podman system prune                # kullanılmayan her şeyi temizle
```

---

## 10. Tipik Sorunlar

| Belirti | Sebep | Çözüm |
|---------|-------|-------|
| Pod `CrashLoopBackOff` | arm64 imaj OCP'de | `--platform linux/amd64` ile tekrar build |
| `image not known` (push) | tag adımı atlandı | önce `podman tag ...` |
| `address already in use` | Port dolu | `-p <başka-port>:8090` veya portu boşalt |
| `no such object` (inspect) | İmaj başka makinede | Build edilen makinede çalıştır |
| Registry login başarısız | Route kapalı | `oc patch ... defaultRoute:true` |
| Build çok yavaş (Mac) | amd64 emülasyon | Normal, bekle (15-20 dk olabilir) |

---

## 11. Tam Akış Özeti

```
┌─ KOD DEĞİŞTİ ────────────────────────────────────────┐
│                                                        │
│  podman build --platform linux/amd64 -t app:vX .      │  ← imaj yap
│  podman inspect app:vX | grep -i arch                 │  ← amd64 doğrula
│                                                        │
│  REGISTRY=$(oc get route default-route ...)           │  ← registry al
│  podman login ... $REGISTRY                            │  ← login
│  podman tag app:vX $REGISTRY/proje/app:vX             │  ← etiketle
│  podman push $REGISTRY/proje/app:vX                   │  ← gönder
│                                                        │
│  oc set image deployment/app app=internal-reg/.../vX  │  ← deploy
│  oc rollout status deployment/app                      │  ← izle
│                                                        │
│  (release için: podman save → gzip → GitHub release)  │
└────────────────────────────────────────────────────────┘
```

---

*Designed & developed by Mert Ertuğral · IBM Expert Labs*
