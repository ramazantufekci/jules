Süper 🙌 O zaman sana **uçtan uca restoran demo akışı**nu çıkarıyorum:
👉 Menü **Google Drive’a yükleniyor → Chroma DB’ye kaydediliyor → SEO blog yazısı üretiliyor → Google Docs’a kaydediliyor**.

---

# 🍽️ Uçtan Uca n8n Workflow – Restoran SEO İçerik Otomasyonu

## 🔹 Genel Akış

1. Restoran sahibi **Google Drive’a `menu.csv` yükler**.
2. n8n → CSV’yi JSON’a çevirir → **FastAPI `/add_menu`** endpointine gönderir.
3. n8n → aynı akışta **`/generate_content`** endpointine istekte bulunur.
4. FastAPI + LangChain → **Chroma DB’den menü bilgilerini alıp SEO uyumlu içerik üretir**.
5. n8n → üretilen içeriği **Google Docs’a kaydeder**.
6. (Opsiyonel) Aynı içerik **Instagram / Facebook API** üzerinden sosyal medyaya gönderilir.

---

## 🔹 Workflow Adımları

### 1. **Google Drive Trigger**

* Restoran sahibi `RestoranMenu` klasörüne `menu.csv` yüklediğinde tetiklenir.

---

### 2. **Google Drive Download**

* Yeni yüklenen dosyayı indirir (binary).

---

### 3. **Function Node – CSV → JSON**

CSV’yi JSON formatına dönüştürür (menü öğeleri).

---

### 4. **HTTP Request – Add Menu**

* Endpoint: `http://backend:8000/add_menu`
* Method: `POST`
* Body:

```json
{
  "menu": {{ $json }}
}
```

* Menü öğeleri Chroma DB’ye kaydedilir.

---

### 5. **HTTP Request – Generate Content**

* Endpoint: `http://backend:8000/generate_content`
* Method: `POST`
* Body:

```json
{
  "keyword": "Ankara kahvaltı mekanı",
  "content_type": "blog",
  "restaurant": "X Restoran"
}
```

* FastAPI → LangChain ile SEO uyumlu içerik üretir.

Örnek yanıt:

```json
{
  "content": "Ankara’da kahvaltı keyfi için en doğru adreslerden biri X Restoran..."
}
```

---

### 6. **Google Docs – Create Document**

* Input: `{{ $json["content"] }}`
* Başlık: `"SEO Blog - {{ $json["keyword"] }}"`
* Çıktı: Blog yazısı otomatik olarak Google Drive’a kaydedilir.

---

### 7. **(Opsiyonel) Instagram / Facebook Paylaşımı**

* **Node**: HTTP Request → Meta Graph API
* İçeriği otomatik olarak restoranın Instagram veya Facebook sayfasında paylaşır.

---

## 🔹 Workflow JSON (özet şablon)

```json
{
  "nodes": [
    { "name": "Google Drive Trigger", "type": "n8n-nodes-base.googleDriveTrigger" },
    { "name": "Google Drive Download", "type": "n8n-nodes-base.googleDrive" },
    { "name": "Convert CSV to JSON", "type": "n8n-nodes-base.function" },
    { "name": "Send to Backend - Add Menu", "type": "n8n-nodes-base.httpRequest" },
    { "name": "Generate Content", "type": "n8n-nodes-base.httpRequest" },
    { "name": "Google Docs Save", "type": "n8n-nodes-base.googleDocs" }
  ],
  "connections": {
    "Google Drive Trigger": { "main": [{ "node": "Google Drive Download" }] },
    "Google Drive Download": { "main": [{ "node": "Convert CSV to JSON" }] },
    "Convert CSV to JSON": { "main": [{ "node": "Send to Backend - Add Menu" }] },
    "Send to Backend - Add Menu": { "main": [{ "node": "Generate Content" }] },
    "Generate Content": { "main": [{ "node": "Google Docs Save" }] }
  }
}
```

---

# ✅ Demo Senaryosu

* Restoran sahibi `menu.csv` dosyasını Google Drive’a yükledi.
* n8n → Chroma DB’ye kaydetti.
* Aynı workflow’da `/generate_content` çağrısı yapıldı.
* SEO blog yazısı Google Docs’a otomatik kaydedildi.
* (Opsiyonel) İçerik aynı anda Instagram’da paylaşıldı.

---

👉 İstersen sana bu sistem için **tam bir demo dosya seti** (örnek `menu.csv`, workflow JSON, FastAPI kodları) hazırlayayım, böylece sen direkt çalıştırabilirsin.
Bunu ister misin?
