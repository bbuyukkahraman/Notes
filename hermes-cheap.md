# 🏆 Hermes-Cheap: Süper Cimri Hermes Agent Rehberi

> Hermes AI Agent v0.18.2'yi **ilk sorguda ~4.2K token** tüketecek şekilde optimize etme kılavuzu.
> Orijinal ~20-30K token'dan **%80+ tasarruf**.

## 📊 Kazanım Tablosu

| Bileşen | Önce | Sonra | Kazanç |
|---|---|---|---|
| Skills disk | 6.7 MB | 76 KB | **%99** |
| Skills snapshot (prompt) | 42 KB | 0.5 KB | **%99** |
| Tool schemas | ~70 KB | 7.2 KB | **%90** |
| System prompt | ~50 KB | 5.2 KB | **%90** |
| Aktif tool sayısı | 30+ | 10 | **%67** |
| **İlk sorgu token** | **~20-30K** | **~4.2K** | **~%80** |

---

## 🔧 1. Runtime Monkey-Patch (En Önemli Adım)

Tool description'larını source code'a dokunmadan, runtime'da otomatik kırpan wrapper.
Hermes güncellemelerinde **silinmez** — kaynak koddan bağımsız çalışır.

### Kurulum

`~/.local/bin/hermes-cheap` dosyasını oluştur:

```python
#!/usr/bin/env python3
"""Hermes-Cheap: Runtime tool description minimizer."""
import os, sys
HERMES_AGENT = os.path.expanduser("~/.hermes/hermes-agent")
sys.path.insert(0, HERMES_AGENT)

import tools.registry as reg_module
_original_register = reg_module.ToolRegistry.register

def _compress_desc(desc: str, maxlen: int = 60) -> str:
    if not isinstance(desc, str) or len(desc) <= maxlen:
        return desc
    short = desc.split(".")[0].split("\n")[0][:maxlen].strip().rstrip("., ")
    return short + "." if not short.endswith(".") else short

def _short_register(self, name, toolset, schema, handler, **kw):
    schema = dict(schema)
    schema["description"] = _compress_desc(schema.get("description", ""), 60)
    params = schema.get("parameters", {})
    if isinstance(params, dict):
        for pdef in params.get("properties", {}).values():
            if isinstance(pdef, dict) and "description" in pdef:
                pdef["description"] = _compress_desc(pdef["description"], 50)
    return _original_register(self, name, toolset, schema, handler, **kw)

reg_module.ToolRegistry.register = _short_register

# Strip emoji
_orig_entry = reg_module.ToolEntry
class _ShortEntry(_orig_entry):
    def __init__(self, *args, **kw):
        kw["emoji"] = ""
        super().__init__(*args, **kw)
reg_module.ToolEntry = _ShortEntry

from hermes_cli.main import main
sys.exit(main())
```

**Çalıştır:** `chmod +x ~/.local/bin/hermes-cheap`

### Güncelleme Sonrası

Hiçbir şey yapmana gerek yok — monkey-patch güncellemelerden etkilenmez.

---

## ⚙️ 2. Konfigürasyon Optimizasyonu

`~/.hermes/config.yaml` dosyasını aşağıdaki gibi sadeleştir:

```yaml
model:
  default: deepseek-v4-flash
  provider: deepseek
  base_url: https://api.deepseek.com/v1
terminal:
  backend: local
  cwd: .
  timeout: 300
compression:
  enabled: true
  threshold: 0.3           # %50 → %30 (daha erken devreye girer)
  target_ratio: 0.1         # %20 → %10 (daha agresif sıkıştırma)
  protect_last_n: 10        # son 20 → 10 mesaj korunsun
  protect_first_n: 2        # ilk 3 → 2 mesaj korunsun
prompt_caching:
  cache_ttl: 5m
memory:
  memory_enabled: false      # Kalıcı hafıza kapalı
  user_profile_enabled: false
agent:
  max_turns: 25              # 60 → 25
  tool_use_enforcement: false
  task_completion_guidance: false
  parallel_tool_call_guidance: false
  environment_probe: false   # Python ortam sorgulaması kapalı
  skip_context_files: true   # AGENTS.md/.cursorrules taranmaz
tool_loop_guardrails:
  hard_stop_enabled: true    # Sonsuz döngüyü engelle (güvenlik)
  warn_after:
    exact_failure: 3
    same_tool_failure: 5
    idempotent_no_progress: 3
  hard_stop_after:
    exact_failure: 8
    same_tool_failure: 12
    idempotent_no_progress: 8
platform_toolsets:
  cli:
    - terminal
    - file
    - web
    - clarify
    - skills
display:
  compact: true
  personality: concise       # Daha kısa cevaplar
  tool_progress: minimal
  show_reasoning: false
  busy_input_mode: steer     # Araç çalışırken yönlendirme yapabilme
  streaming: true
```

