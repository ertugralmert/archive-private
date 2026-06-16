Haklısın, eksik kalmış. Tek seferde tüm akışı adım adım sıralı olarak yazayım — login dahil hiçbir şey eksiksiz:

🚀 Tam Deploy Rehberi — Hiçbir Adım Eksik Değil

📋 Bütün Akış (Sırayla)

1. Mac'te hazırlık (zip aç + veriyi yükle)
2. (Opsiyonel) Lokal Python ile test
3. OCP'ye login
4. Registry'ye login
5. Image build (--platform linux/amd64)
6. Image push
7. Deployment'a yeni image'ı bağla
8. Rollout'u izle
9. (İlk deploy sonrası bir kez) PVC kur
10. Master panel ile gözlemle


1️⃣ Mac’te Hazırlık

# Çalışma dizinine git
cd /Users/mertugral/Desktop/mertugral-work/01_Active_Projects/SGK/Modernization-Project/SGK-ama-portal

# Eski versiyonu kaldır, yeni v3.5.15 zip'i aç
rm -rf ama-portal-v3
unzip ama-portal-v3.5.15.zip
cd ama-portal-v3

# Workspace zip'lerini koy
cp "../mevcut-workspaces/"*.zip workspaces/

# Migration plan + arch + bağımlılık yükle
python3 src/scaffold_extras.py bulk-import ~/Desktop/mertugral-work/01_Active_Projects/SGK/Modernization-Project/SGK-ama-portal/migration_dosyalar/

python3 src/scaffold_extras.py import-md ~/Desktop/mertugral-work/01_Active_Projects/SGK/Modernization-Project/SGK-ama-portal/mimari-new _arch arch

python3 src/scaffold_extras.py import-bagimlilik ~/Desktop/mertugral-work/01_Active_Projects/SGK/Modernization-Project/SGK-ama-portal/bagimlilik

# Kontrol — klasörler dolu mu?
ls workspaces/ techReports/ migrationPlans/


2️⃣ (Opsiyonel) Lokal Python ile Test

İmage oluşturmadan önce her şey çalışıyor mu kontrol et:

python3 src/build.py --all-zips workspaces --output data --clean

AMA_DATA_DIR=$(pwd)/data AMA_FRONTEND_DIR=$(pwd)/frontend python3 src/app.py
# → http://localhost:8080 → admin (admin/sgk2026) → master (master/mertugral2026)
# Test bittiğinde Ctrl+C


3️⃣ OCP’ye Login

Token’ı OCP Console’dan Al

1. https://console-openshift-console.apps.ocpdev.sgk.intra → Console'a gir
2. Sağ üst köşede kullanıcı adına tıkla
3. "Copy login command" seç
4. "Display Token" tıkla
5. Çıkan "Log in with this token" komutunu kopyala


Komut şuna benzer:

oc login --token=sha256~ABC123DEF456... --server=https://api.ocpdev.sgk.intra:6443


Mac terminal’inde çalıştır:

oc login --token=sha256~ABC123DEF456... --server=https://api.ocpdev.sgk.intra:6443


Çıktı:

Logged into "https://api.ocpdev.sgk.intra:6443" as "o-mertugral" using the token provided.

You have access to N projects, the list has been suppressed. You can list all projects with 'oc projects'

Using project "ama-portal".


Project’i Seç (Eğer Otomatik Seçilmediyse)

oc project ama-portal
# → Now using project "ama-portal" on server "https://api.ocpdev.sgk.intra:6443"


Login Doğrula

oc whoami
# → o-mertugral

oc whoami -t
# → sha256~... (token'ın görünür, kopyalama amaçlı)

oc project
# → Using project "ama-portal" on server "https://api.ocpdev.sgk.intra:6443"


4️⃣ Internal Registry’ye Login

# Registry route URL'ini al (değişkene ata, sonra kullanacağız)
REGISTRY=$(oc get route default-route -n openshift-image-registry --template='{{ .spec.host }}')
echo $REGISTRY
# → default-route-openshift-image-registry.apps.ocpdev.sgk.intra

# Podman ile registry'ye login (oc token'ını kullanır)
podman login -u $(oc whoami) -p $(oc whoami -t) $REGISTRY --tls-verify=false


Çıktı:

