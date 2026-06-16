# Git — Bilmen Gereken Komutlar

> Günlük çalışmada işine yarayacak Git komutları, açıklamalı.
> Mert Ertuğral · IBM Expert Labs

---

## 1. Temel Kavramlar

| Kavram | Açıklama |
|--------|----------|
| **Repository (repo)** | Projenin tüm dosyalarını ve geçmişini tutan depo |
| **Commit** | Bir değişiklik kaydı (fotoğraf gibi) |
| **Branch** | Paralel çalışma dalı (`main` ana dal) |
| **Remote** | Uzak repo (GitHub'daki) — genelde `origin` |
| **Push** | Yerel değişiklikleri uzağa gönder |
| **Pull** | Uzaktaki değişiklikleri yerele çek |
| **Stage** | Commit'e dahil edilecek dosyaları işaretle (`git add`) |

```
Çalışma dizini ──git add──► Stage ──git commit──► Yerel repo ──git push──► GitHub (remote)
```

---

## 2. Yeni Repo Başlatma ve İlk Push

```bash
# Klasörde git başlat
cd ~/projem
git init

# Dosyaları stage'le
git add .

# İlk commit
git commit -m "Initial commit"

# Uzak repo'yu bağla (GitHub'da önce boş repo aç)
git remote add origin https://github.ibm.com/KULLANICI/repo-adi.git

# Ana branch'i main yap
git branch -M main

# Push (ilk seferinde -u ile takip ayarla)
git push -u origin main
```

> İlk `-u` sonraki push'larda sadece `git push` yazmanı sağlar.

---

## 3. Günlük Akış (En Çok Kullanılan)

```bash
git status              # neler değişti, neler stage'de — DAİMA önce bak
git add .               # tüm değişiklikleri stage'le
git add dosya.py        # sadece belirli dosyayı stage'le
git commit -m "mesaj"   # commit yap
git push                # GitHub'a gönder
```

Tek satırda sık akış:
```bash
git add . && git commit -m "açıklama" && git push
```

---

## 4. Durum ve Geçmiş Görme

```bash
git status              # mevcut durum (değişen/stage'lenen dosyalar)
git log                 # tüm commit geçmişi
git log --oneline -10   # son 10 commit, tek satır
git log --oneline --graph  # dallanma grafiğiyle
git diff                # stage'lenmemiş değişiklikleri göster
git diff --staged       # stage'lenmiş değişiklikleri göster
git show <commit-id>    # belirli commit'in detayı
```

---

## 5. Remote (Uzak Repo) İşlemleri

```bash
git remote -v                          # bağlı remote'ları göster
git remote add origin <url>            # remote ekle
git remote set-url origin <yeni-url>   # remote URL değiştir
git remote remove origin               # remote sil
```

> **Bu klasör hangi repo'ya bağlı?** → `git remote -v` ile gör. Bir klasör tek remote'a (`origin`) bağlıdır.

---

## 6. Pull (Uzaktan Çekme)

```bash
git pull                # uzaktaki değişiklikleri çek ve birleştir
git fetch               # uzaktakini indir ama birleştirme
git pull origin main    # belirli branch'ten çek
```

> Başka biri repo'ya push yaptıysa, sen push'tan önce `git pull` yapmalısın.

---

## 7. Dosya Geri Alma / Düzeltme

```bash
# Stage'lenmemiş değişikliği geri al (dosyayı son commit'e döndür)
git checkout -- dosya.py

# Stage'den çıkar (ama değişikliği koru)
git restore --staged dosya.py

# Son commit mesajını düzelt
git commit --amend -m "düzeltilmiş mesaj"

# Dosyayı git takibinden çıkar (diskte kalır)
git rm --cached dosya.json

# Son commit'i geri al (değişiklikleri koru)
git reset --soft HEAD~1

# Son commit'i tamamen sil (değişiklikleri de — DİKKATLİ)
git reset --hard HEAD~1
```

> `git rm --cached` çok işe yarar: büyük dosyayı repo'dan çıkarır ama diskte tutar (`.gitignore` ile birlikte).

---

## 8. .gitignore — İstenmeyen Dosyaları Hariç Tutma

Repo'ya gitmesini istemediğin dosyalar (büyük veri, gizli bilgi, derleme çıktıları) için `.gitignore` dosyası oluştur:

```bash
cat > .gitignore << 'EOF'
# Büyük üretilen veri
*.json
backend/drill_index.json

# Container/derleme
*.tar
*.tar.gz
__pycache__/
*.pyc

# OS / gizli
.DS_Store
.env
EOF
```

Zaten takip edilen bir dosyayı sonradan ignore'lamak için:
```bash
git rm --cached dosya.json    # takipten çıkar
# .gitignore'a ekle
git add .gitignore
git commit -m "Add gitignore"
git push
```

---

## 9. Branch İşlemleri

```bash
git branch                  # branch'leri listele
git branch yeni-ozellik     # yeni branch oluştur
git checkout yeni-ozellik   # branch'e geç
git checkout -b yeni-ozellik # oluştur + geç (kısayol)
git merge yeni-ozellik      # branch'i mevcut branch'e birleştir
git branch -d yeni-ozellik  # branch sil
```

---

## 10. GitHub Kimlik Doğrulama (IBM GitHub)

IBM GitHub push isterken kullanıcı adı + **Personal Access Token** (şifre değil) ister.

Token oluşturma:
```
github.ibm.com → sağ üst profil → Settings → Developer settings
  → Personal access tokens → Generate new token
  → repo yetkisini seç → Generate → kopyala
```

Push'ta:
```
Username: <kullanıcı-adın>
Password: <token-buraya-yapıştır>   ← şifre değil, token!
```

Token'ı saklamak için (her seferinde sormaması):
```bash
git config --global credential.helper osxkeychain   # Mac
```

---

## 11. Tag ve Release (Git Tarafı)

```bash
# Tag oluştur
git tag v1.0
git tag -a v1.0 -m "Release v1.0"   # açıklamalı tag

# Tag'i push et
git push origin v1.0
git push --tags                      # tüm tag'leri push et
```

> **Not:** GitHub **Release** (tar.gz, zip ekleme) git ile değil, **web arayüzünden** yapılır. Git sadece tag'i oluşturur; binary dosyalar web'den release'e yüklenir.

---

## 12. Sık Karşılaşılan Durumlar

| Durum | Komut |
|-------|-------|
| "Neler değişti?" | `git status` |
| "Bu klasör hangi repo'ya bağlı?" | `git remote -v` |
| "Son ne yaptım?" | `git log --oneline -5` |
| "nothing to commit" diyor | Değişiklik yok veya `git add` unutuldu |
| "Everything up-to-date" | Push edilecek yeni commit yok |
| Büyük dosya yanlışlıkla gitti | `git rm --cached dosya` + `.gitignore` |
| Push reddedildi (uzak ileride) | önce `git pull`, sonra `git push` |
| Yanlış commit mesajı | `git commit --amend -m "doğru mesaj"` |

---

## 13. Tipik Sorunlar

| Belirti | Sebep | Çözüm |
|---------|-------|-------|
| `nothing to commit, working tree clean` | Değişiklik yok veya `add` yapılmadı | `git status` ile kontrol et |
| `Everything up-to-date` | Yeni commit yok | Önce `commit` yap |
| `rejected ... fetch first` | Uzak senden ileride | `git pull` sonra `git push` |
| `Authentication failed` | Şifre kullanıldı | Token kullan (PAT) |
| `quote>` takılması (zsh) | Kopyala-yapıştırda tırnak hatası | `Ctrl+C` ile çık, komutu tek tek yaz |
| Yanlış remote'a push | Klasör başka repo'ya bağlı | `git remote -v` ile kontrol et |

---

## 14. Güvenli Çalışma İpuçları

1. **Her zaman önce `git status`** — ne yaptığını gör.
2. **Anlamlı commit mesajları** — "fix", "update" değil, "Add offline mode toggle".
3. **Sık commit, küçük commit** — büyük tek commit yerine.
4. **`.gitignore` önce** — repo'yu kirletmeden başla.
5. **Gizli bilgi commit'leme** — token, şifre, `.env`. (IBM GitHub `detect-secrets` ile uyarır.)
6. **Push'tan önce `git pull`** — başkası değişiklik yaptıysa.

---

*Designed & developed by Mert Ertuğral · IBM Expert Labs*
