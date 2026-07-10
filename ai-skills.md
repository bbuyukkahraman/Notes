# AI'da Skills Kavramı: Köken, Amaç ve Gelişim

> **Skills (Yetenekler):** AI ajanlarına belirli görevleri yerine getirme yeteneği kazandıran, yeniden kullanılabilir, paketlenmiş bilgi ve talimat bütünü.

---

## İçindekiler

1. [Özet — 30 Saniyede](#1-özet--30-saniyede)
2. [Kavramın Kökeni: SayCan (Nisan 2022)](#2-kavramın-kökeni-saycan-nisan-2022)
3. [ChatGPT Plugins — İlk Ticari Skill Ekosistemi (Mart 2023)](#3-chatgpt-plugins--ilk-ticari-skill-ekosistemi-mart-2023)
4. [Voyager: Skill Library ile Sürekli Öğrenen Ajan (Mayıs 2023)](#4-voyager-skill-library-ile-sürekli-öğrenen-ajan-mayıs-2023)
5. [GPTs: OpenAI'in Skills Evrimi (Kasım 2023)](#5-gpts-openaiin-skills-evrimi-kasım-2023)
6. [Agent Skills Specification — Açık Standart (2024)](#6-agent-skills-specification--açık-standart-2024)
7. [Anthropic Skills for Claude (2025)](#7-anthropic-skills-for-claude-2025)
8. [Pi Agent Skills — Uygulamada Skills](#8-pi-agent-skills--uygulamada-skills)
9. [Skills vs Plugins vs Tools — Karşılaştırma](#9-skills-vs-plugins-vs-tools--karşılaştırma)
10. [Skills Format: SKILL.md Yapısı](#10-skills-format-skillmd-yapısı)
11. [Zaman Çizelgesi](#11-zaman-çizelgesi)
12. [Referanslar](#12-referanslar)

---

## 1. Özet — 30 Saniyede

AI'da "skills" kavramının kökeni üç farklı kaynağa dayanır:

| # | Çalışma / Ürün | Tarih | Katkı |
|---|---------------|-------|-------|
| 1 | **SayCan** (Google Robotics) | Nisan 2022 | Akademik: "pretrained skills" ile LLM + robot grounding |
| 2 | **ChatGPT Plugins** (OpenAI) | Mart 2023 | Ticari: ilk geniş ölçekli skill ekosistemi |
| 3 | **Voyager** (NVIDIA/MIT) | Mayıs 2023 | Akademik: "skill library" ile sürekli öğrenen ajan |
| 4 | **Agent Skills Spec** (Açık Standart) | 2024 | Standart: sağlayıcı-bağımsız skill paketleme |

**Fikir babası:** SayCan (Ahn et al., Google Robotics, 2022) — LLM'leri robotikte "pretrained skills" ile grounding yaparak kullanan ilk çalışma.

**Skills'in temel fikri:** Bir AI ajanına görevle ilgili talimatlar, kod, referanslar ve yapılandırmayı tek bir pakette ver ve ajanın bu paketi gerektiğinde yüklemesine izin ver. Böylece ajanın eğitim verisinde olmayan görevleri yerine getirmesi sağlanır.

---

## 2. Kavramın Kökeni: SayCan (Nisan 2022)

**"Do As I Can, Not As I Say: Grounding Language in Robotic Affordances"**

- **Yazarlar:** Michael Ahn, Anthony Brohan, Noah Brown, Chelsea Finn, Sergey Levine ve diğerleri (Google Robotics, 30+ yazar)
- **Yayın:** arXiv, 4 Nisan 2022
- **Künye:** arXiv:2204.01691

### Amaç

LLM'ler zengin dünya bilgisine sahiptir ancak **gerçek dünya deneyiminden yoksundur**. Bir robota "çay döküntüsünü temizle" dendiğinde, LLM mantıklı bir açıklama üretebilir ama robotun fiziksel yetenekleriyle uyuşmayan eylemler önerebilir. Amaç, LLM'nin semantik bilgisini robotun yapabildiği **skills (yetenekler)** ile grounding yapmaktır.

### Öneri

**SayCan (Say + Can):** LLM yüksek seviyeli planlama yapar ("say"), önceden eğitilmiş skill'ler (yetenek değer fonksiyonları) hangi eylemlerin **fiziksel olarak mümkün olduğunu** belirler ("can").

```
LLM: "Odayı temizle → süpür → paspasla → toz al"
                ↓
Skill value functions: "paspas şu an mümkün değil, el başka yerde"
                ↓
Uygulanabilir eylem: "süpür"
```

Bu, LLM ve düşük seviyeli robot yetenekleri arasında bir köprü oluşturur. "Pretrained skills" kavramı ilk kez burada LLM bağlamında kullanılmıştır.

### Referans

- **arXiv:** [https://arxiv.org/abs/2204.01691](https://arxiv.org/abs/2204.01691)
- **PDF:** [https://arxiv.org/pdf/2204.01691](https://arxiv.org/pdf/2204.01691)
- **Proje Sayfası:** [https://say-can.github.io/](https://say-can.github.io/)

---

## 3. ChatGPT Plugins — İlk Ticari Skill Ekosistemi (Mart 2023)

OpenAI, 23 Mart 2023'te **ChatGPT Plugins**'i duyurdu. Bu, AI asistanlar için ilk geniş ölçekli skill/eklenti ekosistemiydi.

- **Duyuru:** OpenAI Blog, 23 Mart 2023
- **İçerik:** ChatGPT'nin üçüncü taraf hizmetlere bağlanmasını sağlayan eklentiler
- **Örnekler:** Wolfram Alpha (matematik), Expedia (seyahat), Instacart (market alışverişi)

### Amaç

ChatGPT'nin yeteneklerini doğal eğitim sınırlarının ötesine taşımak: güncel bilgi, hesaplama, üçüncü taraf API'lere erişim.

### Öneri

Her plugin, bir API manifestosu (OpenAPI spec + AI description) ile tanımlanırdı. ChatGPT kullanıcının isteğine göre hangi plugin'in çağrılacağına karar verirdi. Bu, günümüz tool-calling'inin ve skills sistemlerinin temelini attı.

Plugins daha sonra **GPTs** (Kasım 2023) ve ardından OpenAI Skills'e evrildi.

### Referans

- **Duyuru (Wayback Machine):** [https://web.archive.org/web/20230324000000/https://openai.com/blog/chatgpt-plugins](https://web.archive.org/web/20230324000000/https://openai.com/blog/chatgpt-plugins)
- **OpenAI Plugins Dökümantasyonu:** [https://platform.openai.com/docs/plugins/introduction](https://platform.openai.com/docs/plugins/introduction)

---

## 4. Voyager: Skill Library ile Sürekli Öğrenen Ajan (Mayıs 2023)

**"Voyager: An Open-Ended Embodied Agent with Large Language Models"**

- **Yazarlar:** Guanzhi Wang, Yuqi Xie, Yunfan Jiang, Ajay Mandlekar, Chaowei Xiao, Yuke Zhu, Linxi Fan, Anima Anandkumar (NVIDIA, MIT, Caltech)
- **Yayın:** arXiv, 25 Mayıs 2023
- **Künye:** arXiv:2305.16291

### Amaç

Minecraft'ta insan müdahalesi olmadan sürekli keşfeden, çeşitli skill'ler edinen ve yeni keşifler yapan bir ajan yaratmak.

### Öneri

Voyager üç bileşenden oluşur:

1. **Otomatik müfredat (automatic curriculum):** Keşfi maksimize eden görev seçimi
2. **Skill library (beceri kütüphanesi):** Sürekli büyüyen, çalıştırılabilir kod parçalarından oluşan skill deposu
3. **Iterative prompting:** Çevresel geri bildirim ve hata mesajlarıyla program iyileştirme

**Skill library nasıl çalışır:**
- Her skill, çalıştırılabilir bir kod parçasıdır (Python)
- Skill'ler tanımlayıcılarına göre indexlenir ve gerektiğinde kütüphaneden çekilir
- Kodlar birleştirilebilir (compositional) — bir skill diğerini çağırabilir
- Kütüphane büyüdükçe ajanın yetenekleri katlanarak artar
- Catastrophic forgetting (önceden öğrenileni unutma) sorununu çözer

**Sonuçlar:** Voyager, önceki en iyi yöntemlerden 3.3 kat daha fazla eşya toplamış, 2.3 kat daha uzun mesafe seyahat etmiş ve teknoloji ağacında 15.3 kata kadar daha hızlı ilerlemiştir.

### Referans

- **arXiv:** [https://arxiv.org/abs/2305.16291](https://arxiv.org/abs/2305.16291)
- **PDF:** [https://arxiv.org/pdf/2305.16291](https://arxiv.org/pdf/2305.16291)
- **Proje Sayfası:** [https://voyager.minedojo.org/](https://voyager.minedojo.org/)
- **Kod:** [https://github.com/MineDojo/Voyager](https://github.com/MineDojo/Voyager)

---

## 5. GPTs: OpenAI'in Skills Evrimi (Kasım 2023)

OpenAI, Kasım 2023'teki ilk DevDay konferansında **GPTs**'i duyurdu. GPTs, ChatGPT Plugins'in evrilmiş haliydi.

- **Duyuru:** OpenAI DevDay, 6 Kasım 2023
- **İçerik:** Kullanıcıların kendi özel ChatGPT versiyonlarını oluşturması

### Amaç

Teknik bilgisi olmayan kullanıcıların bile doğal dilde kendi AI asistanlarını (skill'lerini) oluşturması. "Herhangi bir amaç için özelleştirilmiş ChatGPT."

### Öneri

GPTs Builder ile kullanıcı:
1. Doğal dilde GPT'sinin ne yapması gerektiğini anlatır
2. GPT otomatik olarak talimatlar, bilgi dosyaları ve tool tanımları oluşturur
3. İsteğe bağlı: API bağlantısı (Actions) eklenebilir
4. GPT Store'da yayımlanabilir veya özel kullanılabilir

Bu, "skills" konseptini milyonlarca kullanıcıya ulaştıran en önemli adımdır.

### Referans

- **Duyuru:** [https://openai.com/index/introducing-gpts/](https://openai.com/index/introducing-gpts/)
- **GPTs Dökümantasyonu:** [https://help.openai.com/en/articles/8554397-creating-a-gpt](https://help.openai.com/en/articles/8554397-creating-a-gpt)

---

## 6. Agent Skills Specification — Açık Standart (2024)

2024'te **agentskills.io** topluluğu, tüm AI ajan sağlayıcıları arasında skill'lerin taşınabilir olmasını sağlayan açık bir standart yayınladı.

- **Web:** [https://agentskills.io/specification](https://agentskills.io/specification)
- **GitHub:** [https://github.com/agentskills/agentskills](https://github.com/agentskills/agentskills)
- **Discord:** [https://discord.gg/MKPE9g8aUy](https://discord.gg/MKPE9g8aUy)

### Amaç

Her AI ajanının (Claude Code, pi, Codex, Cline, vs.) kendi skill formatı vardı. Standart, bu skill'lerin herhangi bir ajanla çalışmasını sağlamayı hedefler.

### Öneri

**SKILL.md** dosyası merkezde olmak üzere basit bir dizin yapısı:

```
my-skill/
├── SKILL.md          # Zorunlu: frontmatter + talimatlar
├── scripts/          # Yardımcı scriptler
│   └── process.sh
└── references/       # Detaylı dökümantasyon
    └── guide.md
```

**SKILL.md frontmatter:**

| Alan | Zorunlu | Açıklama |
|------|---------|----------|
| `name` | Evet | 1-64 karakter, küçük harf, rakam, tire |
| `description` | Evet | Ne işe yarar, ne zaman kullanılır (max 1024 karakter) |
| `license` | Hayır | Lisans bilgisi |
| `compatibility` | Hayır | Ortam gereksinimleri |
| `metadata` | Hayır | Anahtar-değer çiftleri |
| `allowed-tools` | Hayır | Ön onaylı araç listesi |
| `disable-model-invocation` | Hayır | `true` ise skill sadece `/skill:name` ile çağrılır |

### Referans

- **Specification:** [https://agentskills.io/specification](https://agentskills.io/specification)
- **GitHub:** [https://github.com/agentskills/agentskills](https://github.com/agentskills/agentskills)

---

## 7. Anthropic Skills for Claude (2025)

Anthropic, Claude platformunda **skills** özelliğini ekledi. Skills, Claude'un yeteneklerini belirli görevler için genişletmek üzere kullanılır.

- **Dökümantasyon:** [https://docs.anthropic.com/en/docs/build-with-claude/skills](https://docs.anthropic.com/en/docs/build-with-claude/skills)

### Amaç

Claude'un belirli bir görevde (ör: kod inceleme, dosya arama, web araması) uzmanlaşması için talimatlar ve yapılandırma sağlamak.

### Öneri

Anthropic skills, API'de `skills` parametresi ile gönderilir:

```python
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    skills=["web_search", "code_review"],
    messages=[...]
)
```

Skills, mevcut tool-calling ve system prompt yönetiminin üzerine inşa edilmiş daha yüksek seviyeli bir soyutlamadır.

### Referans

- **Skills Dökümantasyonu:** [https://docs.anthropic.com/en/docs/build-with-claude/skills](https://docs.anthropic.com/en/docs/build-with-claude/skills)
- **Skills API Referansı:** [https://docs.anthropic.com/en/docs/build-with-claude/skills/quickstart](https://docs.anthropic.com/en/docs/build-with-claude/skills/quickstart)

---

## 8. Pi Agent Skills — Uygulamada Skills

Pi (bu kodlama ajanı), Agent Skills standardını uygulayan bir harness'tir.

### Çalışma Prensibi

1. **Başlangıçta:** Pi, skill dizinlerini tarar, isim ve açıklamaları çıkarır
2. **Sistem prompt'unda:** Kullanılabilir skill'ler XML formatında listelenir
3. **Talep üzerine:** Görev eşleştiğinde, ajan `read` ile SKILL.md'nin tamamını yükler (progressive disclosure)
4. **Çalıştırma:** Ajan, skill talimatlarını izler, scriptleri ve referansları kullanır

### Skill Komutları

```bash
/skill:brave-search           # Skill'i yükle ve çalıştır
/skill:caveman ultra          # Skill'i argümanla çağır
```

### Referans

- **Pi Skills Dökümantasyonu:** `/usr/local/lib/node_modules/@earendil-works/pi-coding-agent/docs/skills.md`
- **Örnek Skill:** `/root/.pi/agent/skills/caveman/SKILL.md`

---

## 9. Skills vs Plugins vs Tools — Karşılaştırma

Bu üç kavram sıkça karıştırılır. İşte net ayrım:

| Özellik | **Skills** | **Tools (Function Calling)** | **Plugins** |
|----------|-----------|------------------------------|-------------|
| **Nedir** | Paketlenmiş talimatlar + kod + referanslar | JSON Schema ile tanımlanmış tek fonksiyon | Üçüncü taraf API bağlantısı |
| **Kapsam** | Çok geniş — tüm bir iş akışı | Dar — tek bir işlem | Orta — belirli bir hizmet |
| **İçerir** | Talimatlar, scriptler, referanslar, config | Sadece fonksiyon imzası | API spec + auth + description |
| **Nasıl kullanılır** | Progressive disclosure: önce tarif, sonra detay | Ajan çağırır, siz çalıştırırsınız | Ajan doğrudan API'yi çağırır |
| **Format** | SKILL.md (markdown) | JSON Schema | OpenAPI / Manifest |
| **Örnek** | "Code review skill" — tüm review workflow'u | `get_weather(location)` | Wolfram Alpha plugin |
| **Yeniden kullanım** | Cross-harness (standart varsa) | Cross-provider (JSON Schema) | Sadece ChatGPT |

### Somut Örnek

```
Kullanıcı: "Bu commit'i review et"

Senaryo — Tools ile:
  → read_file(diff) + search_similar_bugs(query) + suggest_fix(code)
  → Her adım ayrı tool çağrısı, ayrı JSON

Senaryo — Skills ile:
  → Skill: code-review → 
     SKILL.md içinde: "Önce diff'i oku, sonra güvenlik taraması yap,
     ardından benzer hataları ara, sonuçları commit mesajı formatında raporla"
  → Tek bir soyutlama, içinde birden çok tool kullanabilir
```

---

## 10. Skills Format: SKILL.md Yapısı

Agent Skills standardına göre eksiksiz bir SKILL.md örneği:

```markdown
---
name: code-review
description: >
  Git commit'lerini ve PR'ları review eder. Güvenlik açığı taraması, 
  kod kalitesi analizi ve test kapsamı kontrolü yapar. 
  Kullanıcı "bu commit'i incele" dediğinde aktifleşir.
license: MIT
compatibility: git >= 2.0, Python >= 3.10
---

# Code Review Skill

## Setup

İlk kullanım öncesi:

```bash
pip install -r scripts/requirements.txt
```

## Usage

```bash
# Tek commit review
./scripts/review.sh <commit-hash>

# PR review (tüm commit'ler)
./scripts/review_pr.sh <pr-number>
```

## Workflow

1. `git diff` ile değişiklikleri al
2. Güvenlik taraması çalıştır (semgrep, bandit)
3. Kod kalitesi kontrolü (linter)
4. Test kapsamı raporu
5. Özet ve önerileri markdown olarak döndür

## Referanslar

Detaylı kurallar için [rules.md](references/rules.md) dosyasına bak.
```

---

## 11. Zaman Çizelgesi

```
2022-04 ── SayCan (Google Robotics) — "pretrained skills" ile LLM grounding
               ↓
2023-03 ── ChatGPT Plugins (OpenAI) — ilk ticari skill ekosistemi
               ↓
2023-05 ── Voyager (NVIDIA/MIT) — skill library ile lifelong learning
               ↓
2023-11 ── GPTs (OpenAI) — kullanıcıların kendi skill'lerini oluşturması
               ↓
2024    ── Agent Skills Specification (agentskills.io) — açık standart
               ↓
2024    ── Pi Agent Skills — standart uygulaması
               ↓
2025    ── Anthropic Skills for Claude — platform skills sistemi
```

---

## 12. Referanslar

### Akademik Makaleler

| Çalışma | Bağlantı |
|---------|----------|
| **SayCan** (Google Robotics, 2022) | [arXiv:2204.01691](https://arxiv.org/abs/2204.01691) |
| **Voyager** (NVIDIA/MIT, 2023) | [arXiv:2305.16291](https://arxiv.org/abs/2305.16291) |

### Platform Dökümantasyonları

| Platform | Bağlantı |
|----------|----------|
| OpenAI Function Calling → Skills evrimi | [https://platform.openai.com/docs/guides/function-calling](https://platform.openai.com/docs/guides/function-calling) |
| OpenAI GPTs | [https://openai.com/index/introducing-gpts/](https://openai.com/index/introducing-gpts/) |
| Anthropic Skills | [https://docs.anthropic.com/en/docs/build-with-claude/skills](https://docs.anthropic.com/en/docs/build-with-claude/skills) |
| Pi Skills | `/usr/local/lib/node_modules/@earendil-works/pi-coding-agent/docs/skills.md` |

### Açık Standartlar

| Standart | Bağlantı |
|----------|----------|
| Agent Skills Specification | [https://agentskills.io/specification](https://agentskills.io/specification) |
| GitHub Repository | [https://github.com/agentskills/agentskills](https://github.com/agentskills/agentskills) |

### Diğer

| Kaynak | Bağlantı |
|--------|----------|
| ChatGPT Plugins Duyurusu | [https://openai.com/blog/chatgpt-plugins](https://openai.com/blog/chatgpt-plugins) |
| Voyager Proje Sayfası | [https://voyager.minedojo.org/](https://voyager.minedojo.org/) |
| Voyager GitHub | [https://github.com/MineDojo/Voyager](https://github.com/MineDojo/Voyager) |
| SayCan Proje Sayfası | [https://say-can.github.io/](https://say-can.github.io/) |

---

> **Hazırlanma Tarihi:** Temmuz 2026