Login Succeeded!


❓ Registry route bulunamadı Hatası Alırsan

Internal registry external route’u kapalı olabilir. Cluster-admin yetkisiyle aç:

oc patch configs.imageregistry.operator.openshift.io/cluster \
  --type merge \
  --patch '{"spec":{"managementState":"Managed","defaultRoute":true}}'

# Birkaç saniye bekle, sonra route'u tekrar al
sleep 10
REGISTRY=$(oc get route default-route -n openshift-image-registry --template='{{ .spec.host }}')
echo $REGISTRY


5️⃣ Image Build (–platform linux/amd64)

⚠ KRİTİK: Mac M1/M2 default arm64 build eder. OCP amd64 ister. Unutursan CrashLoopBackOff alırsın.

# Çalışma dizininde olduğundan emin ol
pwd
# → /Users/mertugral/Desktop/.../ama-portal-v3

# Build (15-20 dakika sürer, Mac amd64 emulation yavaş, normal)
podman build --platform linux/amd64 -t ama-portal:v3.5.15 -f Containerfile .


Build adımları sırayla çalışacak:

STEP 1: FROM ubi9/python-311              ✓ Base image
STEP 2-3: pip install                     ✓ Bağımlılıklar
STEP 4-6: COPY src/, frontend/            ✓ Kod
STEP 7-9: COPY workspaces/, techReports/, migrationPlans/  ✓ VERİ
STEP 10: python -m build                  ✓ Data oluşturuluyor (5 dk)
STEP 11: compileall + .py sil             ✓ Bytecode
STEP 12-14: USER, ENV, CMD                ✓ Runtime config


Mimari Doğrula (Push’tan Önce ŞART)

podman inspect ama-portal:v3.5.19 | grep -i architecture


Beklenen:

"Architecture": "amd64"


❌ Eğer arm64 görüyorsan:

podman rmi ama-portal:v3.5.15
podman build --platform linux/amd64 -t ama-portal:v3.5.15 -f Containerfile .
# --platform linux/amd64 doğru yerde olduğundan emin ol


6️⃣ Image Push

# Registry URL ile tag at
podman tag ama-portal:v3.5.19 $REGISTRY/ama-portal/ama-portal:v3.5.19

# Push
podman push $REGISTRY/ama-portal/ama-portal:v3.5.19 --tls-verify=false


Çıktı:

Getting image source signatures
Copying blob ... 
Copying config ... 
Writing manifest to image destination


⏱ 3-5 dakika.

7️⃣ Deployment’ı Yeni Image’a Yönlendir

# ⚠ Burada INTERNAL service URL kullan, external route DEĞİL!
oc set image deployment/ama-portal \
  ama-portal=image-registry.openshift-image-registry.svc:5000/ama-portal/ama-portal:v3.5.19


Çıktı:

deployment.apps/ama-portal image updated


İki URL Farkı (Tekrar Hatırlatma)



|URL                                               |Ne için?                                                  |
|--------------------------------------------------|----------------------------------------------------------|
|`default-route-openshift-image-registry...`       |**Mac’ten push** (external)                               |
|`image-registry.openshift-image-registry.svc:5000`|**Deployment image** (internal, pod cluster içinden çeker)|

8️⃣ Rollout’u İzle

oc rollout status deployment/ama-portal --timeout=5m


Çıktı:

Waiting for deployment "ama-portal" rollout to finish: 1 of 2 updated replicas...
Waiting for deployment "ama-portal" rollout to finish: 2 of 2 updated replicas...
deployment "ama-portal" successfully rolled out


Pod Durumu Kontrol

oc get pods -n ama-portal
# → ama-portal-xxx-yyy   1/1   Running

# Loglara bak
oc logs deployment/ama-portal --tail=20


Portal URL’ini Al

oc get route ama-portal -o jsonpath='{.spec.host}'
echo ""
# → ama-portal-ama-portal.apps.ocpdev.sgk.intra


Tarayıcıdan aç: https://ama-portal-ama-portal.apps.ocpdev.sgk.intra

✅ Portal görünüyor + workspace’ler dolu → ilk deploy başarılı!

9️⃣ PVC Kur (İlk Deploy Sonrası — Tek Sefer)