### Kapatılan Guidance Bloklarının Açıklaması

| Blok | Ne işe yarar | Token maliyeti |
|---|---|---|
| `tool_use_enforcement` | "Tool kullan, anlatma" uyarısı | ~500 token |
| `task_completion_guidance` | "İşi bitir, yarıda bırakma" uyarısı | ~400 token |
| `parallel_tool_call_guidance` | "Paralel tool çağrısı yap" uyarısı | ~300 token |
| `environment_probe` | Python/pip/uv ortam sorgusu | ~200 token |

---

## 🗑️ 3. Skills Temizliği

### Built-in Skills Sync'i Kapat

```bash
hermes skills opt-out
```

Bu komut `~/.hermes/.no-bundled-skills` marker'ını oluşturur.
Hermes güncellemelerinde skill'ler geri gelmez.

### Gereksiz Skill Kategorilerini Sil

Sadece ihtiyacın olanları tut:

```bash
cd ~/.hermes/skills
# Tüm gereksiz kategoriler
rm -rf autonomous-ai-agents creative dogfood email github media \
       mlops productivity research social-media computer-use \
       apple data-science note-taking smart-home

# software-development içinde de gereksiz olanlar
rm -rf software-development/node-inspect-debugger
rm -rf software-development/python-debugpy
rm -rf software-development/simplify-code

# Kalanlar (sadece faydalı workflow'lar):
#   test-driven-development, systematic-debugging, plan,
#   spike, requesting-code-review, hermes-agent-skill-authoring

# Snapshot'ı sıfırla
rm -f ~/.hermes/.skills_prompt_snapshot.json
```

**Kalanlar (6 SKILL.md, 76 KB):**
```
software-development/
├── test-driven-development
├── systematic-debugging
├── plan
├── spike
├── requesting-code-review
└── hermes-agent-skill-authoring
```

**Gereksiz olanlar (silinenler):**
```
apple/           (apple-notes, reminders, findmy, imessage)
data-science/    (jupyter)
note-taking/     (obsidian)
smart-home/      (openhue)
node-inspect-debugger, python-debugpy, simplify-code
```

### Skills Snapshot

Snapshot otomatik yeniden oluşur. Prompt'ta sadece skill adları ve kısa açıklamaları yer alır:
- 17 kategori: **42 KB**
- 5 kategori: **~2 KB**
- 1 kategori (sadece software-development): **0.5 KB**

---

## 🎭 4. Token Dışı İyileştirmeler (Kullanıcı Deneyimi)

### `display.personality: concise`
Sisteme **"Keep responses brief and to the point"** talimatını ekler.
- **Maliyet:** ~15 token (input)
- **Kazanç:** Çok daha kısa cevaplar → completion token'larında büyük tasarruf
- **Net:** Pozitif ✅

### `display.busy_input_mode: steer`
**En güçlü özellik!** Agent araç çalıştırırken (terminal, web, dosya işlemleri) 
sen yazmaya devam edebilirsin. Yazdıkların **mid-turn steering** olarak agent'a 
iletilir — anında yönlendirme yapabilirsin.

Örnek: Agent bir dosyayı analiz ederken sen "şu fonksiyona da bak" yazarsın,
o anki işi bitince hemen yeni talimatı alır. Beklemene gerek kalmaz.

### `tool_loop_guardrails.hard_stop_enabled: true`
Agent aynı tool'da tekrar tekrar hata alırsa (örn. internet yok, terminal kilitli),
**sonsuz döngüyü otomatik keser** ve sana bildirir.
- 3 başarısız deneme → uyarı
- 8 başarısız deneme → hard stop, sebebini açıkla

