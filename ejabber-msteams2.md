Harika 👌 o zaman sana **Windows Server AD’ye bağlanan** bir `docker-compose` örneği hazırlıyorum. Bu örnekte:

* **Ejabberd**: Kullanıcı doğrulamasını **Active Directory** üzerinden yapacak.
* **Middleware**: AD’den `sAMAccountName ↔ mail` eşleşmesini çıkaracak, ejabberd ile Teams arasında mesaj köprüsü olacak.
* **Windows Server AD**: Halihazırda ortamında çalışıyor (yani docker içinde değil, dışarıda).

---

# 📂 Proje Yapısı

```
xmpp-teams-bridge/
├─ docker-compose.yml
├─ ejabberd/
│  └─ ejabberd.yml
└─ middleware/
   ├─ Dockerfile
   ├─ requirements.txt
   ├─ app.py
   └─ ad_lookup.py
```

---

# 1) `docker-compose.yml`

```yaml
version: "3.8"
services:
  ejabberd:
    image: ejabberd/ecs:23.04
    container_name: ejabberd
    volumes:
      - ./ejabberd/ejabberd.yml:/home/ejabberd/conf/ejabberd.yml:ro
    environment:
      - ERLANG_NODE=ejabberd@ejabberd
    ports:
      - "5222:5222"   # client XMPP
      - "5280:5280"   # http api
    restart: always

  middleware:
    build: ./middleware
    container_name: xmpp-teams-middleware
    volumes:
      - ./middleware:/app
    environment:
      - EJABBERD_HTTP_API=http://ejabberd:5280/api/send_message
      - EJABBERD_ADMIN=admin@contoso.com
      - EJABBERD_ADMIN_PASS=AdminSecret
      - AD_HOST=ad1.contoso.com
      - AD_USER=CN=ldapbind,CN=Users,DC=contoso,DC=com
      - AD_PASS=SuperSecretPass!
      - AD_BASE=DC=contoso,DC=com
      - GRAPH_TENANT_ID=YOUR_TENANT_ID
      - GRAPH_CLIENT_ID=YOUR_CLIENT_ID
      - GRAPH_CLIENT_SECRET=YOUR_CLIENT_SECRET
    ports:
      - "5000:5000"
    depends_on:
      - ejabberd
```

> 🔑 Burada `ad1.contoso.com` ortamındaki Windows Server AD domain controller adresi.
> `ldapbind` kullanıcı hesabı salt-okuma yetkili bir service account olmalı.

---

# 2) `ejabberd/ejabberd.yml`

```yaml
hosts:
  - "contoso.com"

auth_method: ldap

ldap_servers:
  - "ad1.contoso.com"
ldap_port: 389
ldap_rootdn: "CN=ldapbind,CN=Users,DC=contoso,DC=com"
ldap_password: "SuperSecretPass!"
ldap_base: "DC=contoso,DC=com"

# Kullanıcılar sAMAccountName ile login olacak (örn: "ali")
ldap_uids:
  - "sAMAccountName"

modules:
  mod_http_api: {}
  mod_event_pusher:
    webhooks:
      - url: "http://middleware:5000/xmpp-hook"
```

---

# 3) `middleware/Dockerfile`

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["python", "app.py"]
```

---

# 4) `middleware/requirements.txt`

```
flask
requests
ldap3
msal
```

---

# 5) `middleware/ad_lookup.py`

```python
from ldap3 import Server, Connection, ALL
import os

AD_HOST = os.environ.get("AD_HOST", "ad1.contoso.com")
AD_USER = os.environ.get("AD_USER")
AD_PASS = os.environ.get("AD_PASS")
AD_BASE = os.environ.get("AD_BASE")

def get_teams_email_from_samaccount(samaccount):
    server = Server(AD_HOST, get_info=ALL)
    conn = Connection(server, AD_USER, AD_PASS, auto_bind=True)
    search_filter = f"(sAMAccountName={samaccount})"
    conn.search(AD_BASE, search_filter, attributes=["mail"])
    if conn.entries:
        return conn.entries[0].mail.value
    return None

