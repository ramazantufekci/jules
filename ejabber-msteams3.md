Tamamdır ✅ O zaman middleware’i dış dünyaya açıp Teams Outgoing Webhook’u test etmek için **ngrok ile test senaryosu** adımlarını hazırlayalım.

---

# 1️⃣ Ngrok Kurulumu

1. [https://ngrok.com/download](https://ngrok.com/download) adresinden ngrok’u indirip bilgisayarına kur.
2. Ücretsiz hesabın varsa `authtoken` alıp ngrok’a login ol:

```bash
ngrok config add-authtoken <YOUR_NGROK_AUTHTOKEN>
```

---

# 2️⃣ Middleware’i Docker ile Çalıştır

Önce `docker-compose up --build` ile middleware’i ayağa kaldır:

```bash
docker-compose up --build middleware
```

* Middleware container 5000 portunu açıyor.
* Localhost üzerinde test ediyorsan `http://localhost:5000/teams-hook` endpoint’i aktif olacak.

---

# 3️⃣ Ngrok ile Public URL Oluştur

```bash
ngrok http 5000
```

* Çıktıda göreceksin:

```
Forwarding                    https://abcd1234.ngrok.io -> http://localhost:5000
```

* Bu `https://abcd1234.ngrok.io/teams-hook` URL’i Teams Outgoing Webhook callback olarak kullanılacak.

---

# 4️⃣ Teams Outgoing Webhook Ayarı

1. Teams kanalında **Manage team → Apps → Create outgoing webhook**
2. Callback URL: `https://abcd1234.ngrok.io/teams-hook`
3. Secret: `abcd1234` (veya kendi oluşturduğun secret)
4. Kaydet → Webhook hazır.

---

# 5️⃣ Test Senaryosu

1. **Ejabberd → Teams** testi:

   * Ali ejabberd client’ından mesaj atar: `Merhaba Ayşe!`
   * Middleware → AD’den Ayşe’nin Teams mail’ini bulur → Graph API ile Teams’e düşer.
   * Teams kanalında mesajı görürsün.

2. **Teams → Ejabberd** testi:

   * Ayşe Teams kanalında `@XMPP Bridge Merhaba Ali` yazar.
   * Teams webhook → ngrok URL → middleware → AD lookup → ejabberd API → Ali’nin XMPP client’ına düşer.

---

# 6️⃣ Curl ile Test (Opsiyonel)

Webhook’u simüle etmek için curl kullanabilirsin:

```bash
curl -X POST https://abcd1234.ngrok.io/teams-hook \
 -H "Content-Type: application/json" \
 -H "Authorization: HMAC abcd1234" \
 -d '{
   "from": {"email": "ayse@contoso.com"},
   "text": "Test mesajı"
 }'
```

* Middleware AD’den `ayse@contoso.com` → `sAMAccountName` bulur → Ejabberd API’ye gönderir.

---

# 7️⃣ Notlar

* Ngrok ücretsiz sürümünde URL her açılışta değişir. Prod ortamında statik public IP veya reverse proxy önerilir.
* Teams → Middleware iletişimi HTTPS olmalı, ngrok bunu otomatik sağlıyor.
* Middleware loglarını takip ederek mesaj akışını doğrulayabilirsin:

```bash
docker logs -f xmpp-teams-middleware
```

---

Böylece **çift yönlü XMPP ↔ Teams köprüsü**, Windows AD kullanıcı eşlemesi ile ngrok üzerinden test edilebilir hale gelmiş oluyor.

İstersen bir sonraki adım olarak, **prod ortamında ngrok olmadan, HTTPS ve statik IP ile nasıl deploy edilir** onu da adım adım hazırlayabilirim.
