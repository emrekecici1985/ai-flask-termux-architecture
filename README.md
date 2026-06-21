# 🌐 Termux & Flask Tabanlı Yerel Yapay Zeka Sistem Mimarisi ve Teknik Analizi

Bu depo, Android (Termux) ortamı üzerinde koşan, Flask tabanlı yerel bir tek sayfa yapay zeka uygulamasının (SPA UI) arkasındaki sistem lojistiğini, dinamik hafıza yönetimini ve kesintisiz bağlantı algoritmasını (Failover) detaylandırmak amacıyla hazırlanmış resmi bir mimari analiz dökümanıdır. 

> Güvenlik Notu: Bu depo sadece sistem mimarisi, mantıksal şemalar ve algoritmik yaklaşımları içerir; uygulamanın kaynak kodları, özel CSS/HTML tasarımları ve API anahtarlarından arındırılarak tamamen izole edilmiş özel (Private) bir depoda saklanmaktadır.

---

## 🏗️ 1. Sistem Mimari Şeması (System Architecture)

Sistem, kısıtlı kaynaklara sahip bir mobil ortamda (Termux) maksimum kararlılık ve sıfır veri kaybı prensibiyle tasarlanmıştır. Tüm süreçler asenkron ve akışkan (Streaming) bir yapıda yönetilir.

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

---

## 🧠 2. Dinamik Hafıza Özetleme Algoritması (condense_memory)

Büyük dil modellerinde (LLM) bağlam penceresini (Context Window) verimli kullanmak ve Termux üzerindeki JSON dosya şişmesini envgellemek adına gelişmiş bir dinamik bellek sıkıştırma algoritması uygulanmaktadır.

### Çalışma Mantığı:
* Chat geçmişindeki mesaj trafiği belirli bir eşiğe (Bariyer: 20 Mesaj) ulaştığı an sistem otomatik olarak tetiklenir.
* Geçmişteki ilk mesaj hariç tutularak, aradaki 10 mesajlık blok (Index 1:11) cımbızla çekilir.
* Bu 10 mesajlık yoğun trafik, arka planda llama-3.1-8b modeli ile tek paragraflık semantik bir özete dönüştürülür.
* Elde edilen bu özet, chat geçmişine "Onceki ozet: [summary]" formatında kalıcı olarak enjekte edilir ve sıkıştırılan o 10 mesaj hafızadan silinerek bağlam alanı anında rahatlatılır.

# Algoritmanın Mantıksal Yapısı (Sözde Kod)
def condense_memory(chat_history):
    # Chat geçmişi 20 mesaja ulaştığında bellek sıkıştırmayı tetikle
    if len(chat_history) >= 20:
        # Özetlenecek mesaj bloğunu seç (Index 1 ile 11 arasındaki 10 mesaj)
        target_messages = chat_history[1:11]
        
        # Seçilen 10 mesajı Llama-3.1-8b modeline göndererek semantik özet üret
        summary_paragraph = call_summary_model(target_messages)
        
        # Hafızayı yeniden yapılandır
        condensed_history = [chat_history[0]]  # İlk sistem promptunu koru
        condensed_history.append({"role": "system", "content": f"Önceki özet: {summary_paragraph}"})
        condensed_history.extend(chat_history[11:])  # Kalan güncel mesajları ekle
        
        return condensed_history
        
    return chat_history

---

## 🛡️ 3. Kesintisiz Bağlantı ve API Yük Devretme Algoritması (rotate_key)

Yerel sistemin dış dünya API servislerine bağımlılığında oluşabilecek kesintileri (Hız limitleri, sunucu çökmeleri veya bağlantı kopmaları) sıfıra indirmek adına "Failover Model Chain" mekanizması kurgulanmıştır.

### Güvenlik ve Dayanıklılık Döngüsü:
1. Zaman Aşımı ve Durum Kontrolü: Atılan her chat isteği 45 saniyelik bir sıkı zaman aşımı (Timeout) ve HTTP 200 OK durum kontrolüne tabi tutulur.
2. Hata Yakalama (Failover): Eğer dış servis çökerse veya hız limitine takılırsa, süreç anında rotate_key() fonksiyonunu tetikler.
3. Anahtar Döndürme: Sistem, bünyesindeki Groq API anahtar havuzundan bir sonraki geçerli anahtara otomatik olarak geçiş yapar.
4. Yeniden Dene (SSE Retry): Değişim tamamlandığı an Server-Sent Events (SSE) üzerinden istemciye 'type: retry' sinyali gönderilir ve chat döngüsü kullanıcının ruhu bile duymadan kaldığı yerden, yeni anahtarla sıfırdan başlar.

---

## 🛠️ 4. Donanım ve Sistem Entegrasyon Görevleri (execute_termux_command)

Uygulama, çalıştığı işletim sisteminin (Android/Termux) donanım katmanıyla doğrudan konuşabilmektedir. Kullanıcı prompt içinde "cihaz durumu", "batarya" veya "sistem durumu" gibi kritik kelimeler kullandığında, sistem arka planda subprocess kütüphanesini tetikler:
* termux-battery-status komutu çalıştırılarak batarya yüzdesi, sıcaklığı ve şarj durumu çekilir.
* uptime komutuyla sistemin ne kadar süredir ayakta olduğu hesaplanır.
* Çekilen bu canlı donanım verileri, kullanıcının prompt'unun arkasına görünmez bir sistem notu olarak eklenerek yapay zekanın o anki cihaz durumuna göre cevap vermesi sağlanır.

---

## 📄 5. Akıllı Doküman Okuma Mekanizması (parse_pdf_bytes)

Sisteme yüklenen Base64 formatındaki PDF dosyaları, sunucu tarafında pypdf kütüphanesi yardımıyla byte seviyesinde işlenir. Bellek optimizasyonunu korumak adına dokümanın ilk 10 sayfası taranarak metin ekstraksiyonu (text extraction) yapılır. Çıkarılan bu strict metin içeriği ilk 8000 karakterle sınırlandırılarak istem penceresine (prompt) güvenli bir şekilde enjekte edilir.

---
*Bu teknik dökümantasyon, Emre Keçici tarafından geliştirilen esnek ve taşınabilir yapay zeka mimarilerinin optimizasyon standartlarını belgelemektedir.*
