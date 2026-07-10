# Tool-Calling: Kavramın Kökeni, Amaç ve Gelişimi

## Giriş

**Tool-calling** (araç çağırma) — bir dil modelinin (LLM) harici API'ler, fonksiyonlar veya veri kaynaklarıyla etkileşime geçmek üzere yapılandırılmış çıktılar üretmesidir. Kavramın kökleri 2022'ye uzanır; bugünkü yaygın kullanımı ise OpenAI'in 2023 yazında duyurduğu **function calling** özelliğiyle başlamıştır.

---

## 1. ReAct: Reasoning + Acting (Ekim 2022) — Kavramsal Temel

Tool-calling fikrinin ilk sistematik sunumu **ReAct** makalesidir.

- **Yazarlar:** Shunyu Yao, Jeffrey Zhao, Dian Yu, Nan Du, Izhak Shafran, Karthik Narasimhan, Yuan Cao (Google Brain / Princeton)
- **Yayın:** arXiv, 6 Ekim 2022
- **Künye:** arXiv:2210.03629

### Amaç
LLM'lerin **muhakeme (reasoning)** ve **eylem (acting)** yeteneklerini ayrı ayrı değil, iç içe geçmiş şekilde kullanarak sinerji yaratmak. Zincirleme düşünme (Chain-of-Thought) sadece mantıksal akış üretirken dış dünyayla etkileşime girmez; ReAct ise akıl yürütme adımlarıyla **API çağrılarını** (Wikipedia sorgulama gibi) birleştirir.

### Öneri
Modelin **Thought → Action → Observation** döngüsüyle çalışması:
1. **Thought:** Ne yapılması gerektiğine dair akıl yürütme
2. **Action:** Bir dış API/ortam çağrısı (tool use)
3. **Observation:** API'den dönen sonucu işleme

Bu döngü, halüsinasyonu azaltır, hata yayılımını engeller ve insan tarafından yorumlanabilir izler üretir.

