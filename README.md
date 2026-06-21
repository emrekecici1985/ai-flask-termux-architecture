# 🌐 Termux & Flask Tabanlı Yerel Yapay Zeka Sistem Mimarisi ve Teknik Analizi

Bu depo, Android (Termux) ortamı üzerinde koşan, Flask tabanlı yerel bir tek sayfa yapay zeka uygulamasının (SPA UI) arkasındaki sistem lojistiğini, dinamik hafıza yönetimini ve kesintisiz bağlantı algoritmasını (Failover) detaylandırmak amacıyla hazırlanmış resmi bir mimari analiz dökümanıdır. 

> **Güvenlik Notu:** Bu depo sadece sistem mimarisi, mantıksal şemalar ve algoritmik yaklaşımları içerir; uygulamanın kaynak kodları, özel CSS/HTML tasarımları ve API anahtarlarından arındırılarak tamamen izole edilmiş özel (Private) bir depoda saklanmaktadır.

---

## 🏗️ 1. Sistem Mimari Şeması (System Architecture)

Sistem, kısıtlı kaynaklara sahip bir mobil ortamda (Termux) maksimum kararlılık ve sıfır veri kaybı prensibiyle tasarlanmıştır. Tüm süreçler asenkron ve akışkan (Streaming) bir yapıda yönetilir.

```text
[ Mobil Tarayıcı (SPA UI) ] 
       │ Max-Width: 480px, Inter/JetBrains Mono
       │
       ▼ (POST /chat - Base64 File & Message)
[ Flask Sunucusu (Termux - Port: 8080) ] ──(threading.Lock)──► [ ~/ai_chat_history.json ]
       │
       ├─► [1. Sistem Görevleri & Subprocess] ──► (termux-battery-status & uptime)
       ├─► [2. Doküman İşleme (pypdf)] ────────► (İlk 10 Sayfa Metin Extraction)
       │
       ▼ [3. Akıllı Model Zinciri & Seçim Motoru]
  (Vision? Llama-4-scout-17b | Seçili Model | Default: Llama-3.3-70b -> Llama-3.1-8b)
       │
       ▼ [4. API Failover Döngüsü (rotate_key)]
  (HTTP 200? / 45s Timeout Kontrolü) ───[Hata Durumu]───► [Groq API Anahtarı Değiştir]
       │                                                                  │
       ├───────────────────◄───[Retry SSE Stream]─────────────────────────┘
       │
       ▼ (Server-Sent Events - SSE Stream)
[ Canlı Yanıt Akışı ] ──► [ SPA UI ]
