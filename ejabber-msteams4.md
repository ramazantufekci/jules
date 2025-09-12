Süper 👍 şimdi **Teams → Middleware** yönünü de tamamlayalım.

Teams tarafında 2 farklı yol var:

1. **Kolay test** için: **Teams Outgoing Webhook** (manuel eklenir).
2. **Gerçek prod** için: **Azure AD Bot Registration + Graph API** (daha güçlü ama biraz daha fazla kurulum).

Sen hızlıca test edebil diye önce **Outgoing Webhook** adımlarını anlatıyorum.

---

# 📤 1. Teams Outgoing Webhook ile Middleware Entegrasyonu

### 🔧 Adımlar (Teams Admin Center / Teams Client içinden)

1. Teams uygulamasında (örneğin bir test kanalında):

   * Sağ üstten `...` (More options) → `Manage team`.
   * Sol menüde **Apps** → **Create an outgoing webhook**.

2. Aşağıdaki bilgileri doldur:

   * **Name**: `XMPP Bridge`
   * **Callback URL**: `http://<middleware-public-ip>:5000/teams-hook`
     (Eğer middleware’i dışarıya açmıyorsan `ngrok http 5000` ile geçici bir URL oluşturabilirsin.)
   * **Description**: `XMPP – Teams message bridge`.

3. Kaydet → sana bir **Webhook secret** verecek (örn. `abcd1234...`).

   * Bu değeri middleware’e env variable olarak ekle:

     ```yaml
     - TEAMS_WEBHOOK_SECRET=abcd1234
     ```

---

# 🐍 2. Middleware’de Outgoing Webhook Doğrulaması

`app.py` içine küçük bir doğrulama ekleyelim:

```python
import hmac, hashlib

TEAMS_WEBHOOK_SECRET = os.environ.get("TEAMS_WEBHOOK_SECRET", "")

def verify_teams_signature(req):
    # Teams gönderdiği HMAC SHA256 signature header'ında olur
    signature = req.headers.get("Authorization", "")
    body = req.get_data()
    computed = "HMAC " + hmac.new(
        TEAMS_WEBHOOK_SECRET.encode(), body, hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(signature, computed)

@app.route("/teams-hook", methods=["POST"])
def teams_hook():
    if not verify_teams_signature(request):
        return jsonify({"error": "Invalid signature"}), 403

    data = request.get_json() or {}
    from_email = data.get("from", {}).get("email")
    text = data.get("text", "")

    xmpp_user = get_samaccount_from_email(from_email)
    if not xmpp_user:
        return jsonify({"error": "No AD mapping"}), 404

    payload = {
        "type": "chat",
        "from": EJABBERD_ADMIN,
        "to": f"{xmpp_user}@contoso.com",
        "body": f"[Teams] {from_email}: {text}"
    }
    r = requests.post(
        EJABBERD_HTTP_API,
        json=payload,
        auth=(EJABBERD_ADMIN, EJABBERD_ADMIN_PASS)
    )
    return jsonify({"status": "sent to XMPP", "response": r.text}), r.status_code
```

---

# 🔄 Çalışma Senaryosu

* Ayşe Teams kanalında `@XMPP Bridge Ali nasılsın?` yazar.
* Teams → Outgoing Webhook → Middleware `/teams-hook` çağırır.
* Middleware → AD’de `Ayşe`’nin mail → sAMAccountName eşlemesini bulur.
* Ejabberd HTTP API’ye mesaj POST edilir.
* Ali XMPP client’ında mesajı `[Teams] ayse@contoso.com: Ali nasılsın?` şeklinde görür.

---

# ✅ Özet

* **Ejabberd → Teams** yönü: Graph API ile özel mesaj.
* **Teams → Ejabberd** yönü: Outgoing Webhook → Middleware → Ejabberd API.
* Doğrulama için Teams’in verdiği Webhook secret kullanılıyor.

---

👉 İstersen ben sana **ngrok ile test senaryosu** (middleware’i dış dünyaya açıp Teams webhook callback’ini test etmek için) de adım adım yazayım mı?
