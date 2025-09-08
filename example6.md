Süper, şimdi bu uçtan uca demoya **Instagram / Facebook paylaşım entegrasyonunu** ekleyelim 🚀

---

# 📲 Meta (Instagram & Facebook) Entegrasyonu

## 🔹 Genel Mantık

* Meta Graph API kullanacağız.
* Restoranın **Facebook Sayfası** ve **Instagram Business hesabı** bağlı olmalı.
* n8n’de **HTTP Request Node** ile içerik paylaşımı yapılacak.

---

## 🔹 Meta API Adımları

1. **Access Token Al**

   * Facebook Developer Console → App oluştur.
   * Sayfa ve IG hesabını bağla.
   * `page_access_token` elde et.

2. **Facebook’ta Gönderi Paylaşımı**

   * Endpoint:

     ```
     POST https://graph.facebook.com/{page-id}/feed
     ```
   * Body:

     ```json
     {
       "message": "Ankara’da kahvaltı keyfi için en doğru adreslerden biri X Restoran...",
       "access_token": "PAGE_ACCESS_TOKEN"
     }
     ```

3. **Instagram’da Gönderi Paylaşımı**

   * 2 aşamalı süreç:

     1. İçeriği oluştur (`/media`)
     2. Yayınla (`/media_publish`)

   * Örnek:

     ```
     POST https://graph.facebook.com/{ig-user-id}/media
     ```

     ```json
     {
       "caption": "X Restoran’da serpme kahvaltı keyfi sizi bekliyor! 🍳🥖",
       "image_url": "https://example.com/kahvalti.jpg",
       "access_token": "PAGE_ACCESS_TOKEN"
     }
     ```

   * Sonra:

     ```
     POST https://graph.facebook.com/{ig-user-id}/media_publish
     ```

     ```json
     {
       "creation_id": "MEDIA_OBJECT_ID",
       "access_token": "PAGE_ACCESS_TOKEN"
     }
     ```

---

# 🔹 n8n Workflow Güncellemesi

Google Docs’a kaydettikten sonra **iki yeni node ekleyeceğiz**:

### 7. **HTTP Request – Facebook Post**

```json
{
  "url": "https://graph.facebook.com/{page-id}/feed",
  "method": "POST",
  "jsonParameters": true,
  "bodyParametersJson": "={ \"message\": $json[\"content\"], \"access_token\": \"YOUR_PAGE_ACCESS_TOKEN\" }"
}
```

---

### 8. **HTTP Request – Instagram Post (Step 1 - Create Media)**

```json
{
  "url": "https://graph.facebook.com/{ig-user-id}/media",
  "method": "POST",
  "jsonParameters": true,
  "bodyParametersJson": "={ \"caption\": $json[\"content\"], \"image_url\": \"https://example.com/default.jpg\", \"access_token\": \"YOUR_PAGE_ACCESS_TOKEN\" }"
}
```

---

### 9. **HTTP Request – Instagram Post (Step 2 - Publish)**

```json
{
  "url": "https://graph.facebook.com/{ig-user-id}/media_publish",
  "method": "POST",
  "jsonParameters": true,
  "bodyParametersJson": "={ \"creation_id\": $json[\"id\"], \"access_token\": \"YOUR_PAGE_ACCESS_TOKEN\" }"
}
```

---

# 🔹 Güncel Akış

1. Google Drive → Menü CSV yüklenir
2. n8n → CSV → JSON → FastAPI `/add_menu` → **Chroma DB’ye kayıt**
3. n8n → `/generate_content` → SEO içerik üretimi
4. n8n → Google Docs → yazıyı kaydet
5. n8n → Facebook → paylaşım yapılır
6. n8n → Instagram → paylaşım yapılır

---

# ✅ Sonuç

Artık sistemin sadece içerik üretmekle kalmıyor, **Google Docs + Facebook + Instagram** entegrasyonu sayesinde restoranın sosyal medya hesaplarında otomatik paylaşım yapıyor. 🎯

---

👉 İstersen sana bunun için **örnek bir görselli demo paylaşımı** (ör. menemen fotoğrafı ile Instagram postu) için küçük bir otomasyon kuralı da ekleyebilirim. İster misin?
