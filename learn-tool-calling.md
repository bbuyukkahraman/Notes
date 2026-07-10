# Tool-Calling: Sıfırdan Ustalığa Eksiksiz Eğitim Materyali

> **Seviye:** Başlangıç → İleri · **Süre:** Kapsamlı · **Format:** Teori + Kod + Pratik

---

## İçindekiler

1. [Tool-Calling Nedir? — 30 Saniyede Özet](#1-tool-calling-nedir--30-saniyede-özet)
2. [Neden Tool-Calling? — Problemin Anatomisi](#2-neden-tool-calling--problemin-anatomisi)
3. [Temel Mimari: Agent Loop](#3-temel-mimari-agent-loop)
4. [Tool Tanımlama — JSON Schema ile](#4-tool-tanımlama--json-schema-ile)
5. [OpenAI Function Calling: Adım Adım](#5-openai-function-calling-adım-adım)
6. [Anthropic Tool Use: Adım Adım](#6-anthropic-tool-use-adım-adım)
7. [Çoklu Tool ve Paralel Çağrılar](#7-çoklu-tool-ve-paralel-çağrılar)
8. [Tool Choice — Modelin Tool Çağırma Davranışını Kontrol Etme](#8-tool-choice--modelin-tool-çağırma-davranışını-kontrol-etme)
9. [Strict / Constrained Tool Calling](#9-strict--constrained-tool-calling)
10. [Hata Yönetimi ve Fallback Stratejileri](#10-hata-yönetimi-ve-fallback-stratejileri)
11. [Prompt Caching ile Tool Calling Optimizasyonu](#11-prompt-caching-ile-tool-calling-optimizasyonu)
12. [Streaming ile Tool Calling](#12-streaming-ile-tool-calling)
13. [İleri Desenler: ReAct, Chain-of-Tool, Tool Loops](#13-i̇leri-desenler-react-chain-of-tool-tool-loops)
14. [Gerçek Dünya Vaka Çalışmaları](#14-gerçek-dünya-vaka-çalışmaları)
15. [Sık Yapılan Hatalar ve Çözümleri](#15-sık-yapılan-hatalar-ve-çözümleri)
16. [Alıştırmalar](#16-alıştırmalar)
17. [Referanslar ve İleri Okuma](#17-referanslar-ve-i̇leri-okuma)

---

## 1. Tool-Calling Nedir? — 30 Saniyede Özet

Tool-calling (veya function calling), bir dil modelinin (LLM) sizin tanımladığınız harici fonksiyonları/araçları ne zaman çağıracağını **kendi karar vermesi** ve bu çağrılar için **yapılandırılmış JSON çıktısı** üretmesidir.

```python
# En basit hali — bir tool tanımı ve çağrısı
tool = {
    "type": "function",
    "function": {
        "name": "get_weather",
        "description": "Verilen şehir için hava durumunu al",
        "parameters": {
            "type": "object",
            "properties": {
                "location": {"type": "string"}
            },
            "required": ["location"]
        }
    }
}

# Model ne zaman bu tool'u çağırması gerektiğine karar verir
# Çıktı: {"name": "get_weather", "arguments": '{"location": "Istanbul"}'}
```

**Kritik nokta:** Siz tool'u tanımlarsınız, model **hangi tool'u, hangi argümanlarla, ne zaman çağıracağına** karar verir. Sizin kodunuz bu çağrıyı alır, çalıştırır ve sonucu modele geri gönderir.

---

## 2. Neden Tool-Calling? — Problemin Anatomisi

### LLM'lerin Doğal Sınırları

| Sınırlılık | Örnek | Tool-Calling Çözümü |
|-------------|-------|--------------------|
| **Güncel bilgi yok** | "Bugün hava kaç derece?" | Hava durumu API'si |
| **Matematik zayıf** | "29384 × 48721 = ?" | Hesap makinesi tool'u |
| **Veritabanı sorgulama** | "En çok satan ürün hangisi?" | SQL sorgu tool'u |
| **Dosya sistemi erişimi** | "log.txt'yi oku" | Dosya okuma tool'u |
| **Halüsinasyon** | var olmayan bir makaleye atıf | Arama motoru tool'u |

### Neden Tool-Calling, Neden Prompt Mühendisliği Değil?

Prompt ile tool kullanımını tarif etmek (ör: _"JSON çıktısı ver"_) güvenilmezdir. Tool-calling'de:

1. **Model tool kullanımı için fine-tune edilmiştir** — prompt'taki talimata güvenmez
2. **JSON Schema validation** — modelin çıktısı doğrudan doğrulanabilir
3. **Structured output** garantisi — `strict: true` ile zorunlu kılınabilir
4. **Stop reason** (`tool_use` / `stop`) — loop mantığı net

---

## 3. Temel Mimari: Agent Loop

Tool-calling'in kalbi **agent loop** (veya **tool loop**) adı verilen döngüdür:

```
┌─────────────────────────────────────────────────────────┐
│                    1. Kullanıcı sorgusu                  │
│         "İstanbul'da hava nasıl, 3 günlük?"             │
└───────────────────────┬─────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│              2. Model + Tool Tanımları                  │
│         (messages + tools parametresi)                   │
└───────────────────────┬─────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│           3. Model Yanıtı — İHTİMAL                      │
│                                                          │
│   A) stop_reason = "stop"                                │
│      → Doğrudan metin yanıt, BİTTİ                       │
│                                                          │
│   B) stop_reason = "tool_use" / "function_call"          │
│      → tool_use blokları içerir                          │
└───────────────────────┬─────────────────────────────────┘
                        │ (B seçeneği)
                        ▼
┌─────────────────────────────────────────────────────────┐
│         4. SİZİN KODUNUZ — Tool Çağrısını Çalıştırır     │
│                                                          │
│   tool_name = response.tool_calls[0].function.name       │
│   args = response.tool_calls[0].function.arguments       │
│   result = your_function(tool_name, args)                │
└───────────────────────┬─────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│          5. Sonucu Modele Geri Gönder                    │
│                                                          │
│   messages.append(tool_result)                           │
│   # Loop: 2. adıma dön                                 │
└─────────────────────────────────────────────────────────┘
                        │
                        ▼
                  (Döngü — model artık tool
                   sonuçlarına dayanarak
                   doğrudan yanıt verebilir)
```

**Python'da Agent Loop (en basit hali):**

```python
import json
from openai import OpenAI

client = OpenAI()

def agent_loop(messages, tools, max_turns=5):
    for turn in range(max_turns):
        response = client.chat.completions.create(
            model="gpt-4o",
            messages=messages,
            tools=tools,
        )
        
        msg = response.choices[0].message
        
        if not msg.tool_calls:  # stop_reason == "stop"
            return msg.content  # İşlem tamam
        
        # Tool çağrısını işle
        messages.append(msg)
        for tc in msg.tool_calls:
            result = execute_tool(tc.function.name, 
                                   json.loads(tc.function.arguments))
            messages.append({
                "role": "tool",
                "tool_call_id": tc.id,
                "content": json.dumps(result)
            })
        # Döngü devam
    
    return "Max turn aşıldı"
```

---

## 4. Tool Tanımlama — JSON Schema ile

Tool'lar **JSON Schema** formatında tanımlanır. Bu, tüm büyük sağlayıcılar (OpenAI, Anthropic, Google, Mistral) tarafından kullanılan standarttır.

### Tool Tanımının Yapısı

```python
tool = {
    "type": "function",  # OpenAI'de zorunlu; Anthropic'te yok
    "function": {         # OpenAI'de iç içe; Anthropic'te düz
        "name": "get_stock_price",        # Zorunlu, benzersiz
        "description": "Hisse senedi güncel fiyatını alır",  # Modelin ne zaman çağıracağını belirler
        "parameters": {                   # JSON Schema
            "type": "object",
            "properties": {
                "symbol": {
                    "type": "string",
                    "description": "Hisse senedi sembolü (örn: AAPL, GOOGL)"
                },
                "currency": {
                    "type": "string",
                    "enum": ["USD", "TRY", "EUR"],
                    "default": "USD"
                }
            },
            "required": ["symbol"]  # Zorunlu alanlar
        },
        "strict": True  # OpenAI: şema uyumunu zorunlu kılar
    }
}
```

### JSON Schema Temel Tipleri

| Tip | Açıklama | Örnek |
|-----|----------|-------|
| `string` | Metin | `"name": "İstanbul"` |
| `number` | Sayı (ondalıklı) | `"temperature": 23.5` |
| `integer` | Tam sayı | `"count": 42` |
| `boolean` | Doğru/Yanlış | `"active": true` |
| `array` | Dizi | `"items": ["a", "b"]` |
| `object` | Nesne | `"address": {"city": "..."}` |
| `enum` | Sabit değerler kümesi | `"unit": ["celsius", "fahrenheit"]` |

### İyi Bir Tool Tanımının Altın Kuralları

1. **Name açık ve fiil ile başlasın**: `get_weather`, `search_database`, `send_email`
2. **Description modelin karar vermesini sağlar**: _"Kullanıcının takvimine yeni etkinlik ekler"_
3. **Parametre description'ları** hangi değerlerin geçerli olduğunu belirtsin
4. **Required** sadece gerçekten zorunlu olanları içersin
5. **Enum** kullanıyorsan tüm olası değerleri listele

---

## 5. OpenAI Function Calling: Adım Adım

OpenAI, tool-calling'i ilk standartlaştıran sağlayıcıdır (Haziran 2023). API'si `tools` ve `tool_choice` parametrelerini kullanır.

### Adım 1: Tool Tanımla

```python
tools = [
    {
        "type": "function",
        "function": {
            "name": "search_wikipedia",
            "description": "Wikipedia'da arama yapar ve özet döndürür",
            "parameters": {
                "type": "object",
                "properties": {
                    "query": {
                        "type": "string",
                        "description": "Arama sorgusu"
                    },
                    "lang": {
                        "type": "string",
                        "enum": ["tr", "en"],
                        "default": "tr"
                    }
                },
                "required": ["query"]
            }
        }
    }
]
```

### Adım 2: Tool Çağrısını Al

```python
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {"role": "user", "content": "Einstein kimdir?"}
    ],
    tools=tools,
)

message = response.choices[0].message

if message.tool_calls:
    for tool_call in message.tool_calls:
        print(f"Çağrılan tool: {tool_call.function.name}")
        print(f"Argümanlar: {tool_call.function.arguments}")
        print(f"Tool Call ID: {tool_call.id}")
```

### Adım 3: Sonucu İşle ve Geri Gönder

```python
# Tool sonucu
for tool_call in message.tool_calls:
    if tool_call.function.name == "search_wikipedia":
        args = json.loads(tool_call.function.arguments)
        result = wikipedia_search(args["query"], args.get("lang", "tr"))
        
        messages.append({
            "role": "tool",
            "tool_call_id": tool_call.id,
            "content": json.dumps(result)
        })

# Model sonuçla birlikte tekrar çağrılır
final_response = client.chat.completions.create(
    model="gpt-4o",
    messages=messages,  # ← tool sonucu eklenmiş mesajlar
    tools=tools,
)

print(final_response.choices[0].message.content)
```

### OpenAI API Referansı

| Parametre | Tip | Açıklama |
|-----------|-----|----------|
| `tools` | array | Tool tanımları dizisi |
| `tool_choice` | string/object | `"auto"` (varsayılan), `"none"`, `"required"`, veya `{"type": "function", "function": {"name": "..."}}` |
| `parallel_tool_calls` | bool | Varsayılan `true` — model birden çok tool'u paralel çağırabilir |
| `tool_choice.function.name` | string | Belirli bir tool'u zorla |

---

## 6. Anthropic Tool Use: Adım Adım

Anthropic'in yaklaşımı OpenAI'ye benzer ancak bazı farkları vardır.

### Farklar

| Özellik | OpenAI | Anthropic |
|----------|--------|-----------|
| Parametre adı | `tools` | `tools` |
| Tool seçimi | `tool_choice` | `tool_choice` |
| Stop reason | `finish_reason: "tool_calls"` | `stop_reason: "tool_use"` |
| Yanıt formatı | `message.tool_calls[]` | `content` içinde `tool_use` block |
| Tool sonucu | `role: "tool"` | `role: "user"` ile `content` içinde `tool_result` |
| Strict mode | `strict: true` | `strict: true` (yeni) |

### Adım 1: Tool Tanımla

```python
import anthropic

client = anthropic.Anthropic()

tools = [
    {
        "name": "get_exchange_rate",
        "description": "İki para birimi arasındaki güncel döviz kurunu alır",
        "input_schema": {  # ← OpenAI'den farklı: "parameters" yerine "input_schema"
            "type": "object",
            "properties": {
                "from_currency": {
                    "type": "string",
                    "description": "Kaynak para birimi (USD, EUR, TRY...)"
                },
                "to_currency": {
                    "type": "string", 
                    "description": "Hedef para birimi"
                }
            },
            "required": ["from_currency", "to_currency"]
        }
    }
]
```

### Adım 2: Tool Çağrısını Al

```python
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "100 dolar kaç TL?"}
    ],
    tools=tools,
)

for block in response.content:
    if block.type == "tool_use":
        print(f"Tool: {block.name}")
        print(f"Input: {block.input}")
        print(f"ID: {block.id}")
```

### Adım 3: Sonucu Geri Gönder (Anthropic Formatı)

```python
tool_results = []

for block in response.content:
    if block.type == "tool_use":
        result = get_exchange_rate(block.input["from_currency"],
                                    block.input["to_currency"])
        tool_results.append({
            "type": "tool_result",
            "tool_use_id": block.id,
            "content": str(result)
        })

# Anthropic'te tool_result, user mesajının content'i içinde gönderilir
final_response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "100 dolar kaç TL?"},
        {"role": "assistant", "content": response.content},
        {"role": "user", "content": tool_results}
    ],
    tools=tools,
)
```

---

## 7. Çoklu Tool ve Paralel Çağrılar

Bir model tek bir yanıtta birden çok tool çağırabilir.

### Paralel Tool Çağrısı (OpenAI)

```python
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {"role": "user", "content": "İstanbul ve Londra için hava durumunu karşılaştır"}
    ],
    tools=[
        {"type": "function", "function": {"name": "get_weather", ...}},
        {"type": "function", "function": {"name": "get_time", ...}},
    ]
)

message = response.choices[0].message
# message.tool_calls — 2 veya daha fazla tool çağrısı içerebilir!
for tc in message.tool_calls:
    print(f"{tc.function.name}: {tc.function.arguments}")
```

### Önemli: Paralel Çağrılarda Sıra

Paralel tool çağrıları **birbirinden bağımsız** olmalıdır. Eğer 2. tool çağrısı 1. tool'un sonucuna bağımlıysa:

1. İlk turda 1. tool'u çağır
2. Sonucu al
3. İkinci turda 2. tool'u çağır (artık gerekli bağlam var)

```python
# YANLIŞ — aynı turda bağımlı çağrılar
# Tool A: search_flights(origin, dest) 
# Tool B: book_flight(flight_id) ← flight_id A'nın sonucundan gelir

# DOĞRU — iki turlu
# Tur 1: search_flights("IST", "LHR") → flight_id: "TK1978"
# Tur 2: book_flight("TK1978")
```

---

## 8. Tool Choice — Modelin Tool Çağırma Davranışını Kontrol Etme

Her iki sağlayıcı da modelin tool çağırma davranışını kontrol etmenize izin verir.

### OpenAI `tool_choice` Seçenekleri

```python
# 1. Varsayılan — model karar verir
tool_choice = "auto"

# 2. Tool çağırma — model tool çağırmak zorunda değil
tool_choice = "none"  

# 3. Zorunlu tool çağırma — her turda en az bir tool çağrılır
tool_choice = "required"

# 4. Belirli bir tool'u zorla
tool_choice = {
    "type": "function",
    "function": {"name": "get_weather"}
}
```

### Anthropic `tool_choice` Seçenekleri

```python
# 1. Varsayılan — model karar verir
tool_choice = {"type": "auto"}

# 2. Hiç tool çağırma
tool_choice = {"type": "none"}

# 3. Belirli bir tool'u zorla
tool_choice = {"type": "tool", "name": "get_weather"}

# 4. Herhangi bir tool (zorunlu)
tool_choice = {"type": "any"}
```

### Ne Zaman Hangi Seçenek?

| Durum | Kullanılacak `tool_choice` |
|-------|---------------------------|
| Genel amaçlı sohbet | `"auto"` |
| Sadece tool çağrısı gereken işlem | `"required"` / `{"type": "any"}` |
| Belirli bir işlemi zorunlu kıl | `{"function": {"name": "..."}}` |
| Sohbet modu, tool yok | `"none"` / `{"type": "none"}` |

---

## 9. Strict / Constrained Tool Calling

### Strict Mode (OpenAI)

`strict: true` parametresi, modelin çıktısının **her zaman** JSON Schema'ya tam uyumlu olmasını garanti eder.

```python
tool = {
    "type": "function",
    "function": {
        "name": "extract_person",
        "strict": True,  # ← ZORUNLU şema uyumu
        "parameters": {
            "type": "object",
            "properties": {
                "name": {"type": "string"},
                "age": {"type": "integer"},
                "email": {"type": "string", "format": "email"}
            },
            "required": ["name", "age"],
            "additionalProperties": False  # strict ile otomatik
        }
    }
}
```

**Strict mode kısıtlamaları:**
- Tüm `properties` için `type` belirtilmeli
- `additionalProperties: false` otomatik eklenir
- `$ref` kullanılamaz
- Enum değerleri olmayan herhangi bir string tipi kabul edilir

### Strict Tool Use (Anthropic)

Anthropic'te `strict: true` tool tanımına eklenir:

```python
tool = {
    "name": "get_weather",
    "strict": True,
    "input_schema": {
        "type": "object",
        "properties": {
            "location": {"type": "string"}
        },
        "required": ["location"]
    }
}
```

---

## 10. Hata Yönetimi ve Fallback Stratejileri

Tool-calling'de hata kaçınılmazdır. İşte yaygın hata türleri ve çözümleri:

### Hata Türü 1: Tool Çağrılmaz

```python
# Model tool çağırmadı, doğrudan yanıt verdi
if not message.tool_calls:
    # Seçenek A: tool_choice = "required" ile tekrar dene
    # Seçenek B: sisteme "mutlaka tool kullan" talimatı ekle
    # Seçenek C: kullanıcıya "tool kullanmam gerekiyor" bilgisi ver
```

### Hata Türü 2: Geçersiz Argüman

```python
try:
    args = json.loads(tc.function.arguments)
except json.JSONDecodeError:
    # Hata mesajını tool_result olarak geri gönder
    messages.append({
        "role": "tool",
        "tool_call_id": tc.id,
        "content": "HATA: Argümanlar JSON formatında değil. Lütfen geçerli JSON kullanın."
    })
```

### Hata Türü 3: API Hatası (Dış Servis Çalışmadı)

```python
try:
    result = call_external_api(args)
except Exception as e:
    # Hatayı tool_result olarak bildir
    result = {"error": str(e), "suggestion": "Lütfen daha sonra tekrar dene"}
    
messages.append({
    "role": "tool",
    "tool_call_id": tc.id,
    "content": json.dumps(result)
})
```

### Hata Türü 4: Sonsuz Döngü

```python
# Makul sayıda turlama ile sınırla
MAX_TOOL_TURNS = 10
turn_count = 0

while message.tool_calls and turn_count < MAX_TOOL_TURNS:
    # ... işle ...
    turn_count += 1

if turn_count >= MAX_TOOL_TURNS:
    return "İşlem çok fazla adım gerektiriyor, lütfen basitleştirin."
```

### Fallback Stratejisi Şablonu

```python
def safe_tool_call(tc, max_retries=2):
    for attempt in range(max_retries):
        try:
            args = json.loads(tc.function.arguments)
            result = execute_tool(tc.function.name, args)
            return {"success": True, "data": result}
        except json.JSONDecodeError:
            return {"success": False, "error": "invalid_json", 
                    "message": "Argümanlar geçersiz JSON"}
        except ToolExecutionError as e:
            if attempt < max_retries - 1:
                continue  # Tekrar dene
            return {"success": False, "error": "execution_failed",
                    "message": str(e)}
```

---

## 11. Prompt Caching ile Tool Calling Optimizasyonu

Tool tanımları (özellikle çok sayıdaysa) context window'un büyük kısmını kaplar. Prompt caching bu maliyeti düşürür.

### OpenAI Prompt Caching

```python
# Tool tanımları cache'e alınır — tekrar kullanımda %50 maliyet avantajı
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {"role": "system", "content": "Sen bir yardımcı asistansın."},
        {"role": "user", "content": "..."}
    ],
    tools=very_large_tool_list,  # Cache'e kaydedilir
)

# response.usage.prompt_tokens_details.cached_tokens → cache isabeti
```

### Anthropic Prompt Caching

```python
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    system=[
        {
            "type": "text",
            "text": "Sen bir yardımcı asistansın.",
            "cache_control": {"type": "ephemeral"}
        }
    ],
    messages=[...],
    tools=tools,  # Otomatik olarak cache'lenir
)
```

---

## 12. Streaming ile Tool Calling

Streaming'de tool çağrıları delta chunk'lar halinde gelir. Birleştirilmesi gerekir.

### OpenAI Streaming + Tool Calling

```python
stream = client.chat.completions.create(
    model="gpt-4o",
    messages=[...],
    tools=tools,
    stream=True,
)

tool_calls = {}  # index → {id, function: {name, arguments}}

for chunk in stream:
    delta = chunk.choices[0].delta
    
    if delta.tool_calls:
        for tc in delta.tool_calls:
            idx = tc.index
            
            if idx not in tool_calls:
                tool_calls[idx] = tc
                tool_calls[idx].function.arguments = ""
            
            if tc.id:
                tool_calls[idx].id = tc.id
            if tc.function.name:
                tool_calls[idx].function.name = tc.function.name
            if tc.function.arguments:
                tool_calls[idx].function.arguments += tc.function.arguments

# Son hali
for idx, tc in tool_calls.items():
    print(f"Tool: {tc.function.name}")
    print(f"Args: {tc.function.arguments}")
```

### Anthropic Streaming + Tool Use

```python
with client.messages.stream(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    messages=[...],
    tools=tools,
) as stream:
    for event in stream:
        if event.type == "content_block_delta" and event.delta.type == "input_json_delta":
            # Partial JSON parçaları
            pass
        elif event.type == "content_block_stop":
            # Tool use bloğu tamamlandı
            pass
```

---

## 13. İleri Desenler: ReAct, Chain-of-Tool, Tool Loops

### ReAct Deseni (Reasoning + Acting)

ReAct, modelin **düşünmesini** (reasoning) ve **eylemini** (tool use) iç içe geçirir:

```
Soru: "2024 Olimpiyatları'nda en çok altın madalya alan ülke hangisi?"

Thought: Önce 2024 Olimpiyatları'nın nerede yapıldığını bilmiyorum.
         Arama yapmalıyım.
Action: search_web(query="2024 Olympics medal count")
Observation: [arama sonuçları]

Thought: Sonuçlara göre ABD 40 altın madalyayla lider.
         Kaynak: olympics.com
Action: get_page_content(url="olympics.com/medal-count")
Observation: [sayfa içeriği]

Thought: ABD 40 altın, 42 gümüş, 44 bronz toplam 126 madalya.
         Bu bilgiyi doğruladım.
Answer: 2024 Olimpiyatları'nda en çok altın madalya...
```

**Kod:**

```python
REACT_SYSTEM = """Sen bir asistansın. Adım adım düşün ve araçları kullan.

Format:
Thought: şu anki durum hakkında düşüncen
Action: tool_adı(tool_argümanları)
Observation: araçtan gelen sonuç
... (bu döngü tekrarlanır)
Answer: nihai yanıt
"""
```

### Chain-of-Tool (Ardışık Tool)

Bir tool'un çıktısı diğerinin girdisi olur:

```python
# Tool 1: search_products(kategori) → ürün listesi
# Tool 2: get_product_details(ürün_id) → detaylar
# Tool 3: calculate_shipping(ürün_id, adres) → kargo ücreti

# Her turda bir tool — sıralı işlem
```

### Conditional Tool Branching

```python
# Tool sonucuna göre farklı yol
if result["status"] == "not_found":
    # Alternatif tool dene
    next_tool = "search_synonyms"
elif result["status"] == "found":
    # Devam et
    next_tool = "get_details"
```

---

## 14. Gerçek Dünya Vaka Çalışmaları

### Vaka 1: Müşteri Destek Botu

```python
tools = [
    get_order_status(order_id),
    search_faq(query),
    escalate_to_human(reason),
    track_shipment(tracking_no),
    cancel_order(order_id, reason),
]

# Agent Loop:
# 1. Kullanıcı: "Siparişim nerede?"
# 2. Model: get_order_status("12345")
# 3. Sonuç: "Kargoya verildi, takip no: TR123"
# 4. Kullanıcı: "Nerede şu an?"
# 5. Model: track_shipment("TR123")
# 6. Sonuç: "İstanbul aktarma merkezinde"
# 7. Model doğrudan yanıt verir
```

### Vaka 2: Kod Asistanı (Benzerini Kullanıyorsunuz!)

```python
tools = [
    read_file(path, offset, limit),
    write_file(path, content),
    bash(command, timeout),
    search_code(query, path),
]
```

### Vaka 3: Veri Analisti

```python
tools = [
    query_database(sql_query),
    visualize_data(data, chart_type),
    export_csv(data, filename),
    search_documentation(query),
]

# Kullanıcı: "Son 3 ayın satış trendini göster"
# → query_database("SELECT ... GROUP BY month")
# → visualize_data(result, "line_chart")
```

---

## 15. Sık Yapılan Hatalar ve Çözümleri

### Hata 1: Tool Result Format Yanlış

```python
# YANLIŞ — OpenAI
messages.append({"role": "tool", "content": "sıcaklık 25 derece"})
# Eksik: tool_call_id

# DOĞRU
messages.append({
    "role": "tool",
    "tool_call_id": "call_abc123",  # ← ZORUNLU
    "content": json.dumps({"temperature": 25})
})
```

### Hata 2: Tool Result Çok Büyük

```python
# Tool sonucu çok büyükse (50K+ token):
# 1. Özetle
# 2. Sayfala: "Sayfa 1/10, devam etmek için get_next_page çağır"
# 3. Sadece ilk N sonucu döndür
```

### Hata 3: Aynı Tool ID'yi İki Kere Kullanma

```python
# Her tool_call benzersiz ID'ye sahiptir
# Aynı ID ile iki result gönderemezsiniz
```

### Hata 4: Anthropic'te Yanlış Mesaj Sırası

```python
# YANLIŞ — tool_result, assistant mesajından sonra gelmeli
messages = [
    {"role": "user", "content": "..."},
    {"role": "assistant", "content": tool_use_block},
    {"role": "assistant", "content": "..."}  # ← İKİNCİ ASSISTANT YASAK
]

# DOĞRU
messages = [
    {"role": "user", "content": "..."},
    {"role": "assistant", "content": tool_use_block},
    {"role": "user", "content": tool_results}  # ← tool_result user içinde
]
```

### Hata 5: Sonsuz Loop — Tool Her Zaman Aynı Şeyi Çağırır

```python
# Tool hata döndürüyor, model aynı çağrıyı tekrarlıyor
# Çözüm: hata durumunda farklı bir strateji dene
```

---

## 16. Alıştırmalar

### Alıştırma 1: Temel Tool Calling

**Görev:** Hava durumu sorgulayan bir tool-calling sistemi yaz.

1. `get_weather(city)` tool'u tanımla
2. Kullanıcı "İstanbul'da hava nasıl?" dediğinde tool çağrılsın
3. Tool sonucu modele geri dönsün

### Alıştırma 2: Çoklu Tool

**Görev:** 3 tool'lu bir sistem:
- `search_restaurants(cuisine, city)` — restoran ara
- `get_menu(restaurant_id)` — menüyü getir
- `make_reservation(restaurant_id, time, people)` — rezervasyon yap

**Senaryo:** "İstanbul'da İtalyan restoranı ara, menüsüne bak, 4 kişilik rezervasyon yap"

### Alıştırma 3: Hata Yönetimi

**Görev:** Aşağıdaki hata durumlarını ele al:
- Aranan restoran bulunamadı
- Rezervasyon saatinde yer yok
- Tool JSON parse hatası
- API timeout

### Alıştırma 4: ReAct Deseni

**Görev:** Çok adımlı araştırma:
1. "2024 Nobel Fizik Ödülü sahibi kim?"
2. "Hangi üniversitede çalışıyor?"
3. "En önemli makalesi hangisi?"

### Alıştırma 5: Streaming Tool Calling

**Görev:** Streaming ile tool çağrılarını işle. Kullanıcıya partial JSON göstermeden, tool çağrısı tamamlanana kadar bekle.

---

## 17. Referanslar ve İleri Okuma

### Birincil Kaynaklar

| Kaynak | Bağlantı |
|--------|----------|
| OpenAI Function Calling Dökümantasyonu | [https://platform.openai.com/docs/guides/function-calling](https://platform.openai.com/docs/guides/function-calling) |
| Anthropic Tool Use Dökümantasyonu | [https://docs.anthropic.com/en/docs/build-with-claude/tool-use](https://docs.anthropic.com/en/docs/build-with-claude/tool-use) |
| Google Gemini Function Calling | [https://ai.google.dev/docs/function_calling](https://ai.google.dev/docs/function_calling) |
| Mistral Tool Calling | [https://docs.mistral.ai/capabilities/function_calling/](https://docs.mistral.ai/capabilities/function_calling/) |
| ReAct: Reasoning + Acting | [https://arxiv.org/abs/2210.03629](https://arxiv.org/abs/2210.03629) |
| Toolformer | [https://arxiv.org/abs/2302.04761](https://arxiv.org/abs/2302.04761) |
| Gorilla: API Uzmanı LLM | [https://arxiv.org/abs/2305.15334](https://arxiv.org/abs/2305.15334) |

### Araçlar ve Frameworkler

| Araç | Açıklama | Bağlantı |
|------|----------|----------|
| **LangChain** | Tool calling ve agent framework | [https://python.langchain.com/](https://python.langchain.com/) |
| **Vercel AI SDK** | Çoklu sağlayıcı tool calling | [https://sdk.vercel.ai/](https://sdk.vercel.ai/) |
| **Instructor** | Structured output + tool calling | [https://python.useinstructor.com/](https://python.useinstructor.com/) |
| **Pydantic AI** | Pydantic + LLM tool calling | [https://ai.pydantic.dev/](https://ai.pydantic.dev/) |

### JSON Schema Kaynakları

- [JSON Schema Spec](https://json-schema.org/)
- [JSON Schema Türkiye için Örnekler](https://json-schema.org/learn/examples)

---

## Ek: Hızlı Başvuru Kartı

### OpenAI vs Anthropic API Karşılaştırması

```
┌──────────────────────┬──────────────────────┬──────────────────────┐
│                       │ OpenAI               │ Anthropic            │
├──────────────────────┼──────────────────────┼──────────────────────┤
│ Parametre             │ tools                │ tools                │
│ Tool format           │ type: "function"     │ name, input_schema   │
│                       │ function: {name,     │                      │
│                       │   parameters}        │                      │
│ Yanıt                 │ message.tool_calls[] │ content[type=tool_use│
│ Stop reason           │ finish_reason:       │ stop_reason:         │
│                       │ "tool_calls"         │ "tool_use"           │
│ Tool result role      │ "tool"               │ "user" (content      │
│                       │                      │ içinde tool_result)  │
│ Tool result ID        │ tool_call_id         │ tool_use_id          │
│ Strict mode           │ strict: true         │ strict: true         │
│ Tool choice: auto     │ "auto"               │ {"type": "auto"}     │
│ Tool choice: none     │ "none"               │ {"type": "none"}     │
│ Tool choice: forced   │ {"function":         │ {"type": "tool",     │
│                       │   {"name": "..."}}   │   "name": "..."}     │
│ Tool choice: required │ "required"           │ {"type": "any"}      │
│ Parallel calls        | parallel_tool_calls  | Varsayılan evet      │
└──────────────────────┴──────────────────────┴──────────────────────┘
```

### Agent Loop Şablonu (Sağlayıcı Bağımsız)

```python
def run_agent(messages, tools, model="gpt-4o", max_turns=10):
    """Herhangi bir LLM sağlayıcısı ile çalışan temel agent loop."""
    
    for turn in range(max_turns):
        response = call_llm(model, messages, tools)
        msg = parse_response(response)
        
        if not has_tool_calls(msg):
            return get_text_content(msg)
        
        messages.append(msg)
        
        for tool_call in get_tool_calls(msg):
            result = execute_tool(tool_call)
            messages.append(make_tool_result(tool_call.id, result))
    
    return "Maksimum tur sayısına ulaşıldı."
```

---

> **Hazırlayan:** Kendi kendine öğrenen bir AI ajanı  
> **Tarih:** Temmuz 2026  
> **Lisans:** Açık kaynak — geliştirmekte özgürsünüz  
> **Katkı:** PR'larınızı bekliyorum :)
