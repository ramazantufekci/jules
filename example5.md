Harika 🙌 O zaman sana **tam demo dosya setini** hazırlayayım:

---

# 📂 Restoran SEO İçerik Otomasyonu – Demo Dosya Seti

Bu setle beraber:

* Menü CSV dosyan olacak
* FastAPI backend kodların olacak (`/add_menu` + `/generate_content`)
* n8n workflow JSON’un olacak

---

## 1️⃣ Menü CSV (örnek: `menu.csv`)

```csv
Yemek,Açıklama,Fiyat,Kategori
Menemen,Taze domates, biber ve yumurtayla hazırlanan geleneksel kahvaltı,90₺,Kahvaltı
Sucuklu Yumurta,Izgara sucuk ve yumurtayla hazırlanan kahvaltı klasiği,100₺,Kahvaltı
Kuzu Tandır,Odun fırınında saatlerce pişirilen kuzu tandır,350₺,Ana Yemek
Baklava,Ev yapımı cevizli baklava,120₺,Tatlı
```

---

## 2️⃣ FastAPI Backend (`app.py`)

```python
from fastapi import FastAPI
from pydantic import BaseModel
from typing import List
from langchain.vectorstores import Chroma
from langchain.embeddings import OpenAIEmbeddings
from langchain.chat_models import ChatOpenAI
from langchain.prompts import PromptTemplate

# --- FastAPI app ---
app = FastAPI()

# --- Config ---
CHROMA_DB_DIR = "./chroma_db"
MODEL_NAME = "gpt-3.5-turbo"
embeddings = OpenAIEmbeddings()

# --- Vector store ---
vectorstore = Chroma(persist_directory=CHROMA_DB_DIR, embedding_function=embeddings)

# --- LLM ---
llm = ChatOpenAI(model=MODEL_NAME, temperature=0.7)

# --- Prompt ---
TEMPLATE = """
Sen bir restoran için SEO uyumlu içerik üreten bir yazarsın.

Restoran adı: {restaurant}
Anahtar kelime: {keyword}
Kullanılacak bilgiler (menü/özellikler): {context}

Lütfen SEO uyumlu bir {content_type} yazısı üret.
Metin akıcı, ikna edici ve kullanıcıyı restorana yönlendiren bir üslup taşımalı.
"""
prompt = PromptTemplate(template=TEMPLATE, input_variables=["restaurant", "keyword", "context", "content_type"])

# --- Models ---
class MenuItem(BaseModel):
    Yemek: str
    Açıklama: str
    Fiyat: str
    Kategori: str

class MenuRequest(BaseModel):
    menu: List[MenuItem]

class ContentRequest(BaseModel):
    keyword: str
    content_type: str
    restaurant: str

# --- Endpoints ---
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

@app.post("/generate_content")
def generate_content(req: ContentRequest):
    retriever = vectorstore.as_retriever(search_kwargs={"k": 3})
    docs = retriever.get_relevant_documents(req.keyword)
    context = "\n".join([d.page_content for d in docs])

    final_prompt = prompt.format(
        restaurant=req.restaurant,
        keyword=req.keyword,
        context=context,
        content_type=req.content_type
    )

    result = llm.predict(final_prompt)
    return {"content": result}
```

---

## 3️⃣ n8n Workflow JSON (`restoran_workflow.json`)

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
        "functionCode": "const csv = $binary.data.toString();\nconst lines = csv.split(\"\\n\");\nconst headers = lines[0].split(\",\");\nconst result = [];\nfor (let i = 1; i < lines.length; i++) {\n  const obj = {};\n  const currentline = lines[i].split(\",\");\n  headers.forEach((h, j) => { obj[h.trim()] = currentline[j]; });\n  result.push(obj);\n}\nreturn result.map(r => ({ json: r }));"
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
        "bodyParametersJson": "={ \"menu\": $json }"
      },
      "name": "Send to Backend - Add Menu",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 1,
      "position": [800, 300]
    },
    {
      "parameters": {
        "url": "http://backend:8000/generate_content",
        "method": "POST",
        "jsonParameters": true,
        "bodyParametersJson": "{ \"keyword\": \"Ankara kahvaltı mekanı\", \"content_type\": \"blog\", \"restaurant\": \"X Restoran\" }"
      },
      "name": "Generate Content",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 1,
      "position": [1000, 300]
    },
    {
      "parameters": {
        "documentTitle": "SEO Blog - Ankara kahvaltı mekanı",
        "documentContent": "={{ $json[\"content\"] }}"
      },
      "name": "Google Docs Save",
      "type": "n8n-nodes-base.googleDocs",
      "typeVersion": 1,
      "position": [1200, 300]
    }
  ],
  "connections": {
    "Google Drive Trigger": {
      "main": [{ "node": "Google Drive Download", "type": "main", "index": 0 }]
    },
    "Google Drive Download": {
      "main": [{ "node": "Convert CSV to JSON", "type": "main", "index": 0 }]
    },
    "Convert CSV to JSON": {
      "main": [{ "node": "Send to Backend - Add Menu", "type": "main", "index": 0 }]
    },
    "Send to Backend - Add Menu": {
      "main": [{ "node": "Generate Content", "type": "main", "index": 0 }]
    },
    "Generate Content": {
      "main": [{ "node": "Google Docs Save", "type": "main", "index": 0 }]
    }
  }
}
```

---

# ✅ Çalıştırma Adımları

1. `menu.csv` dosyasını **Google Drive’daki `RestoranMenu` klasörüne yükle**.
2. n8n workflow otomatik tetiklenecek.
3. Menü → **Chroma DB’ye kaydedilecek**.
4. `/generate_content` çağrısı yapılıp **SEO blog yazısı üretilecek**.
5. Yazı otomatik olarak **Google Docs’a kaydedilecek**.

---

💡 Artık elinde **uçtan uca demo sistemi** var:

* Restoran sahibi menüyü yüklüyor →
* Senin sistemin içerik üretiyor →
* Google Docs’a kaydediyor →
* (opsiyonel) Instagram / Facebook paylaşımı eklenebilir.

---

👉 İstersen şimdi ben sana bu demo için bir de **Instagram / Facebook paylaşım entegrasyonunu (Meta Graph API ile)** ekleyeyim mi?