Portal düzgün çalışıyorsa PVC ekle. Bu komutlar bir kez çalıştırılır.

PVC Oluştur

oc apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ama-portal-persist
  namespace: ama-portal
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 10Gi
EOF


Çıktı:

persistentvolumeclaim/ama-portal-persist created


# PVC bağlandı mı kontrol
oc get pvc -n ama-portal
# → ama-portal-persist   Bound   pv-xxx   10Gi   RWO   <storage-class>


PVC’yi Deployment’a Mount Et

oc set volume deployment/ama-portal --add \
  --name=persist \
  --type=pvc \
  --claim-name=ama-portal-persist \
  --mount-path=/app/persist


Env Var’larla Yönlendir (Tüm Runtime Klasörleri)

oc set env deployment/ama-portal \
  AMA_DATA_DIR=/app/persist/data \
  AMA_WORKSPACES_DIR=/app/persist/workspaces \
  AMA_TECH_REPORTS_DIR=/app/persist/techReports \
  AMA_MIGRATION_PLANS_DIR=/app/persist/migrationPlans \
  AMA_RUNTIME_DIR=/app/persist/data


Rollout İzle (Otomatik Restart)

oc rollout status deployment/ama-portal --timeout=5m


Bootstrap Log’unu Doğrula

oc logs deployment/ama-portal --tail=30 | grep -i bootstrap


Beklenen:

Bootstrap [workspaces]: ✓ tamamlandı
Bootstrap [techReports]: ✓ tamamlandı
Bootstrap [migrationPlans]: ✓ tamamlandı
Bootstrap [data]: ✓ tamamlandı


PVC İçeriğini Kontrol

oc exec deployment/ama-portal -- ls /app/persist
# → data/  workspaces/  techReports/  migrationPlans/

oc exec deployment/ama-portal -- ls /app/persist/workspaces
# → BTGM_EYDB.zip, BTGM_KYDB.zip, ... (image seed'den kopyalandı)


🔟 Master Panel ile Gözlemle

https://ama-portal-ama-portal.apps.ocpdev.sgk.intra/master

Kullanıcı: master
Şifre:    mertugral2026


	•	👥 Portal Ziyaretleri: Kaç ziyaretçi
	•	🌐 Benzersiz IP: Kaç farklı IP
	•	🎯 Hızlı filtre: “Sadece Portal Ziyaretleri” → IP + tarayıcı listesi

📋 Tek Sayfada Komutlar (Kopyala-Yapıştır)

# ═══ 1. HAZIRLIK (Mac'te) ════════════════════════════════════
cd /Users/mertugral/Desktop/mertugral-work/01_Active_Projects/SGK/Modernization-Project/SGK-ama-portal
rm -rf ama-portal-v3
unzip ama-portal-v3.5.15.zip
cd ama-portal-v3

cp "../mevcut-workspaces/"*.zip workspaces/
python3 src/scaffold_extras.py bulk-import ~/Desktop/.../migration_dosyalar/
python3 src/scaffold_extras.py import-md ~/Desktop/.../mimari-new _arch arch
python3 src/scaffold_extras.py import-bagimlilik ~/Desktop/.../bagimlilik


# ═══ 2. OCP LOGIN ════════════════════════════════════════════
# Önce OCP Console → kullanıcı adın → "Copy login command" → Display Token
# Çıkan komutu kopyala ve çalıştır:
oc login --token=<TOKEN_BURAYA> --server=https://api.ocpdev.sgk.intra:6443
oc project ama-portal


# ═══ 3. REGISTRY LOGIN ═══════════════════════════════════════
REGISTRY=$(oc get route default-route -n openshift-image-registry --template='{{ .spec.host }}')
echo "Registry: $REGISTRY"
podman login -u $(oc whoami) -p $(oc whoami -t) $REGISTRY --tls-verify=false


# ═══ 4. BUILD (--platform linux/amd64 ŞART!) ═════════════════
podman build --platform linux/amd64 -t ama-portal:v3.5.15 -f Containerfile .
podman inspect ama-portal:v3.5.15 | grep -i architecture   # amd64 olmalı!


# ═══ 5. PUSH ═════════════════════════════════════════════════
podman tag ama-portal:v3.5.15 $REGISTRY/ama-portal/ama-portal:v3.5.15
podman push $REGISTRY/ama-portal/ama-portal:v3.5.15 --tls-verify=false