### Referans
- **arXiv:** [https://arxiv.org/abs/2210.03629](https://arxiv.org/abs/2210.03629)
- **PDF:** [https://arxiv.org/pdf/2210.03629](https://arxiv.org/pdf/2210.03629)
- **Proje Sayfası:** [https://react-lm.github.io](https://react-lm.github.io)

---

## 2. Toolformer: Kendi Kendine Araç Kullanmayı Öğrenme (Şubat 2023)

ReAct'in ardından Meta AI, LLM'lerin **kendi kendine** araç kullanmayı öğrenmesini sağlayan Toolformer'ı yayınladı.

- **Yazarlar:** Timo Schick, Jane Dwivedi-Yu, Roberto Dessì, Roberta Raileanu, Maria Lomeli, Luke Zettlemoyer, Nicola Cancedda, Thomas Scialom (Meta AI)
- **Yayın:** arXiv, 9 Şubat 2023
- **Künye:** arXiv:2302.04761

### Amaç
LLM'ler doğal dilde müthiş yetenekler gösterirken, basit aritmetik veya güncel bilgi sorgulama gibi işlemlerde başarısız olur. Amaç, modelin **hangi API'yi, ne zaman, hangi argümanlarla çağıracağını** ve sonucu nasıl entegre edeceğini kendi kendine (self-supervised) öğrenmesidir.

### Öneri
Toolformer, her bir API için sadece **birkaç demostrasyon** ile modelin kendi kendine API çağrıları eklemeyi öğrenmesini sağlar. Desteklenen araçlar: hesap makinesi, Q&A sistemi, arama motorları, çeviri, takvim.

### Referans
- **arXiv:** [https://arxiv.org/abs/2302.04761](https://arxiv.org/abs/2302.04761)
- **PDF:** [https://arxiv.org/pdf/2302.04761](https://arxiv.org/pdf/2302.04761)

---

## 3. Gorilla: LLM'leri API Çağrılarında Uzmanlaştırma (Mayıs 2023)

UC Berkeley ve Microsoft araştırmacıları, API çağrıları üretme konusunda GPT-4'ü geçen bir model yayınladı.

- **Yazarlar:** Shishir G. Patil, Tianjun Zhang, Xin Wang, Joseph E. Gonzalez (UC Berkeley)
- **Yayın:** arXiv, 24 Mayıs 2023
- **Künye:** arXiv:2305.15334

### Amaç
LLM'lerin API kullanımındaki iki temel sorunu çözmek: (1) doğru girdi argümanlarını üretememe, (2) API kullanımında halüsinasyon görme.

### Öneri
LLaMA tabanlı **Gorilla** modeli, bir döküman getirici (retriever) ile birleşerek güncel API dökümantasyonuna uyum sağlar. **APIBench** veri kümesi (HuggingFace, TorchHub, TensorHub API'leri) ile değerlendirilmiştir.

### Referans
- **arXiv:** [https://arxiv.org/abs/2305.15334](https://arxiv.org/abs/2305.15334)
- **PDF:** [https://arxiv.org/pdf/2305.15334](https://arxiv.org/pdf/2305.15334)
- **Proje:** [https://gorilla.cs.berkeley.edu](https://gorilla.cs.berkeley.edu)

---

## 4. OpenAI Function Calling — Endüstri Standardı (13 Haziran 2023)

Tool-calling kavramını **milyonlarca geliştiriciye** ulaştıran ve "function calling" adını veren dönüm noktası.

- **Duyuru:** OpenAI Blog, 13 Haziran 2023
- **Yazarlar (duyuru):** Atty Eleti, Jeff Harris, Logan Kilpatrick
- **Modeller:** `gpt-4-0613`, `gpt-3.5-turbo-0613`

### Amaç
Geliştiricilerin GPT'yi dış araçlara ve API'lere **güvenilir** şekilde bağlamasını sağlamak. Modelin ne zaman fonksiyon çağırması gerektiğini algılaması ve JSON şemasına uygun çıktı üretmesi.

### Öneri
Chat Completions API'ye iki yeni parametre:
- **`functions`**: JSON Schema ile tanımlanmış fonksiyon listesi
- **`function_call`**: Zorunlu/isteğe bağlı fonksiyon çağırma kontrolü

Kullanım alanları:
1. Harici araçlarla sohbet (ChatGPT Plugins benzeri)
2. Doğal dilden API/veritabanı sorgusuna dönüşüm
3. Yapılandırılmış veri çıkarma

### Referans
- **Duyuru (Wayback Machine arşivi):** [https://web.archive.org/web/20230613220457/https://openai.com/blog/function-calling-and-other-api-updates](https://web.archive.org/web/20230613220457/https://openai.com/blog/function-calling-and-other-api-updates)
- **Orijinal (artık yönlendirme yapıyor):** [https://openai.com/index/function-calling-and-other-api-updates/](https://openai.com/index/function-calling-and-other-api-updates/)
- **API Dökümantasyonu:** [https://platform.openai.com/docs/guides/function-calling](https://platform.openai.com/docs/guides/function-calling)

---

## 5. Anthropic Tool Use (2024)

Anthropic, Claude modellerine **tool use** (araç kullanma) yeteneği ekledi.

- **Dökümantasyon:** [https://docs.anthropic.com/en/docs/build-with-claude/tool-use](https://docs.anthropic.com/en/docs/build-with-claude/tool-use)
- **Özellikler:** JSON Schema ile tool tanımı, `strict: true` ile şema uyum garantisi, sistem prompt'u ile tool çağırma davranışını yönlendirme

---

## 6. Diğer Önemli Katkılar

| Çalışma | Yıl | Katkı |
|---------|-----|-------|
| **Tool Augmented Language Models** (Google) | 2022 | Araç destekli modeller üzerine kapsamlı inceleme |
| **WebGPT** (OpenAI) | 2021 | Web araması yaparak soru cevaplama |
| **SayCan** (Google/Robotics) | 2022 | Robotik eylemler için tool calling |
| **LLM+P** (Princeton) | 2023 | Planlayıcıyı dış araç olarak kullanma |

---

## Zaman Çizelgesi

```
2021    ── WebGPT (OpenAI) — web araması
2022-10 ── ReAct (Google/Princeton) — reasoning + acting
2023-02 ── Toolformer (Meta) — self-supervised tool learning
2023-05 ── Gorilla (UC Berkeley) — API uzmanı LLM
2023-06 ── OpenAI Function Calling — endüstri standardı
2024    ── Anthropic Tool Use, Google Function Calling
```

---

## Özet

- **Kavramı ilk ortaya atan:** ReAct (Yao et al., 2022) — reasoning ve acting'i birleştiren çerçeve
- **Amaç:** LLM'lerin dış dünyayla (API'ler, veritabanları, ortamlar) etkileşime girerek halüsinasyonu azaltması ve daha güvenilir sonuçlar üretmesi
- **Endüstri standardı haline getiren:** OpenAI Function Calling (Haziran 2023)
- **Temel mekanizma:** Model → yapılandırılmış JSON çıktısı (fonksiyon adı + argümanlar) → dış sistem çağrısı → sonucun modele geri beslenmesi

---

*Hazırlanma Tarihi: Temmuz 2026*