def get_samaccount_from_email(email):
    server = Server(AD_HOST, get_info=ALL)
    conn = Connection(server, AD_USER, AD_PASS, auto_bind=True)
    search_filter = f"(mail={email})"
    conn.search(AD_BASE, search_filter, attributes=["sAMAccountName"])
    if conn.entries:
        return conn.entries[0].sAMAccountName.value
    return None
```

---

# 6) `middleware/app.py`

```python
from flask import Flask, request, jsonify
import requests
import json
import os
from msal import ConfidentialClientApplication
from ad_lookup import get_teams_email_from_samaccount, get_samaccount_from_email

app = Flask(__name__)

EJABBERD_HTTP_API = os.environ.get("EJABBERD_HTTP_API")
EJABBERD_ADMIN = os.environ.get("EJABBERD_ADMIN")
EJABBERD_ADMIN_PASS = os.environ.get("EJABBERD_ADMIN_PASS")

TENANT_ID = os.environ.get("GRAPH_TENANT_ID")
CLIENT_ID = os.environ.get("GRAPH_CLIENT_ID")
CLIENT_SECRET = os.environ.get("GRAPH_CLIENT_SECRET")
SCOPES = ["https://graph.microsoft.com/.default"]

msal_app = ConfidentialClientApplication(
    CLIENT_ID,
    authority=f"https://login.microsoftonline.com/{TENANT_ID}",
    client_credential=CLIENT_SECRET,
)

def get_graph_token():
    result = msal_app.acquire_token_silent(SCOPES, account=None)
    if not result:
        result = msal_app.acquire_token_for_client(scopes=SCOPES)
    return result.get("access_token")

@app.route("/xmpp-hook", methods=["POST"])
def xmpp_hook():
    data = request.get_json() or {}
    sender = data.get("from")
    to = data.get("to")
    body = data.get("body", "")

    # "ayse@contoso.com" → "ayse" localpart
    to_local = to.split("@")[0]
    teams_email = get_teams_email_from_samaccount(to_local)
    if not teams_email:
        return jsonify({"error": "No mapping"}), 404

    token = get_graph_token()
    if not token:
        return jsonify({"error": "No graph token"}), 500

    headers = {"Authorization": f"Bearer {token}", "Content-Type": "application/json"}
    msg_payload = {
      "message": {
        "subject": "XMPP message",
        "body": {"contentType": "Text", "content": f"{sender}: {body}"},
        "toRecipients": [{"emailAddress": {"address": teams_email}}]
      }
    }
    r = requests.post(
        f"https://graph.microsoft.com/v1.0/users/{teams_email}/sendMail",
        headers=headers, data=json.dumps(msg_payload)
    )
    return jsonify({"status": "sent to Teams", "response": r.text}), r.status_code

@app.route("/teams-hook", methods=["POST"])
def teams_hook():
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
    r = requests.post(EJABBERD_HTTP_API, json=payload, auth=(EJABBERD_ADMIN, EJABBERD_ADMIN_PASS))
    return jsonify({"status": "sent to XMPP", "response": r.text}), r.status_code

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000, debug=True)
```

---

# 🚀 Nasıl Çalışır?

* Kullanıcılar Windows AD hesabıyla ejabberd’e login olur (`ali` / `ali@contoso.com`).
* Ejabberd mesajları middleware’e webhook gönderir.
* Middleware → AD’den mail adresini bulur → Graph API ile Teams’e mesaj gönderir.
* Teams’ten gelen mesaj → webhook ile middleware’e gelir → AD’de lookup → ejabberd HTTP API üzerinden karşı tarafa iletilir.

---

# 🔑 Yapman Gerekenler

1. Windows Server AD’de bir service account (`ldapbind`) oluştur, sadece **read** yetkisi olsun.
2. Kullanıcıların AD’de `mail` attribute’u doğru doldurulmuş olsun (Office 365 adresleri).
3. Azure AD’de bir app register et, `Client ID / Secret / Tenant ID` al.

   * İzinler: `Mail.Send`, `User.Read.All`, `Chat.ReadWrite`
4. `docker-compose up --build` ile sistemi çalıştır.

---

👉 Sana bir de **Teams Outgoing Webhook konfigürasyon örneği** (Teams → Middleware yönü) hazırlayayım mı, yoksa önce bu temel yapıyı test etmek ister misin?
