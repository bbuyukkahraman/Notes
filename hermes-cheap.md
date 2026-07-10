# 🏆 Hermes-Cheap: Süper Cimri Hermes Agent Rehberi

> Hermes AI Agent v0.18.2'yi **ilk sorguda ~4.5K token** tüketecek şekilde optimize etme kılavuzu.
> Orijinal ~20-30K token'dan **%75+ tasarruf**.

## 📊 Kazanım Tablosu

| Bileşen | Önce | Sonra | Kazanç |
|---|---|---|---|
| Skills disk | 6.7 MB | 172 KB | **%97** |
| Skills snapshot (prompt) | 42 KB | ~2 KB | **%95** |
| Tool schemas | ~70 KB | 7.2 KB | **%90** |
| System prompt | ~50 KB | 6.4 KB | **%87** |
| Aktif tool sayısı | 30+ | 10 | **%67** |
| **İlk sorgu token** | **~20-30K** | **~4.5K** | **~%80** |

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
  timeout: 180
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
platform_toolsets:
  cli:
    - terminal
    - file
    - web
    - clarify
    - skills
display:
  compact: true
  tool_progress: minimal
  show_reasoning: false
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
# Silinecekler (ihtiyacına göre değiştir)
rm -rf autonomous-ai-agents creative dogfood email github media \
       mlops productivity research social-media computer-use
# Snapshot'ı sıfırla
rm -f ~/.hermes/.skills_prompt_snapshot.json
```

**Önerilen tutulacaklar:** `apple`, `data-science`, `note-taking`, `smart-home`, `software-development`

### Skills Snapshot

Snapshot otomatik yeniden oluşur. Prompt'ta sadece skill adları ve kısa açıklamaları yer alır:
- 17 kategorili snapshot: **42 KB**
- 5 kategorili snapshot: **~2 KB**

---

## 📝 4. SOUL.md (Kimlik)

`~/.hermes/SOUL.md` dosyasını ultra kısa yap:

```markdown
You are Hermes, a concise AI assistant. Help users efficiently by using tools
for file ops, terminal, web, and code. Be brief and direct.
```

138 byte ile yetin. Orijinali 514 byte'dı.

---

## 🧹 5. .env Temizliği

`~/.hermes/.env` dosyasında sadece gerçekten kullanılan değişkenleri tut:

```bash
DEEPSEEK_API_KEY="sk-..."
# TERMINAL_MODAL_IMAGE=...   # sadece container kullanıyorsan gerekli
```

Gereksiz environment variable'lar (BROWSER_*, VISION_*, MOA_*, IMAGE_*, TERMINAL_*) 
tool'ları etkilemez ama kafa karıştırır.

---

## 🔄 6. Alias

`.zshrc`'ye ekle:

```bash
alias hermes="hermes-cheap"
```

Böylece `hermes chat` yazdığında otomatik cimri modda çalışır.
Orijinal Hermes'e ihtiyacın olursa: `~/.local/bin/hermes chat`

---

## 🧠 Mimari Bilgisi (Neden Çalışıyor)

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

## 🚫 Kapatılan Toolset'ler

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

## 📊 Token Hesaplama Pratiği

```bash
# Prompt boyutunu ölç
hermes prompt-size

# DeepSeek tokenizer için kabaca:
# - İngilizce metin: ~4 karakter/token
# - JSON schema: ~2.5-3 karakter/token
# - Türkçe metin: ~2-3 karakter/token
```

---

## 🔄 Güncelleme Sonrası Yapılması Gerekenler

1. `hermes-cheap` wrapper'ı **etkilenmez** — çalışmaya devam eder
2. Skills sync **kapalı** kalır (`.no-bundled-skills` marker'ı duruyorsa)
3. Config dosyası **manuel** güncellenir — `hermes config migrate` çalıştır, yeni anahtarları kontrol et
4. Tool schema boyutunu kontrol et: `hermes prompt-size`

---

## 🎯 Özet: Yapılacaklar Listesi (Yeni Kurulumda)

```bash
# 1. Wrapper
chmod +x ~/.local/bin/hermes-cheap

# 2. Alias
echo 'alias hermes="hermes-cheap"' >> ~/.zshrc

# 3. Skills
hermes skills opt-out
rm -rf ~/.hermes/skills/{autonomous-ai-agents,creative,dogfood,email,github,media,mlops,productivity,research,social-media,computer-use}
rm -f ~/.hermes/.skills_prompt_snapshot.json

# 4. Config (yukarıdaki gibi)
# 5. SOUL.md (kısa versiyon)
# 6. .env (sadece API key)
# 7. source ~/.zshrc
# 8. Test: hermes-cheap prompt-size
```

---

> **Sonuç: İlk sorgu ~4.5K token. Orijinalin ~%20'si.**
