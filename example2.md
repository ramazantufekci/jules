Süper 🙌 O zaman sana restoran menüsünü **Chroma DB’ye yüklemek** için örnek bir Python scripti yazıyorum. Bu sayede menü → embedding → vektör veritabanı süreci tamamlanmış olacak.

---

# 🛠️ Menü Yükleme Scripti (Python + LangChain + Chroma)

```python
from langchain.vectorstores import Chroma
from langchain.embeddings import OpenAIEmbeddings
import csv

# --- Config ---
CHROMA_DB_DIR = "./chroma_db"
CSV_FILE = "./menu.csv"   # Menü bilgilerini CSV’den okuyacağız

# --- Embedding modeli ---
embeddings = OpenAIEmbeddings()

# --- Vektör veritabanı oluştur ---
vectorstore = Chroma(
    persist_directory=CHROMA_DB_DIR,
    embedding_function=embeddings
)

def load_menu_to_chroma():
    documents = []
    metadatas = []

    # Menü CSV formatı: Yemek, Açıklama, Fiyat, Kategori
    with open(CSV_FILE, "r", encoding="utf-8") as f:
        reader = csv.DictReader(f)
        for i, row in enumerate(reader):
            text = f"{row['Yemek']} - {row['Açıklama']} (Fiyat: {row['Fiyat']})"
            documents.append(text)
            metadatas.append({
                "kategori": row["Kategori"],
                "fiyat": row["Fiyat"]
            })

    # Embedding + Chroma’ya ekle
    vectorstore.add_texts(texts=documents, metadatas=metadatas)
    vectorstore.persist()

    print(f"{len(documents)} menü öğesi Chroma DB'ye eklendi ✅")

if __name__ == "__main__":
    load_menu_to_chroma()
```

---

# 📂 Örnek `menu.csv`

```csv
Yemek,Açıklama,Fiyat,Kategori
Menemen,Taze domates, biber ve yumurtayla hazırlanan geleneksel kahvaltı,90₺,Kahvaltı
Sucuklu Yumurta,Izgara sucuk ve yumurtayla hazırlanan kahvaltı klasiği,100₺,Kahvaltı
Kuzu Tandır,Odun fırınında saatlerce pişirilen kuzu tandır,350₺,Ana Yemek
Baklava,Ev yapımı cevizli baklava,120₺,Tatlı
```

---

# ✅ Çalıştırma

1. Menü dosyasını (`menu.csv`) hazırla.
2. Script’i çalıştır:

   ```bash
   python load_menu.py
   ```
3. Menü embedding’leri `./chroma_db` klasörüne kaydedilir.
4. Daha sonra FastAPI servisindeki `/generate_content` endpoint’i bu verileri kullanarak içerik üretebilir.

---

💡 Böylece:

* Restoran sahibi → Menü CSV’sini yükler.
* Senin script → Chroma DB’ye kaydeder.
* FastAPI servisi → SEO uyumlu içerik üretir.
* n8n → İçeriği Google Docs / Instagram’a gönderir.

---

👉 İstersen ben sana bir de **n8n tarafında bu menü CSV’sini otomatik işleyip Chroma’ya gönderen workflow** örneği hazırlayayım mı? (Yani restoran sahibi Google Drive’a CSV yükleyince → otomatik pipeline çalışsın.)
