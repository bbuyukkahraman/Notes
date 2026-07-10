# Skills — AI Ajanları İçin Yetenek Paketleme Rehberi

> **Skills, AI ajanlarına özel görevleri yerine getirme yeteneği kazandıran, yeniden kullanılabilir talimat ve kaynak paketleridir.**

Bu rehber, skills kavramını sıfırdan öğretir: nedir, neden gereklidir, nasıl çalışır, nasıl yazılır ve nasıl kullanılır. Her bölümde konuyu derinlemesine anlamanızı sağlayacak açıklamalar, kod örnekleri, diyagramlar ve alıştırmalar bulacaksınız.

---

## İçindekiler

1. [Giriş: Skills Nedir?](#1-giriş-skills-nedir)
2. [Neden Skills? — Problemin Tanımı](#2-neden-skills--problemin-tanımı)
3. [Skills Mimarisi: Progressive Disclosure](#3-skills-mimarisi-progressive-disclosure)
4. [SKILL.md Formatı — Frontmatter](#4-skillmd-formatı--frontmatter)
5. [SKILL.md Formatı — Gövde (Body)](#5-skillmd-formatı--gövde-body)
6. [Skills Dizin Yapısı](#6-skills-dizin-yapısı)
7. [Keşif (Discovery): Ajan Skills'i Nasıl Bulur?](#7-keşif-discovery-ajan-skillsi-nasıl-bulur)
8. [Aktivasyon: Skills Nasıl Yüklenir ve Çalıştırılır](#8-aktivasyon-skills-nasıl-yüklenir-ve-çalıştırılır)
9. [Skills'in Yaşam Döngüsü](#9-skillsin-yaşam-döngüsü)
10. [Skills vs Tools vs Plugins vs Prompts — Karşılaştırma](#10-skills-vs-tools-vs-plugins-vs-prompts--karşılaştırma)
11. [Skills Yazma En İyi Uygulamaları](#11-skills-yazma-en-iyi-uygulamaları)
12. [Skills Çalışma Zamanı: Scriptler ve Kaynaklar](#12-skills-çalışma-zamanı-scriptler-ve-kaynaklar)
13. [Adım Adım: İlk Skill'inizi Yazın](#13-adım-adım-ilk-skillinizi-yazın)
14. [Skills Doğrulama ve Test](#14-skills-doğrulama-ve-test)
15. [İleri Seviye: Skill Kompozisyonu ve Çoklu Adım](#15-i̇leri-seviye-skill-kompozisyonu-ve-çoklu-adım)
16. [Platformlar Arası Skills (Pi, Claude, Codex)](#16-platformlar-arası-skills-pi-claude-codex)
17. [Skills Depoları ve Topluluk](#17-skills-depoları-ve-topluluk)
18. [Sık Yapılan Hatalar](#18-sık-yapılan-hatalar)
19. [Alıştırmalar](#19-alıştırmalar)
20. [Hızlı Başvuru Kartı](#20-hızlı-başvuru-kartı)

---

## 1. Giriş: Skills Nedir?

Bir **skill**, bir AI ajanına (pi, Claude Code, Codex gibi) belirli bir görevi nasıl yapacağını öğreten, markdown formatında paketlenmiş talimatlar bütünüdür.

### Somut Benzetme

| Gerçek Dünya | AI Dünyası |
|:---|:---|
| Bir aşçının yemek tarifi defteri | Skills dizini |
| Tek bir tarif (malzemeler + adımlar + ipuçları) | Bir skill (SKILL.md + scriptler + referanslar) |
| Tarifi okumak için defteri açmak | Ajanın SKILL.md'i `read` ile yüklemesi |
| Tarifteki "çırpma teli kullan" talimatı | Skill içindeki bir araç çağrısı |
| Tarifin altındaki "püf noktaları" notu | Skill içindeki `references/` dosyası |

### Basit Bir Skill Örneği

```
pdf-extract/
├── SKILL.md              # Talimatlar
├── scripts/
│   └── extract.py        # Çalıştırılabilir kod
└── references/
    └── form-fields.md    # Detaylı referans
```

```markdown
---
name: pdf-extract
description: Extracts text and tables from PDF files, fills PDF forms.
  Use when user mentions PDFs, forms, or document extraction.
---

# PDF Extraction

## Setup
```bash
pip install pymupdf pandas
```

## Usage
Run the extraction script:
```bash
scripts/extract.py <input.pdf> <output-dir>
```

## Workflow
1. Read the PDF with pymupdf
2. Extract text page by page
3. Detect and extract tables
4. Save results as markdown files
```

### Skills'in Temel Özellikleri

| Özellik | Açıklama |
|:---------|:----------|
| **Kendi kendine yeten (self-contained)** | Tüm talimatlar, kod ve kaynaklar skill dizini içinde |
| **İsteğe bağlı yüklenir (lazy loading)** | Sadece ihtiyaç duyulduğunda tam içerik yüklenir |
| **Platform-bağımsız** | Agent Skills standardı sayesinde herhangi bir uyumlu ajanda çalışır |
| **Sürüm kontrolü dostu** | Git ile yönetilebilir, paylaşılabilir |
| **Genişletilebilir** | Scriptler, referanslar, şablonlar eklenebilir |

---

## 2. Neden Skills? — Problemin Tanımı

### Problem: Ajanlar Ne Yapacağını Bilmez

Bir AI ajanı (ör: pi) size kod yazmada yardımcı olabilir ancak:

- PDF'den form alanı nasıl çıkarılır? **Bilmez** — eğitim verisinde bu spesifik workflow yoktur
- Şirketinizin commit mesajı formatı nedir? **Bilmez** — bu özel bilgidir
- Bir web sayfasını markdown'a nasıl çevirirsiniz? **Bilmez** — hangi aracı kullanacağını bilmez

### Çözüm: Skills

Skills, ajana **bağlam (context)** sağlar:

```
Skillsiz Ajan:       Skills'li Ajan:
  ↓                       ↓
"PDF'i işle"           "PDF'i işle"
  ↓                       ↓
Ne yapacağını bilmez    Skill aktif: "pdf-extract"
  ↓                       ↓
Hata verir              scripts/extract.py dosyasını çalıştırır
                        Sonucu döndürür
```

### Problem: Token Maliyeti

Her konuşma başında ajanın tüm bilgiyi sistem prompt'unda taşıması **çok pahalıdır**.

```
Sistem prompt'una 20 farklı workflow talimatı koymak:
  → Her konuşma için ~50.000 token sabit maliyet
  → Kullanıcı sadece 1 workflow kullanıyor olsa bile
```

**Skills çözümü — Progressive Disclosure:**

| Aşama | Ne Yüklenir? | Token Maliyeti |
|:------|:-------------|:---------------|
| 1. Başlangıç | Sadece skill isimleri ve açıklamaları | ~50-100 token/skill |
| 2. Aktivasyon | Tam SKILL.md içeriği | ~1000-5000 token |
| 3. İhtiyaç | Scriptler, referanslar | Değişken |

> 20 skill'iniz varsa, başlangıçta sadece ~1500 token harcarsınız. Sadece 1-2 skill aktifleşir, geri kalanı hiç yüklenmez.

---

## 3. Skills Mimarisi: Progressive Disclosure

Progressive disclosure (aşamalı açıklama), skills sisteminin kalbidir. Üç katmandan oluşur:

### Katman 1: Katalog (Catalog)

```
Ajan başlatılır
    ↓
    Skills dizinleri taranır
    ↓
    Her skill için sadece:
      <skill name="pdf-extract">
        <description>Extracts text and tables from PDF files...</description>
      </skill>
    ↓
    Sistem prompt'una XML olarak eklenir
    ↓
    Ajan hangi skill'lerin mevcut olduğunu BİLİR
    ama içeriğini OKUMAMIŞTIR
```

### Katman 2: Talimatlar (Instructions)

```
Kullanıcı: "Bu PDF'deki form alanlarını çıkar"
    ↓
Ajan: "pdf-extract skill'i var, bunu kullanmalıyım"
    ↓
Ajan SKILL.md dosyasını read() ile yükler
    ↓
Ajan artık workflow'u bilmektedir
```

### Katman 3: Kaynaklar (Resources)

```
SKILL.md: "scripts/extract.py dosyasını çalıştır"
    ↓
Ajan: scripts/extract.py dosyasını read() ile yükler
    ↓
Gerekirse references/form-fields.md'yi de yükler
    ↓
Script'i çalıştırır
```

### Görselleştirme

```
Zaman çizelgesi:
├── Session başlangıcı ──────────────────────────────────────────
│   ┌─ Katalog (50 token/skill) ──┐
│   │  pdf-extract, code-review,  │
│   │  web-search, data-analysis  │
│   └──────────────────────────────┘
│
├── Kullanıcı: "PDF işle" ──────────────────────────────────────
│   ┌─ Talimatlar (2000 token) ──┐
│   │  PDF form alanlarını çıkar │
│   │  ...adımlar...             │
│   └─────────────────────────────┘
│
├── Ajan: "extract.py çalıştır" ────────────────────────────────
│   ┌─ Kaynaklar (500 token) ────┐
│   │  scripts/extract.py       │
│   └─────────────────────────────┘
│
└── Token tasarrufu: 20 skill × 2000 token = 40.000 token
    (sadece 1 skill aktifleşti = 3000 token toplam)
```

---

## 4. SKILL.md Formatı — Frontmatter

Her skill'in kalbi `SKILL.md` dosyasıdır. İki bölümden oluşur: **frontmatter** (YAML metaveri) ve **gövde** (markdown talimatlar).

### Frontmatter Alanları (Zorunlu)

```yaml
---
name: pdf-processing          # Zorunlu
description: >               # Zorunlu
  Extracts text and tables from PDF files, fills PDF forms,
  and merges multiple PDFs. Use when working with PDF documents.
---
```

#### `name` — Zorunlu

Skill'in benzersiz tanımlayıcısı.

| Kural | Geçerli | Geçersiz |
|:------|:--------|:---------|
| 1-64 karakter | `pdf-processing` | (64+ karakter) |
| Küçük harf, rakam, tire | `data-analysis-v2` | `Data-Analysis` |
| Tire ile başlamaz/bitmez | `code-review` | `-code-review` |
| Ardışık tire olmaz | `multi-step` | `multi--step` |

```yaml
# Geçerli
name: pdf-processing
name: data-analysis
name: code-review
name: my-skill-v2

# Geçersiz
name: PDF-Processing    # Büyük harf
name: -pdf              # Tire ile başlar
name: pdf--processing   # Ardışık tire
```

#### `description` — Zorunlu

Skill'in ne yaptığı ve ne zaman kullanılacağı. **Ajanın skill'i seçmesini sağlayan kritik alandır.**

| Kural | Değer |
|:------|:------|
| Uzunluk | 1-1024 karakter |
| İçerik | Ne yapar + ne zaman kullanılır |
| Stil | Spesifik anahtar kelimeler içermeli |

```yaml
# İyi
description: >
  Extracts text and tables from PDF files, fills PDF forms,
  and merges multiple PDFs. Use when working with PDF documents
  or when the user mentions PDFs, forms, or document extraction.

# Kötü
description: Helps with PDFs.
```

**Neden iyi açıklama önemlidir?** Ajan, kullanıcının isteğini skill açıklamalarıyla eşleştirir. "PDF form alanlarını doldur" isteği -> `pdf-processing` skill'inin açıklamasındaki "fills PDF forms" ile eşleşir.

### Frontmatter Alanları (İsteğe Bağlı)

#### `license`

```yaml
license: MIT
license: Apache-2.0
license: Proprietary. LICENSE.txt has complete terms
```

#### `compatibility`

```yaml
compatibility: Requires git, docker, jq, and access to the internet
compatibility: Requires Python 3.14+ and uv
compatibility: Designed for Claude Code (or similar products)
```

#### `metadata`

```yaml
metadata:
  author: example-org
  version: "1.0"
  category: document-processing
```

#### `allowed-tools` (Deneysel)

Ön onaylı araçlar. Uzay-boşluk ile ayrılır:

```yaml
allowed-tools: Bash(git:*) Bash(jq:*) Read
```

Bu, skill'in hangi araçları kullanabileceğini kısıtlar. Tüm implementasyonlar desteklemez.

#### `disable-model-invocation`

```yaml
disable-model-invocation: true
```

`true` olduğunda skill sistem prompt'unda gösterilmez. Kullanıcı `/skill:name` ile manuel çağırmalıdır.

### Eksiksiz Frontmatter Örneği

```yaml
---
name: code-review
description: >
  Reviews git commits and pull requests. Performs security scanning,
  code quality analysis, and test coverage checks.
  Use when user says "review this commit" or "review PR".
license: MIT
compatibility: git >= 2.0, Python >= 3.10
metadata:
  author: dev-team
  version: "1.2"
allowed-tools: Bash(git:*) Bash(jq:*) Read Edit
---
```

---

## 5. SKILL.md Formatı — Gövde (Body)

Frontmatter'dan sonra gelen markdown içeriği, skill'in **çalışma talimatlarını** içerir. Herhangi bir format kısıtlaması yoktur — ajan ne yazılırsa onu okur ve uygular.

### Önerilen Bölümler

#### Setup (Kurulum)

Skill'in çalışması için gereken bağımlılıklar:

````markdown
## Setup

One-time setup before first use:

```bash
cd /path/to/skill && npm install
pip install -r scripts/requirements.txt
```
````

#### Usage (Kullanım)

Nasıl çalıştırılacağı:

````markdown
## Usage

```bash
./scripts/extract.py <input.pdf> <output-dir>
./scripts/extract.py --help   # Tüm seçenekler
```
````

#### Workflow (İş Akışı)

Adım adım talimatlar:

```markdown
## Workflow

1. Read the input file with `scripts/read.py`
2. Validate the format
3. Process each section:
   a. Extract text
   b. Detect tables
   c. Save as markdown
4. Generate summary report
```

#### Examples (Örnekler)

````markdown
## Examples

**Input:** `form.pdf` (3 pages, AcroForm fields)
**Output:**
```markdown
# Form Fields
- name: John Doe
- email: john@example.com
- signature: [missing]
```
````

#### Guidelines (Kurallar)

```markdown
## Guidelines

- Never modify the original file
- Preserve image metadata when possible
- Use UTF-8 encoding for all outputs
- Report errors with file name and line number
```

#### Edge Cases (Kenar Durumlar)

```markdown
## Edge Cases

- **Empty PDF:** Return "No content found"
- **Password protected:** Ask user for password
- **Corrupted file:** Report specific corruption error
- **Very large PDF (>100MB):** Process page by page
```

### Eksiksiz SKILL.md Örneği

````markdown
---
name: pdf-processing
description: >
  Extracts text and tables from PDF files, fills PDF forms,
  and merges multiple PDFs. Use when working with PDF documents.
license: MIT
compatibility: Python >= 3.10, pymupdf >= 1.24
---

# PDF Processing

## Setup

```bash
pip install pymupdf pandas
```

## Usage

```bash
./scripts/extract.py input.pdf output/     # Extract text and tables
./scripts/merge.py output.pdf input/*.pdf  # Merge multiple PDFs
```

## Workflow

1. **Validate:** Check file exists, is readable, is valid PDF
2. **Extract:** Run `scripts/extract.py` on the input file
3. **Review:** Check output for completeness
4. **Report:** Present results as markdown summary

## Guidelines

- Extract text first, then tables
- Tables go into separate markdown files
- Maintain original page order

## Edge Cases

- **Encrypted PDF:** Ask for password
- **Scanned PDF (no text):** Inform user OCR is needed
- **Empty pages:** Skip with note

## References

See [advanced topics](references/advanced.md) for form filling and OCR.
````

---

## 6. Skills Dizin Yapısı

Agent Skills standardına göre bir skill dizini şu yapıda olabilir:

```
my-skill/
├── SKILL.md              # ZORUNLU: frontmatter + talimatlar
├── scripts/              # İSTEĞE BAĞLI: çalıştırılabilir kod
│   ├── process.sh
│   ├── extract.py
│   └── requirements.txt
├── references/           # İSTEĞE BAĞLI: detaylı dökümantasyon
│   ├── API.md
│   ├── form-fields.md
│   └── CHANGELOG.md
├── assets/               # İSTEĞE BAĞLI: statik kaynaklar
│   ├── template.json
│   ├── logo.png
│   └── schema.sql
└── README.md             # İSTEĞE BAĞLI: insanlar için doküman
```

### Dizin Bileşenleri

| Bileşen | Zorunlu? | Açıklama |
|:---------|:----------|:----------|
| `SKILL.md` | **Evet** | Frontmatter + talimatlar. Ajanın okuyacağı ana dosya |
| `scripts/` | Hayır | Çalıştırılabilir kod dosyaları. Kendi kendine yetebilmeli veya bağımlılıkları belirtilmeli |
| `references/` | Hayır | Ajanın ihtiyaç duydukça okuyacağı referans dokümanlar. Kısa ve odaklı olmalı |
| `assets/` | Hayır | Statik kaynaklar (şablonlar, resimler, veri dosyaları) |
| `README.md` | Hayır | İnsan okuyucular için doküman (ajan tarafından yüklenmez) |

### Dizin Kuralları

1. `SKILL.md` **doğrudan** skill dizininin kökünde olmalıdır
2. Diğer dosyalara SKILL.md içinde **göreceli yol** ile başvurulur:

```markdown
Doğru:
  See [the reference guide](references/REFERENCE.md) for details.
  Run: scripts/extract.py

Yanlış:
  See /home/user/skills/my-skill/references/REFERENCE.md  # Mutlak yol
  See ../other-skill/SKILL.md                               # Dışarı referans
```

3. Referans zinciri en fazla 1-2 seviye derin olmalıdır

### Keşifte Dizin Yapısı

Ajann tarama sırasında dizinler şöyle taranır:

```
~/.agents/skills/              ← Global skills
├── pdf-processing/
│   └── SKILL.md              ← BULUNDU
├── data-analysis/
│   └── SKILL.md              ← BULUNDU
└── README.md                  ← İGNORE (skill dizini değil)
```

**Önemli kural:** Sadece `SKILL.md` içeren dizinler skill olarak algılanır. Kök dizindeki `.md` dosyaları (örn. `README.md`) skill olarak algılanmaz.

---

## 7. Keşif (Discovery): Ajan Skills'i Nasıl Bulur?

### Tarama Konumları

Ajan, başlangıçta skills'i şu konumlarda arar:

| Konum | Amaç | Örnek |
|:------|:-----|:------|
| **Global kullanıcı** | Tüm projelerde kullanılabilir | `~/.pi/agent/skills/` |
| **Global standart** | Cross-client paylaşım | `~/.agents/skills/` |
| **Proje (güvenilir)** | Sadece bu projede | `.pi/skills/` (proje root) |
| **Proje standart** | Cross-client proje paylaşımı | `.agents/skills/` |
| **Paket** | npm paketleri ile gelen | `node_modules/.../skills/` |
| **Kullanıcı tanımlı** | settings.json ile | `"skills": ["~/custom-skills"]` |
| **CLI ile** | Tek seferlik | `pi --skill ./my-skill` |

### Tarama Sırası

```
1. Global: ~/.pi/agent/skills/     ← root .md dosyaları da skill sayılır
2. Global: ~/.agents/skills/       ← sadece SKILL.md içeren dizinler
3. Proje:  .pi/skills/             ← root .md dosyaları da skill sayılır
4. Proje:  .agents/skills/         ← sadece SKILL.md içeren dizinler
5. Paketler: node_modules/...      ← SKILL.md içeren dizinler
```

### İsim Çakışması (Name Collision)

Aynı isimde iki skill varsa:

```
~/.agents/skills/code-review/  ← Global (daha düşük öncelik)
./.pi/skills/code-review/      ← Proje (daha yüksek öncelik)
```

**Kural:** Proje seviyesi global seviyeyi ezer. Aynı seviyede ise ilk bulunan geçerlidir. Çakışma uyarısı verilir.

### Tarama Kuralları

- `.git/` ve `node_modules/` gibi dizinler atlanır
- Maksimum derinlik genelde 4-6 seviye
- `.gitignore`'a saygı duyulabilir
- Maksimum 2000 dizin sınırı (performans)

### Performans Notu

Eğer skills'iniz yoksa veya kullanmak istemiyorsanız:

```bash
pi --no-skills    # Skills keşfini tamamen devre dışı bırakır
```

---

## 8. Aktivasyon: Skills Nasıl Yüklenir ve Çalıştırılır

### Model Tarafından Aktivasyon

Ajan, kullanıcının isteğini skill açıklamalarıyla eşleştirir:

```
Kullanıcı: "Şu PDF'deki metni çıkar"

Ajan düşünür:
  1. Mevcut skill'ler: pdf-processing, code-review, web-search
  2. "PDF'deki metni çıkar" → pdf-processing açıklamasındaki
     "Extracts text...from PDF files" ile eşleşir
  3. SKILL.md'i oku
  4. Talimatları uygula
```

Her ajan modeli skills'i otomatik olarak kullanmayabilir. Bazı durumlarda ajanın SKILL.md'i `read` ile yüklemesi için prompt'ta yönlendirme gerekir.

### Kullanıcı Tarafından Aktivasyon (Manuel)

Kullanıcı, skill'i doğrudan komutla çağırabilir:

```bash
/skill:pdf-processing              # Skill'i yükle ve çalıştır
/skill:pdf-processing merge        # Argümanla çağır
/skill:caveman ultra               # Skill adı + parametre
```

Argümanlar, skill içeriğine `User: <args>` olarak eklenir.

### Aktivasyon Anında Ne Olur?

```
Kullanıcı: /skill:pdf-processing report.pdf
    ↓
Ajan SKILL.md dosyasını read() ile okur
    ↓
Ajan talimatları sistem prompt'una ekler
    ↓
Ajan:
  "Tamam, PDF processing skill'ini yükledim.
   Şimdi scripts/extract.py report.pdf output/ çalıştıracağım"
    ↓
Ajan bash() ile script'i çalıştırır
    ↓
Sonucu kullanıcıya sunar
```

### Aktivasyonun Kapatılması

```yaml
# SKILL.md frontmatter'ında
disable-model-invocation: true
```

Bu skill sistem prompt'unda görünmez. Sadece `/skill:name` ile çağrılabilir. Özel veya tehlikeli skill'ler için kullanışlıdır.

---

## 9. Skills'in Yaşam Döngüsü

```
OLUŞTUR  →  DOĞRULA  →  YERLEŞTİR  →  KEŞFET  →  YÜKLE  →  ÇALIŞTIR
```

### Adım 1: Oluştur (Create)

```bash
mkdir -p my-skill/scripts my-skill/references
touch my-skill/SKILL.md
```

SKILL.md'yi frontmatter ve talimatlarla doldurun.

### Adım 2: Doğrula (Validate)

```bash
# skills-ref aracı ile doğrulama
skills-ref validate ./my-skill

# Veya manuel kontrol:
# - name geçerli mi?
# - description var mı (1-1024 karakter)?
# - SKILL.md okunabilir mi?
# - Script'ler çalışabiliyor mu?
```

### Adım 3: Yerleştir (Deploy)

```bash
# Global
cp -r my-skill ~/.pi/agent/skills/

# Proje
cp -r my-skill /path/to/project/.pi/skills/

# Cross-client
cp -r my-skill ~/.agents/skills/
```

### Adım 4: Keşfet (Discovery)

Ajan başlatıldığında:

```
pi start
  ↓
Tarama: ~/.pi/agent/skills/ → pdf-processing bulundu
Tarama: ~/.agents/skills/ → (yok)
Tarama: ./.pi/skills/ → (yok)
  ↓
Sistem prompt'una skills katalogu eklendi:
  <skills>
    <skill name="pdf-processing">
      <description>Extracts text and tables from PDF files...</description>
    </skill>
  </skills>
```

### Adım 5: Yükle (Load)

Kullanıcı isteği veya `/skill:pdf-processing` ile SKILL.md yüklenir:

```
Ajan SKILL.md'i read() ile okur
  ↓
Talimatlar ajanın context'ine eklenir
  ↓
Artık workflow bilinmektedir
```

### Adım 6: Çalıştır (Execute)

Ajan talimatları takip eder, script'leri çalıştırır, sonuçları döndürür.

---

## 10. Skills vs Tools vs Plugins vs Prompts — Karşılaştırma

Bu dört kavram sıkça karıştırılır. İşte net ayrımlar:

### Tanımlar

| Kavram | Nedir? | Boyut |
|:-------|:-------|:------|
| **Tool (Araç)** | JSON Schema ile tanımlanmış tek bir fonksiyon | ~10-50 satır |
| **Skill** | Bir iş akışını anlatan paketlenmiş talimatlar | ~50-500 satır + dosyalar |
| **Plugin** | Üçüncü taraf API'ye bağlanan yapılandırma | API spec + auth |
| **System Prompt** | Sistematik olarak ajanın davranışını yönlendiren metin | ~10-200 satır |

### Karşılaştırma Tablosu

| Özellik | **Tool** | **Skill** | **Plugin** | **System Prompt** |
|:--------|:---------|:----------|:-----------|:------------------|
| **Kapsam** | Tek işlem | Tüm iş akışı | Belirli hizmet | Genel davranış |
| **Format** | JSON Schema | SKILL.md (markdown) | OpenAPI manifest | Düz metin |
| **Yükleme** | Her zaman prompt'ta | Aşamalı (progressive) | Her zaman | Her zaman |
| **Akıl yürütme** | Model karar verir | Model okur + uygular | Model doğrudan çağırır | Modele talimat verir |
| **Kod içerir** | Hayır | Evet (scripts/) | Hayır | Hayır |
| **Referanslar** | Hayır | Evet (references/) | Hayır | Hayır |
| **Platform** | Herhangi bir LLM API | Agent Skills uyumlu ajan | Belirli ürün | Herhangi bir LLM |
| **Paylaşım** | API spesifikasyonu | GitHub reposu | Store | Kopyala-yapıştır |

### Somut Örnek

```
Kullanıcı: "Bu commit'i review et"

TOOL ile:
  → Ajan çağırır:
      read_file(diff)          → diff döner
      search_similar(query)    → benzer hatalar döner
      suggest_fix(code)        → düzeltme önerisi döner
  → Her adım ayrı JSON fonksiyon çağrısı
  → prompt'ta her zaman yer kaplar

SKILL ile:
  → Ajan SKILL.md'i okur:
      "Önce diff'i oku, güvenlik taraması yap,
       benzer hataları ara, sonuçları raporla"
  → Talimatlar sadece ihtiyaç anında yüklenir
  → Script'ler ve referanslar ayrı dosyalarda

PLUGIN ile:
  → GitHub API'sine bağlanır
  → Sadece o API'nin yapabildikleriyle sınırlı
  → Harici bir servise bağımlı

SYSTEM PROMPT ile:
  → "Her commit review'de diff'i oku, güvenlik kontrolü yap..."
  → Her konuşmada yer kaplar, güncellemesi zor
```

### Ne Zaman Hangisi?

| Durum | Çözüm |
|:------|:------|
| Basit bir işlem (hava durumu sorgula) | **Tool** |
| Karmaşık, çok adımlı iş akışı (PDF işleme) | **Skill** |
| Harici bir servise bağlanma (Slack'e mesaj at) | **Plugin** veya **Tool** |
| Kalıcı davranış değişikliği (hep kibar ol) | **System Prompt** |

---

## 11. Skills Yazma En İyi Uygulamaları

### 1. Açıklama (Description) Kritiktir

İyi açıklama, ajanın skill'i doğru zamanda seçmesini sağlar.

```yaml
# Kötü — çok genel, eşleşme zor
description: Helps with PDFs.

# İyi — spesifik, anahtar kelimeli
description: >
  Extracts text and tables from PDF files, fills PDF forms,
  and merges multiple PDFs into one. Use when working with PDFs,
  extracting document content, or when user mentions PDF forms.

# Daha iyi — kullanım senaryosu eklenmiş
description: >
  PDF belgelerden metin ve tablo çıkarır, PDF formları doldurur,
  birden çok PDF'i birleştirir. Kullanıcı "PDF işle", "form doldur",
  "belge çıkar" gibi ifadeler kullandığında aktifleşir.
```

### 2. SKILL.md Kısa Tutun (500 satır altı)

Uzun SKILL.md dosyaları context tüketir. Detaylı referansları ayrı dosyalara koyun:

```markdown
# SKILL.md (kısa — workflow ana hatları)

## Quick Start
scripts/extract.py input.pdf output/

## Workflow
1. Validate input
2. Run extraction script
3. Review output

## References
Detaylı form doldurma için [Form Fields](references/form-fields.md) dosyasına bakın.
```

### 3. Script'leri Kendi Kendine Yeterli Yapın

```python
#!/usr/bin/env python3
"""PDF extraction script. Self-contained with error handling."""

import sys
import argparse

def main():
    parser = argparse.ArgumentParser(description="Extract PDF content")
    parser.add_argument("input", help="Input PDF file")
    parser.add_argument("output", help="Output directory")
    args = parser.parse_args()

    try:
        # Extraction logic
        print(f"Extracted content to {args.output}")
    except Exception as e:
        print(f"Error: {e}", file=sys.stderr)
        sys.exit(1)

if __name__ == "__main__":
    main()
```

### 4. Hata Yönetimi Ekleyin

```markdown
## Error Handling

| Hata | Ajan Ne Yapmalı |
|:-----|:----------------|
| Dosya bulunamadı | Kullanıcıya sor, doğru yolu iste |
| Şifre korumalı | Şifreyi sor |
| Bozuk PDF | Hata mesajını göster, alternatif öner |
| Çok büyük dosya | Sayfa sayfa işle, ilerleme raporu ver |
```

### 5. Kenar Durumları (Edge Cases) Belirtin

```markdown
## Edge Cases

- **Boş PDF:** "No content found" döndür
- **Sadece resim içeren PDF:** OCR gerekli olduğunu belirt
- **Farklı dil:** Dil otomatik algılanır, UTF-8 kullanılır
- **Parçalı PDF:** Sayfa numaralarını koru
```

### 6. Göreceli Yollar Kullanın

```markdown
Doğru:
  scripts/extract.py input.pdf output/
  references/form-fields.md

Yanlış:
  /home/user/skills/pdf-processing/scripts/extract.py
  ../../other-skill/SKILL.md
```

### 7. Kurulum Adımlarını Ekleyin

```markdown
## Setup

Before first use:

```bash
# Python dependencies
pip install pymupdf pandas

# Or using uv (faster)
uv pip install pymupdf pandas
```

> **Not:** macOS'ta `brew install pymupdf` gerekebilir.
```

### 8. Dil ve Kültür Farkı

Skills, ajanın dilinde yazılmalıdır. Türkçe bir skill:

```yaml
---
name: turkce-ozet
description: >
  Türkçe metinleri özetler, Türkçe karakterleri korur.
  Kullanıcı Türkçe bir metin verdiğinde veya "özet çıkar"
  dediğinde aktifleşir.
---
```

---

## 12. Skills Çalışma Zamanı: Scriptler ve Kaynaklar

### Script'ler

Script'ler, skill'in çalıştırılabilir kod dosyalarıdır. Ajan bunları `bash` veya benzeri bir araçla çalıştırır.

**Desteklenen diller (ajana bağlı):**

| Dil | Kim Çalıştırır? | Örnek |
|:----|:----------------|:------|
| Bash | Tüm ajanlar | `scripts/process.sh` |
| Python | Çoğu ajan | `scripts/extract.py` |
| JavaScript/Node | Bazı ajanlar | `scripts/search.js` |
| Herhangi bir çalıştırılabilir | Shebang ile | `scripts/tool` |

**Script yazma kuralları:**

1. **Shebang ekleyin:** `#!/usr/bin/env bash` veya `#!/usr/bin/env python3`
2. **Hata kodları döndürün:** Başarı için 0, hata için 1
3. **Yardım mesajı ekleyin:** `--help` veya `-h` ile
4. **Argümanları doğrulayın:** Eksik argüman durumunda anlamlı hata mesajı
5. **Çıktıyı yapılandırın:** Ajanın okuyabileceği formatta (genelde stdout)

### Referanslar

Referans dosyaları, detaylı bilgi içeren yardımcı dokümanlardır. Ajan ihtiyaç duydukça `read` ile yükler.

**İyi referans dosyası:**
- Kısa ve odaklı (200-500 satır)
- Belirli bir konuda derinlemesine bilgi
- Örneklerle desteklenmiş
- Kolayca taranabilir başlıklar

**Kötü referans dosyası:**
- 2000+ satır
- Birden çok konuyu karıştırmış
- Örnek yok
- Düz metin, başlık yok

### Script Örneği: PDF Extract

```python
#!/usr/bin/env python3
"""Extract text and tables from PDF."""

import sys
import json
import fitz  # pymupdf

def extract(pdf_path):
    """Extract text and tables from PDF."""
    doc = fitz.open(pdf_path)
    result = []
    
    for page_num, page in enumerate(doc, 1):
        text = page.get_text()
        tables = page.find_tables()
        
        result.append({
            "page": page_num,
            "text": text.strip(),
            "tables": [str(t.extract()) for t in tables]
        })
    
    return result

if __name__ == "__main__":
    if len(sys.argv) < 3:
        print("Usage: extract.py <input.pdf> <output.json>", file=sys.stderr)
        sys.exit(1)
    
    data = extract(sys.argv[1])
    with open(sys.argv[2], "w") as f:
        json.dump(data, f, indent=2, ensure_ascii=False)
    
    print(f"✅ Extracted {len(data)} pages to {sys.argv[2]}")
```

### Kaynak Dosyaları (Assets)

Statik kaynaklar: şablonlar, resimler, yapılandırma dosyaları.

```
assets/
├── report-template.md     # Rapor şablonu
├── config.json            # Varsayılan yapılandırma
└── logo.png               # Şirket logosu
```

---

## 13. Adım Adım: İlk Skill'inizi Yazın

Şimdi bir **Markdown belge düzenleyici** skill'i yazalım. Bu skill, bir markdown dosyasını alıp belirli kurallara göre düzenler.

### Adım 1: Dizin Yapısını Oluşturun

```bash
mkdir -p md-formatter/scripts md-formatter/references
```

### Adım 2: SKILL.md Yazın

```markdown
---
name: md-formatter
description: >
  Formats Markdown files according to style rules: fixes heading hierarchy,
  ensures consistent list formatting, adds proper spacing.
  Use when user wants to format, clean up, or standardize a Markdown file.
license: MIT
compatibility: Python >= 3.8
---

# Markdown Formatter

## Setup

```bash
pip install mdformat
```

## Usage

```bash
scripts/format.sh <input.md> [output.md]
```

If no output file specified, formats in-place.

## Workflow

1. Validate input file exists and is readable Markdown
2. Run `mdformat` with the style config
3. Check for common issues:
   - Headings in wrong order (e.g., H3 after H1 without H2)
   - Mixed list styles (- vs *)
   - Missing blank lines around code blocks
   - Trailing whitespace
4. Fix detected issues
5. Report what was changed

## Style Rules

- ATX headings (`##` not `## ` with space) — already standard
- Unordered lists use `-`
- One blank line before code blocks
- No trailing whitespace
- Maximum line length: 100 characters

## References

See [style-guide.md](references/style-guide.md) for full rule details.
```

### Adım 3: Script'i Yazın

```bash
#!/usr/bin/env bash
# Format Markdown file with mdformat and custom checks

set -euo pipefail

INPUT="${1:-}"
OUTPUT="${2:-}"

if [ -z "$INPUT" ]; then
    echo "Usage: format.sh <input.md> [output.md]"
    exit 1
fi

if [ ! -f "$INPUT" ]; then
    echo "Error: File '$INPUT' not found"
    exit 1
fi

if [ -n "$OUTPUT" ]; then
    mdformat "$INPUT" --output "$OUTPUT"
else
    mdformat "$INPUT"
fi

echo "✅ Formatted: $INPUT"
```

### Adım 4: Referans Dosyası Ekleyin (İsteğe Bağlı)

```markdown
# MD Format Style Guide

## Headings
- Always use ATX-style: `## Heading`
- Never use closing `#` characters
- Maintain hierarchy (no jumps)

## Lists
- Unordered: use `-`
- Ordered: use `1.`
- Nested lists: 2-space indent

## Code Blocks
- Use fenced code blocks with language specifier
- One blank line before and after
- No indented code blocks

## Tables
- Use GitHub Flavored Markdown table syntax
- Alignment markers optional
```

### Adım 5: Skills Dizinine Kopyalayın

```bash
# Global kullanım
cp -r md-formatter ~/.pi/agent/skills/

# Test et
pi
> /skill:md-formatter README.md
```

### Adım 6: Doğrulayın

```bash
# skills-ref aracı varsa
skills-ref validate ./md-formatter

# Manuel kontrol
cat md-formatter/SKILL.md          # Frontmatter + talimatlar
ls md-formatter/scripts/           # Script mevcut
chmod +x md-formatter/scripts/*.sh # Çalıştırılabilir
```

---

## 14. Skills Doğrulama ve Test

### skills-ref Aracı

Agent Skills standardı, skills'i doğrulamak için `skills-ref` aracı sağlar:

```bash
# Kurulum
npm install -g @agentskills/skills-ref

# Doğrulama
skills-ref validate ./my-skill

# Çıktı:
# ✅ Name valid: pdf-processing
# ✅ Description valid: 142 chars
# ✅ Frontmatter complete
# ⚠ License not specified (optional)
```

### Manuel Doğrulama Kontrol Listesi

```
[ ] SKILL.md mevcut mu?
[ ] name alanı var mı? (1-64 karakter, geçerli formatta)
[ ] description alanı var mı? (1-1024 karakter)
[ ] Frontmatter doğru YAML formatında mı? (--- ile ayrılmış)
[ ] Gövde kısmı var mı? (en azından başlık)
[ ] Göreceli yollar doğru mu?
[ ] Script'ler çalıştırılabilir mi? (chmod +x)
[ ] Script'ler shebang içeriyor mu?
[ ] Script'ler hata durumunda anlamlı mesaj veriyor mu?
[ ] references/ dosyaları mevcut ve okunabilir mi?
```

### Test Senaryoları

Skill'inizi şu durumlarda test edin:

| Senaryo | Beklenen |
|:--------|:---------|
| Normal kullanım | Skill çalışır, doğru sonuç döner |
| Hatalı girdi | Anlamlı hata mesajı |
| Eksik bağımlılık | Kurulum talimatı ister |
| Çok büyük girdi | Performans sorunsuz |
| Özel karakterler | UTF-8 korunur |
| Boş girdi | Uygun mesaj döner |

---

## 15. İleri Seviye: Skill Kompozisyonu ve Çoklu Adım

### Skill'lerin Birleştirilmesi

Bir skill, başka bir skill'in sonucunu kullanabilir. Örneğin:

```
Kullanıcı: "Şu PDF'deki verileri analiz et ve raporla"

1. pdf-processing skill'i çalışır → PDF'den veri çıkarır
2. data-analysis skill'i çalışır → Veriyi analiz eder
3. report-generator skill'i çalışır → Rapor oluşturur
```

### SKILL.md'de Alt-İş Akışları

```markdown
## Workflow

### Phase 1: Extract
1. Validate input file
2. Run `scripts/extract.py` to get raw data
3. Save intermediate result to `output/raw.json`

### Phase 2: Analyze
1. Read `output/raw.json`
2. Run statistical analysis
3. Generate summary metrics

### Phase 3: Report
1. Combine metrics into template
2. Generate markdown report
3. Present to user
```

### Çoklu Dil Desteği

```yaml
---
name: multi-lang-summary
description: >
  Summarizes text in multiple languages. Detects input language
  automatically. Supports EN, TR, DE, FR, ES.
metadata:
  languages: en, tr, de, fr, es
---
```

### conditional Workflow

```markdown
## Workflow

IF input is PDF:
  1. Extract text with scripts/extract.py
  2. Process extracted text

IF input is DOCX:
  1. Convert to markdown with scripts/convert.sh
  2. Process markdown

IF input is plain text:
  1. Process directly

ALWAYS:
  - Validate output
  - Report summary
```

---

## 16. Platformlar Arası Skills (Pi, Claude, Codex)

### Agent Skills Standardı

Skills, **Agent Skills** standardı sayesinde farklı ajanlarda çalışabilir. Standardı destekleyen ajanlar:

| Ajan | Skills Dizini | Notlar |
|:-----|:-------------|:-------|
| **Pi** | `~/.pi/agent/skills/` | root `.md` dosyalarını da skill sayar |
| **Claude Code** | `~/.claude/skills/` | SKILL.md zorunlu |
| **OpenAI Codex** | `~/.codex/skills/` | SKILL.md zorunlu |
| **Herhangi** | `~/.agents/skills/` | Standart cross-client konum |

### Skills Paylaşımı

Bir skill'i birden çok ajanda kullanmak için:

```bash
# Pi'de kullan
cp -r my-skill ~/.pi/agent/skills/

# Aynı skill'i Claude Code'da da kullan
ln -s ~/.pi/agent/skills/my-skill ~/.claude/skills/my-skill
```

Veya standart dizini kullanın:

```bash
cp -r my-skill ~/.agents/skills/
# → Pi, Claude Code, Codex hepsi burayı tarar
```

### Pi'de Diğer Ajanların Skills'lerini Kullanma

```json
// ~/.pi/settings.json
{
  "skills": [
    "~/.claude/skills",
    "~/.codex/skills"
  ]
}
```

### Pi'ye Özgü Özellikler

| Özellik | Pi Davranışı |
|:--------|:-------------|
| **root .md dosyaları** | `~/.pi/agent/skills/` altındaki `.md` dosyaları da skill sayılır |
| **İsim çakışması** | Proje → Global. Aynı seviyede ilk bulunan kazanır |
| **Validasyon** | Standarttan sapan durumlarda uyarı verir, yine de yükler |
| **Eksik description** | Skill yüklenmez (standarttan katı) |
| **Bilinmeyen frontmatter** | Yok sayılır (standarttan esnek) |
| **--no-skills** | Tüm keşfi devre dışı bırakır |

---

## 17. Skills Depoları ve Topluluk

### Önemli Skills Kaynakları

| Kaynak | Açıklama |
|:-------|:---------|
| [Anthropic Skills Repo](https://github.com/anthropics/skills) | 15+ örnek skill: belge işleme, tasarım, geliştirme |
| [Pi Skills Repo](https://github.com/badlogic/pi-skills) | Web arama, browser otomasyonu, Google API'leri |
| [Agent Skills Standard](https://agentskills.io) | Resmi spesifikasyon ve dökümantasyon |
| [Agent Skills GitHub](https://github.com/agentskills/agentskills) | Standart geliştirme ve validasyon araçları |

### Kendi Skill'inizi Paylaşma

1. Skill'inizi bir GitHub reposunda yayınlayın
2. README.md ekleyin (insanlar için)
3. skills.sh rozeti ekleyin (opsiyonel):

```markdown
[![skills.sh](https://skills.sh/b/kullaniciadi/repo)](https://skills.sh/kullaniciadi/repo)
```

### Skills Toplulukları

- **Discord:** [Agent Skills Discord](https://discord.gg/MKPE9g8aUy)
- **GitHub Discussions:** [agentskills/agentskills](https://github.com/agentskills/agentskills/discussions)

---

## 18. Sık Yapılan Hatalar

### Hata 1: Açıklama Çok Kısa

```yaml
# HATALI
description: PDF işler.

# DOĞRU
description: >
  PDF belgelerden metin ve tablo çıkarır, formları doldurur.
  Use when working with PDFs or document extraction.
```

**Sonuç:** Ajan, ne zaman kullanacağını bilemez, skill hiç aktifleşmez.

### Hata 2: Frontmatter Eksik veya Bozuk

```markdown
# HATALI - frontmatter yok
PDF Processing Skill
==================
...

# HATALI - YAML hatası
---
name: pdf-processing
description: "PDF işleme
---

# DOĞRU
---
name: pdf-processing
description: PDF işleme skill'i.
---
```

**Sonuç:** Skill hiç yüklenmez veya hatalı yüklenir.

### Hata 3: Mutlak Yol Kullanımı

```markdown
# HATALI
Run /home/user/skills/pdf-processing/scripts/extract.py

# DOĞRU
Run scripts/extract.py
```

**Sonuç:** Skill başka bir ajana taşındığında çalışmaz.

### Hata 4: Script'leri Çalıştırılabilir Yapmamak

```bash
# HATALI
ls -l scripts/extract.py
# -rw-r--r-- ... extract.py  ← çalıştırılamaz

# DOĞRU
chmod +x scripts/extract.py
ls -l scripts/extract.py
# -rwxr-xr-x ... extract.py  ← çalıştırılabilir
```

**Sonuç:** Ajan script'i çalıştıramaz, hata alır.

### Hata 5: Çok Uzun SKILL.md

```markdown
# HATALI - 2000 satırlık SKILL.md
...
## Bölüm 1: Giriş
...
## Bölüm 2: Detaylı Kurulum
...
## Bölüm 3: Tüm Seçenekler
...
# vs. 50 bölüm daha
```

**Sorun:** Context'i gereksiz yere şişirir.
**Çözüm:** SKILL.md'yi kısa tutun (>500 satır), detayları `references/` dosyalarına taşıyın.

### Hata 6: Dil Uyumsuzluğu

```yaml
---
name: turkce-skill
description: >
  This skill does Turkish text processing. Use when...
# BU YANLIŞ — kullanıcı Türkçe konuşuyor,
# açıklama İngilizce olunca ajan eşleştiremez
---
```

**Çözüm:** Açıklamayı, skill'in kullanılacağı dilde yazın.

### Hata 7: Hata Yönetimi Eksik

```markdown
## Workflow
1. Run script
2. Get results
# HATA: Hiçbir hata durumu belirtilmemiş
```

**Çözüm:** Her adım için olası hataları ve yapılacakları belirtin.

### Hata 8: Skill İsmi Kurallara Uymuyor

```yaml
# HATALI
name: PDF-Processing    # Büyük harf
name: -my-skill         # Tire ile başlıyor
name: code--review      # Ardışık tire
name: a_very_long_name_that_exceeds_sixty_four_characters_easily  # Çok uzun

# DOĞRU
name: pdf-processing
name: code-review
name: data-analysis-v2
```

---

## 19. Alıştırmalar

### Alıştırma 1: Basit Skill

Bir Markdown dosyasının başlıklarını listeleyen bir skill yazın.

**İpuçları:**
- Skill adı: `md-headings`
- Script: `grep '^##'` veya Python ile
- Çıktı: başlık seviyesi ve metin

**Beklenen çözüm yapısı:**

```
md-headings/
├── SKILL.md
└── scripts/
    └── list-headings.sh
```

### Alıştırma 2: Dosya İşleme Skill'i

Bir dizindeki tüm `.log` dosyalarını tarayıp hata satırlarını bulan bir skill yazın.

**İpuçları:**
- `grep -r "ERROR\|FATAL"` veya `rg "ERROR"`
- Çıktı: dosya adı, satır numarası, hata mesajı

### Alıştırma 3: Karmaşık İş Akışı

Bir Git reposunda son 7 günde yapılan değişiklikleri özetleyen bir skill yazın.

**İpuçları:**
- `git log --since="7 days ago"`
- `git diff --stat`
- Çıktı: commit özeti, değişen dosyalar, eklenen/silinEN satırlar

### Alıştırma 4: Cross-Platform Skill

Hem Pi'de hem Claude Code'da çalışan, ortak `~/.agents/skills/` dizinine yerleştirilen bir skill yazın.

### Alıştırma 5: Hata Yönetimi

Bir skill yazın ve şu hata durumlarını ele alın:
1. Girdi dosyası yok
2. Girdi dosyası boş
3. Yetki reddi (permission denied)
4. Çıktı dizini yok (otomatik oluştur)

---

## 20. Hızlı Başvuru Kartı

### SKILL.md Şablonu

```markdown
---
name: skill-name
description: What this skill does and when to use it.
license: MIT
compatibility: Requirements (optional)
metadata:
  author: your-name
  version: "1.0"
---

# Skill Name

## Setup
```bash
# Dependencies
```

## Usage
```bash
scripts/tool.sh <input>
```

## Workflow
1. Step one
2. Step two
3. Step three

## Error Handling
| Error | Action |
|-------|--------|
| File not found | Ask user for correct path |

## References
See [details](references/details.md).
```

### Frontmatter Alanları

| Alan | Zorunlu? | Kısıtlar |
|:-----|:---------|:---------|
| `name` | Evet | 1-64 karakter, küçük harf + rakam + tire |
| `description` | Evet | 1-1024 karakter |
| `license` | Hayır | Lisans adı veya dosya referansı |
| `compatibility` | Hayır | 500 karakter, ortam gereksinimleri |
| `metadata` | Hayır | Anahtar-değer çiftleri |
| `allowed-tools` | Hayır | Uzay-ayrılmış araç listesi (deneysel) |
| `disable-model-invocation` | Hayır | true=skill gizli, sadece `/skill:ad` ile |

### Skills Komutları

```bash
/skill:skill-name              # Skill'i yükle ve çalıştır
/skill:skill-name arg1 arg2    # Argümanlarla çağır
pi --no-skills                 # Skills keşfini kapat
```

### Skills Dizin Konumları

```bash
~/.pi/agent/skills/              # Pi global
~/.agents/skills/                # Cross-client global
.pj/skills/                      # Pi proje
.agents/skills/                  # Cross-client proje
~/.claude/skills/                # Claude Code global
~/.codex/skills/                 # OpenAI Codex global
```

### Doğrulama

```bash
skills-ref validate ./my-skill
# veya manuel:
ls SKILL.md scripts/ references/
head -5 SKILL.md  # frontmatter kontrol
chmod +x scripts/*
```

### İletişim ve Kaynaklar

| Kaynak | Adres |
|:-------|:------|
| Spesifikasyon | [agentskills.io/specification](https://agentskills.io/specification) |
| Anthropic Skills | [github.com/anthropics/skills](https://github.com/anthropics/skills) |
| Pi Skills | [github.com/badlogic/pi-skills](https://github.com/badlogic/pi-skills) |
| Agent Skills GitHub | [github.com/agentskills/agentskills](https://github.com/agentskills/agentskills) |
| Discord | [discord.gg/MKPE9g8aUy](https://discord.gg/MKPE9g8aUy) |
| Pi Skills Dökümanı | `/usr/local/lib/node_modules/.../docs/skills.md` |

---

> **Hazırlanma Tarihi:** Temmuz 2026