### `terminal.timeout: 300`
Uzun süren build'ler, testler, deployment'lar için 5 dakika.
180 saniye bazen yetmez (özellikle büyük npm/pip install'ler).

---

## 📝 5. SOUL.md (Kimlik + Davranış Kuralları)

`~/.hermes/SOUL.md` hem kimlik hem de agent'ın davranış kurallarını belirler.
Sadece "kimsin" değil, "nasıl davranmalısın" da burada.

### Önerilen Versiyon

```markdown
You are Hermes, a concise AI assistant.
- Prefer terminal, file, and web tools over guessing
- Turkish query → Turkish response; English → English
- Verify results proactively (run the code, check output)
- When stuck: retry once with a different approach, then report clearly
- Keep responses brief and direct
```

**Neden bu kadar etkili:**
- "retry once differently" → tek hata sonrası pes etmez, farklı dener
- "verify proactively" → "şimdi çalıştırayım mı?" sorusunu ortadan kaldırır
- "Turkish → Turkish" → Türkçe sorunca İngilizce cevap vermez
- Maliyet: 311 byte (~75 token) — çok küçük bir maliyet

### Ultra Minimal Versiyon (Token Kritikse)

```markdown
You are Hermes, a concise AI assistant. Be brief and direct.
Prefer tools over guessing. Turkish → Turkish, English → English.
```

138 byte. Sadece temel yönergeler.

---

## 📁 6. Proje Context Dosyası (.hermes.md)

Proje bazlı bilgi vermek için projenin köküne `.hermes.md` koy.
Hermes git root'a kadar otomatik bulur.

### Önerilen Şablon

```markdown
# Project: <name>
Tech stack: <languages, frameworks>
Conventions: <style rules, indent, naming>
Commands:
  build:   <build command>
  test:    <test command>
  lint:    <lint command>
```

### Notes Reposu İçin Örnek

```markdown
# Notes
Personal notes, guides, and references repository.

## Commands
  preview:  (none — pure Markdown)
  publish:  git push origin master

## Conventions
- Markdown files (.md) with UTF-8
- Turkish content for personal notes
- File names: kebab-case
```

Maliyet: ~300 byte (~75 token) — proje bağlamı için çok değerli.

### Akıllı Kullanım

Shell fonksiyonu (bkz. 🔄 7.) `.hermes.md` olan dizinlerde **otomatik context açar**,
olmayanlarda token tasarrufu için kapalı tutar. Manuel config değiştirmene gerek kalmaz.

---

## 🧹 7. .env Temizliği

`~/.hermes/.env` dosyasında sadece gerçekten kullanılan değişkenleri tut:

```bash
DEEPSEEK_API_KEY="sk-..."
# TERMINAL_MODAL_IMAGE=...   # sadece container kullanıyorsan gerekli
```

Gereksiz environment variable'lar (BROWSER_*, VISION_*, MOA_*, IMAGE_*, TERMINAL_*) 
tool'ları etkilemez ama kafa karıştırır.

---

## 🔄 7. Akıllı Shell Entegrasyonu

### Tavsiye: Fonksiyon (Alias Yerine)

Alias yerine **akıllı shell fonksiyonu** — CWD'de `.hermes.md`, `AGENTS.md`, `.cursorrules` varsa
otomatik context'i açar, yoksa token tasarrufu için kapalı tutar.

```bash
# Smart Hermes: auto-detects context files in CWD
hermes() {
  local has_context=0
  local dir="$PWD"
  while [ "$dir" != "/" ]; do
    if [ -f "$dir/.hermes.md" ] || [ -f "$dir/HERMES.md" ] || [ -f "$dir/AGENTS.md" ] || [ -f "$dir/.cursorrules" ]; then
      has_context=1
      break
    fi
    dir="$(dirname "$dir")"
  done

  if [ "$has_context" = "1" ]; then
    HERMES_CONTEXT_FILES=1 hermes-cheap "$@"
  else
    hermes-cheap "$@"
  fi
}
```

**Nasıl çalışır:**
- Proje dizininde `.hermes.md` varsa → context aktif, proje bilgisi prompt'a eklenir
- Rastgele bir dizinde → context kapalı, token tasarrufu
- Git root'a kadar arar (alt dizinler dahil)

### Basit Alternatif: Alias

İsteyen sadece alias kullanabilir:

```bash
alias hermes="hermes-cheap"
```

---

## 🧠 9. Mimari Bilgisi (Neden Çalışıyor)

### Sistem Promptu 3 Kademeli

| Kademe | İçerik | Değişkenlik | Önbellek |
|---|---|---|---|
| **stable** | SOUL.md + tool guidance + skills index + env/platform hints | Session boyu sabit | ✅ Prompt cache |
| **context** | AGENTS.md, .cursorrules, caller system_message | Session başı değişir | ❌ |
| **volatile** | Memory + USER.md + timestamp | Her tur değişir | ❌ |

### Prompt Caching (DeepSeek)

System prompt **byte-stable** olduğu için DeepSeek'in prefix caching'inden faydalanır:
- **İlk sorgu:** ~4.5K token (full price)
- **Sonraki sorgular:** ~0.5-1K token (sadece yeni mesaj, system prompt cached)

---

## 🚫 10. Kapatılan Toolset'ler

| Toolset | Tool'lar | Neden kapatıldı |
|---|---|---|
| `browser` | navigate, click, type, scroll... | Web sayfası etkileşimi gerekmiyor |
| `image_gen` | image_generate | Görsel üretim gerekmiyor |
| `computer_use` | computer_use | Masaüstü kontrolü gerekmiyor |
| `kanban` | kanban_show, complete... | Multi-agent koordinasyon gerekmiyor |
| `cronjob` | cronjob | Zamanlanmış görev gerekmiyor |
| `tts` | text_to_speech | Ses sentezi gerekmiyor |
| `memory` | memory | Kalıcı hafıza kapalı |
| `session_search` | session_search | Geçmiş konuşma arama kapalı |
| `vision` | vision_analyze | Görsel analiz gerekmiyor |
| `code_execution` | execute_code | Terminal alternatifi var |
| `delegation` | delegate_task | Alt-agent gerekmiyor |
| `discord` / `homeassistant` vs. | platform tools | Kullanılmıyor |

**Kalanlar (10 tool):**
- **terminal:** terminal, process
- **file:** read_file, write_file, patch, search_files
- **web:** web_search, web_extract
- **clarify:** clarify
- **skills:** skills_list, skill_view, skill_manage

---

## 📊 11. Token Hesaplama Pratiği

```bash
# Prompt boyutunu ölç
hermes prompt-size

# DeepSeek tokenizer için kabaca:
# - İngilizce metin: ~4 karakter/token
# - JSON schema: ~2.5-3 karakter/token
# - Türkçe metin: ~2-3 karakter/token
```

---

## 🔄 12. Güncelleme Sonrası Yapılması Gerekenler

1. `hermes-cheap` wrapper'ı **etkilenmez** — çalışmaya devam eder
2. Skills sync **kapalı** kalır (`.no-bundled-skills` marker'ı duruyorsa)
3. Config dosyası **manuel** güncellenir — `hermes config migrate` çalıştır, yeni anahtarları kontrol et
4. Tool schema boyutunu kontrol et: `hermes prompt-size`

---

## 🎯 13. Özet: Yapılacaklar Listesi (Yeni Kurulumda)

```bash
# 1. Wrapper
chmod +x ~/.local/bin/hermes-cheap

# 2. Shell fonksiyonu (önerilen, alias yerine)
# .zshrc'ye akıllı fonksiyonu ekle (bkz. bölüm 7)

# 3. Skills
hermes skills opt-out
rm -rf ~/.hermes/skills/{autonomous-ai-agents,creative,dogfood,email,github,media,mlops,productivity,research,social-media,computer-use,apple,data-science,note-taking,smart-home}
rm -rf ~/.hermes/skills/software-development/{node-inspect-debugger,python-debugpy,simplify-code}
rm -f ~/.hermes/.skills_prompt_snapshot.json

# 4. Config (yukarıdaki gibi)
# 5. SOUL.md (iyileştirilmiş versiyon)
# 6. .hermes.md (proje bazlı, isteğe bağlı)
# 7. .env (sadece API key)
# 8. source ~/.zshrc
# 9. Test: hermes-cheap prompt-size
```

---

> **Sonuç: İlk sorgu ~4.2K token. Orijinalin ~%15'i.**
> **Bonus: Akıllı shell fonksiyonu ile proje context'i otomatik, token israfı yok.**
