Harika 😄 O zaman sana **menemen görseli ile Instagram postu otomatik paylaşım demo kuralını** çıkarıyorum. Bu, restoran SEO içerik akışını tamamlayacak küçük bir görselli otomasyon örneği.

---

# 🍳 Instagram Görselli Paylaşım Demo

## 🔹 Senaryo

* Menüdeki “Menemen” öğesi için **hazır bir görsel** URL’si var.
* SEO blog yazısı üretildikten sonra, n8n **otomatik olarak bu görseli ve yazıyı Instagram’a gönderiyor**.

---

## 🔹 Görsel URL Örneği

```text
https://example.com/images/menemen.jpg
```

> Gerçek kullanımda restoranın kendi fotoğraflarını barındırdığı bir server veya Google Drive linki kullanabilirsin.

---

## 🔹 n8n Node Ayarları

### 1. **HTTP Request – Instagram Create Media**

* Endpoint:

```
POST https://graph.facebook.com/{ig-user-id}/media
```

* Body:

```json
{
  "caption": "X Restoran’da serpme kahvaltı keyfi sizi bekliyor! 🍳🥖 #AnkaraKahvaltı #Menemen",
  "image_url": "https://example.com/images/menemen.jpg",
  "access_token": "YOUR_PAGE_ACCESS_TOKEN"
}
```

* Output: `id` → gerekli, publish için kullanılacak

---

### 2. **HTTP Request – Instagram Publish Media**

* Endpoint:

```
POST https://graph.facebook.com/{ig-user-id}/media_publish
```

* Body:

```json
{
  "creation_id": "={{ $json[\"id\"] }}",
  "access_token": "YOUR_PAGE_ACCESS_TOKEN"
}
```

> Bu iki node’u Google Docs’a kaydeden node’un **ardına bağla**. Böylece içerik üretiminden sonra otomatik paylaşım gerçekleşir.

---

## 🔹 Workflow Akışı (Görselli)

1. Google Drive → Menü CSV yüklenir
2. CSV → JSON → FastAPI `/add_menu` → Chroma DB’ye kaydedilir
3. `/generate_content` → SEO blog yazısı üretilir
4. Google Docs → yazı kaydedilir
5. Facebook → yazı paylaşılır
6. Instagram → **Menemen görseli + yazı** paylaşılır

---

## 🔹 Ekstra İpuçları

* Farklı menü öğeleri için **koşullu node** ekleyebilirsin:

  * Menemen → menemen.jpg
  * Sucuklu Yumurta → sucuklu.jpg
  * Kuzu Tandır → kuzutandir.jpg
* Böylece her içerik kendi görseliyle paylaşılır.
* Hashtag’leri otomatik oluşturmak için **keyword extraction + prompt** kullanabilirsin.

---

✅ Böylece sistem tamamen **uçtan uca restoran SEO ve sosyal medya otomasyonu** haline geldi:

* Menü CSV → Chroma DB
* SEO içerik üretimi → Google Docs
* Otomatik paylaşım → Facebook + Instagram (görselli)

---

İster misin, bunu bir sonraki adımda **tam test edilebilir örnek workflow dosyası** olarak birleştirip sana JSON halinde vereyim, böylece direkt n8n’e import edip çalıştırabilirsin?
