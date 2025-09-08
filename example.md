Harika 🙌 O zaman sana restoran SEO içerik üretici için **örnek FastAPI + LangChain backend** kodunu hazırlıyorum. Bu servis, n8n’in çağıracağı `/generate_content` endpoint’ini barındıracak.

---

# 🛠️ FastAPI + LangChain Backend (örnek)

```python
from fastapi import FastAPI
from pydantic import BaseModel
from langchain.vectorstores import Chroma
from langchain.embeddings import OpenAIEmbeddings
from langchain.chat_models import ChatOpenAI
from langchain.prompts import PromptTemplate
from langchain.chains import RetrievalQA

# --- FastAPI app ---
app = FastAPI()

# --- Config ---
CHROMA_DB_DIR = "./chroma_db"
MODEL_NAME = "gpt-3.5-turbo"   # düşük maliyet için
embeddings = OpenAIEmbeddings()

# --- Vektör veritabanı ---
vectorstore = Chroma(
    persist_directory=CHROMA_DB_DIR,
    embedding_function=embeddings
)

retriever = vectorstore.as_retriever(search_kwargs={"k": 3})
llm = ChatOpenAI(model=MODEL_NAME, temperature=0.7)

# --- Prompt şablonu ---
TEMPLATE = """
Sen bir restoran için SEO uyumlu içerik üreten bir yazarsın.

Restoran adı: {restaurant}
Anahtar kelime: {keyword}
Kullanılacak bilgiler (menü/özellikler): {context}

Lütfen SEO uyumlu bir {content_type} yazısı üret.
Metin akıcı, ikna edici ve kullanıcıyı restorana yönlendiren bir üslup taşımalı.
"""

prompt = PromptTemplate(
    template=TEMPLATE,
    input_variables=["restaurant", "keyword", "context", "content_type"]
)

# --- API giriş modeli ---
class ContentRequest(BaseModel):
    keyword: str
    content_type: str  # blog, social_post, google_business
    restaurant: str

# --- Endpoint ---
@app.post("/generate_content")
def generate_content(req: ContentRequest):
    # ilgili bilgileri Chroma'dan getir
    docs = retriever.get_relevant_documents(req.keyword)
    context = "\n".join([d.page_content for d in docs])

    # prompt doldur
    final_prompt = prompt.format(
        restaurant=req.restaurant,
        keyword=req.keyword,
        context=context,
        content_type=req.content_type
    )

    # modelden yanıt al
    result = llm.predict(final_prompt)

    return {"content": result}
```

---

# ⚙️ Çalıştırma

1. Dosyayı `app.py` olarak kaydet.
2. FastAPI’yi çalıştır:

   ```bash
   uvicorn app:app --reload --port 8000
   ```
3. n8n’den şu endpoint’e POST isteği gönder:

   ```
   http://localhost:8000/generate_content
   ```

   Body örneği:

   ```json
   {
     "keyword": "Ankara kahvaltı mekanı",
     "content_type": "blog",
     "restaurant": "X Restoran"
   }
   ```

---

# ✅ Örnek Yanıt

```json
{
  "content": "Ankara’da kahvaltı keyfi için en doğru adreslerden biri X Restoran. Serpme kahvaltı menümüz..."
}
```

---

💡 Bu backend’i n8n **HTTP Request node** üzerinden kolayca çağırabilirsin. Çıktıyı Google Docs’a ya da Instagram API’sine gönderip zinciri tamamlamış olursun.

---

👉 İstersen sana bunun için bir de **Chroma DB’ye restoran menüsünü yükleme scripti (örnek Python kodu)** yazayım mı? Böylece menüyü önceden vektör veritabanına ekleyip içerik üretiminde kullanabilirsin.
