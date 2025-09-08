Süper 👍 Şimdi sana restoran menüsünü **Google Drive’a yüklenen CSV dosyasından otomatik olarak alıp Chroma DB’ye gönderen bir n8n workflow örneği** hazırlayayım.

---

# 🍽️ n8n Workflow – Menü CSV → Chroma DB Pipeline

## 🔹 Senaryo

* Restoran sahibi menüsünü **Google Drive’a** yükler (ör. `menu.csv`).
* n8n bu dosyayı algılar → okur → **Python backend’e gönderir**.
* Backend (`load_menu.py`) CSV’yi alır → **Chroma DB’ye kaydeder**.

---

## 🔹 Adımlar

### 1. **Trigger: Google Drive**

* **Node**: *Google Drive Trigger*
* Ayar: Belirli bir klasör (ör. `RestoranMenu`) izlenir.
* Event: *“New File”* → yeni CSV yüklendiğinde tetiklenir.

---

### 2. **Node: Google Drive – Download File**

* Dosya içeriklerini çeker.
* Çıktı: CSV dosyasının binary içeriği.

---

### 3. **Node: Function – Convert CSV to JSON**

```javascript
const csv = $binary.data.toString();
const lines = csv.split("\n");
const headers = lines[0].split(",");
const result = [];

for (let i = 1; i < lines.length; i++) {
  const obj = {};
  const currentline = lines[i].split(",");
  headers.forEach((h, j) => {
    obj[h.trim()] = currentline[j];
  });
  result.push(obj);
}

return result.map(r => ({ json: r }));
```

> Böylece CSV → JSON formatına dönüştürülür.

---

### 4. **Node: HTTP Request – Chroma API (veya Backend API)**

* Yöntem: `POST`
* URL: `http://backend:8000/add_menu`
* Body:

```json
{
  "menu": {{ $json }}
}
```

> Bu isteği senin **FastAPI backend’inde yeni bir `/add_menu` endpoint’i** karşılar.
> O endpoint → LangChain embeddings + Chroma DB’ye kayıt yapar.

---

### 5. **Opsiyonel: Slack/Email Bildirimi**

* n8n, “Menü başarıyla Chroma DB’ye kaydedildi ✅” mesajını restoran sahibine gönderebilir.

---

## 🔹 Workflow JSON (örnek)

```json
{
  "nodes": [
    {
      "parameters": {
        "triggerOn": "fileUploaded",
        "folder": "RestoranMenu"
      },
      "name": "Google Drive Trigger",
      "type": "n8n-nodes-base.googleDriveTrigger",
      "typeVersion": 1,
      "position": [200, 300]
    },
    {
      "parameters": {
        "fileId": "={{$json[\"id\"]}}",
        "binaryProperty": "data"
      },
      "name": "Google Drive Download",
      "type": "n8n-nodes-base.googleDrive",
      "typeVersion": 1,
      "position": [400, 300]
    },
    {
      "parameters": {
        "functionCode": "const csv = $binary.data.toString();\nconst lines = csv.split(\"\\n\");\nconst headers = lines[0].split(\",\");\nconst result = [];\n\nfor (let i = 1; i < lines.length; i++) {\n  const obj = {};\n  const currentline = lines[i].split(\",\");\n  headers.forEach((h, j) => {\n    obj[h.trim()] = currentline[j];\n  });\n  result.push(obj);\n}\n\nreturn result.map(r => ({ json: r }));"
      },
      "name": "Convert CSV to JSON",
      "type": "n8n-nodes-base.function",
      "typeVersion": 1,
      "position": [600, 300]
    },
    {
      "parameters": {
        "url": "http://backend:8000/add_menu",
        "method": "POST",
        "jsonParameters": true,
        "options": {},
        "bodyParametersJson": "={ \"menu\": $json }"
      },
      "name": "Send to Backend",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 1,
      "position": [800, 300]
    }
  ],
  "connections": {
    "Google Drive Trigger": {
      "main": [
        [{ "node": "Google Drive Download", "type": "main", "index": 0 }]
      ]
    },
    "Google Drive Download": {
      "main": [
        [{ "node": "Convert CSV to JSON", "type": "main", "index": 0 }]
      ]
    },
    "Convert CSV to JSON": {
      "main": [
        [{ "node": "Send to Backend", "type": "main", "index": 0 }]
      ]
    }
  }
}
```

---

## 🔹 Ekleme: Backend için `/add_menu` endpoint

FastAPI tarafında basitçe:

```python
from fastapi import FastAPI
from pydantic import BaseModel
from typing import List, Dict
from langchain.vectorstores import Chroma
from langchain.embeddings import OpenAIEmbeddings

app = FastAPI()

CHROMA_DB_DIR = "./chroma_db"
embeddings = OpenAIEmbeddings()
vectorstore = Chroma(persist_directory=CHROMA_DB_DIR, embedding_function=embeddings)

class MenuItem(BaseModel):
    Yemek: str
    Açıklama: str
    Fiyat: str
    Kategori: str

class MenuRequest(BaseModel):
    menu: List[MenuItem]

@app.post("/add_menu")
def add_menu(req: MenuRequest):
    docs = []
    metas = []

    for item in req.menu:
        text = f"{item.Yemek} - {item.Açıklama} (Fiyat: {item.Fiyat})"
        docs.append(text)
        metas.append({"kategori": item.Kategori, "fiyat": item.Fiyat})

    vectorstore.add_texts(texts=docs, metadatas=metas)
    vectorstore.persist()

    return {"status": "success", "added": len(docs)}
```

---

✅ Artık:

1. Restoran sahibi Google Drive’a `menu.csv` yükler.
2. n8n → CSV’yi JSON’a çevirir → FastAPI `/add_menu` endpoint’ine gönderir.
3. Menü öğeleri Chroma DB’ye kaydedilir.
4. İçerik üretim pipeline hazır olur.

---

👉 İstersen sana şimdi **tam bir uçtan uca demo akışını** hazırlayayım:

* Menü Drive’a yüklendiğinde → Chroma’ya kaydedilsin → ardından otomatik SEO blog yazısı üretilip Google Docs’a kaydedilsin.

Bunu ister misin?