# ═══ 6. DEPLOY ═══════════════════════════════════════════════
oc set image deployment/ama-portal \
  ama-portal=image-registry.openshift-image-registry.svc:5000/ama-portal/ama-portal:v3.5.15
oc rollout status deployment/ama-portal --timeout=5m


# ═══ 7. TEST ═════════════════════════════════════════════════
oc get route ama-portal -o jsonpath='{.spec.host}'
echo ""
# → tarayıcıdan aç ve test et


# ═══ 8. PVC EKLE (sadece İLK DEPLOY sonrası tek sefer) ═══════
oc apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata: {name: ama-portal-persist, namespace: ama-portal}
spec:
  accessModes: [ReadWriteOnce]
  resources: {requests: {storage: 10Gi}}
EOF

oc set volume deployment/ama-portal --add --name=persist \
  --type=pvc --claim-name=ama-portal-persist --mount-path=/app/persist

oc set env deployment/ama-portal \
  AMA_DATA_DIR=/app/persist/data \
  AMA_WORKSPACES_DIR=/app/persist/workspaces \
  AMA_TECH_REPORTS_DIR=/app/persist/techReports \
  AMA_MIGRATION_PLANS_DIR=/app/persist/migrationPlans \
  AMA_RUNTIME_DIR=/app/persist/data

oc rollout status deployment/ama-portal --timeout=5m


# ═══ 9. DOĞRULA ══════════════════════════════════════════════
oc logs deployment/ama-portal --tail=30 | grep -i bootstrap
oc exec deployment/ama-portal -- ls /app/persist
oc get route ama-portal -o jsonpath='{.spec.host}'

# https://ama-portal-.../master  → master / mertugral2026


🔄 Sonraki Versiyon Çıkarma (v3.5.16, v3.5.17, …)

İlk deploy + PVC kurulduktan sonra, yeni versiyon çıkarırken sadece 4-7 adımları yap. PVC ve env var’lar zaten kurulu:

# Yeni token gerekirse:
oc login --token=<YENİ_TOKEN> --server=https://api.ocpdev.sgk.intra:6443

# 4-7 adımları aynen tekrar (tag'i değiştirerek):
REGISTRY=$(oc get route default-route -n openshift-image-registry --template='{{ .spec.host }}')
podman login -u $(oc whoami) -p $(oc whoami -t) $REGISTRY --tls-verify=false

podman build --platform linux/amd64 -t ama-portal:v3.5.16 -f Containerfile .
podman tag ama-portal:v3.5.16 $REGISTRY/ama-portal/ama-portal:v3.5.16
podman push $REGISTRY/ama-portal/ama-portal:v3.5.16 --tls-verify=false

oc set image deployment/ama-portal \
  ama-portal=image-registry.openshift-image-registry.svc:5000/ama-portal/ama-portal:v3.5.16
oc rollout status deployment/ama-portal


PVC verisi (hidden, alias, runtime upload’lar, audit log) korunur — sadece kod güncellenir.

↩ Rollback (Geri Dönüş)

# Önceki versiyona
oc rollout undo deployment/ama-portal

# Veya spesifik versiyona
oc set image deployment/ama-portal \
  ama-portal=image-registry.openshift-image-registry.svc:5000/ama-portal/ama-portal:v3.5.14


PVC verisi rollback’ten etkilenmez ✓

  ┌──────────────────────────────────────┐
  │                                      │
  │   🎯 TAM AKIŞ                        │
  │                                      │
  │   İlk deploy: 1→2→3→4→5→6→7→8→9     │
  │                                      │
  │   Sonraki versiyonlar: 3→4→5→6→7→8  │
  │   (1-2 ve PVC tekrar gerekmez)      │
  │                                      │
  └──────────────────────────────────────┘


Bütün adımlar burada — login dahil hiçbir şey eksik değil! 🚀

Bir şey kaldıysa söyle, ama bence şimdi gerçekten eksiksiz bir rehber oldu! 👋



Multi-Attach error for volume "pvc-dcaaf105-e701-4f41-8135-ea9b20691ce8" Volume is already used by pod(s) ama-portal-666575bdfb-zh8xz
