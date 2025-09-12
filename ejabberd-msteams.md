Harika — aşağıda **tam çalışan örnek** sunuyorum: bu örnek bir `docker-compose` yığını içeriyor (ejabberd, OpenLDAP, Redis, middleware), ejabberd LDAP ile kullanıcıları doğruluyor ve middleware LDAP’tan kullanıcı eşlemesini (XMPP ↔ Teams) okuyor. Middleware hem **Ejabberd → Teams** hem **Teams → Ejabberd (iki yönlü, birebir mesajlaşma)** akışlarını gerçekleştirir. MS Graph API için gerekli yerlerde **placeholder** koydum; Azure AD uygulama kaydı ve gerekli izinlerle (token alıp) bunu doldurman gerekir — ilgili adımları alt bölümde özetledim.

Aşağıdaki dosya yapısını al ve projenin köküne koy:

```
xmpp-teams-bridge/
├─ docker-compose.yml
├─ ejabberd/
│  └─ ejabberd.yml
├─ ldap/
│  └─ init.ldif
├─ middleware/
│  ├─ Dockerfile
│  ├─ app.py
│  ├─ requirements.txt
│  └─ user_map.py          # (opsiyonel) önceden ek map'ler için
└─ README.md               # (açıklama, azure ad notları vs.)
```

Aşağıda her dosyanın içeriğini veriyorum — kopyala/yapıştır yapıp çalıştırabilirsin.

---

# 1) `docker-compose.yml`

```yaml
version: "3.8"
services:
  openldap:
    image: osixia/openldap:1.5.0
    environment:
      - LDAP_ORGANISATION=Example Inc
      - LDAP_DOMAIN=example.local
      - LDAP_ADMIN_PASSWORD=admin
    volumes:
      - ./ldap/init.ldif:/container/service/slapd/assets/config/bootstrap/ldif/50-bootstrap.ldif:ro
    ports:
      - "389:389"

  redis:
    image: redis:7
    ports:
      - "6379:6379"

  ejabberd:
    image: ejabberd/ecs:23.04 # veya ejabberd/ejabberd:latest
    volumes:
      - ./ejabberd/ejabberd.yml:/home/ejabberd/conf/ejabberd.yml:ro
    environment:
      - ERLANG_NODE=ejabberd@ejabberd
    depends_on:
      - openldap
    ports:
      - "5222:5222"   # client
      - "5269:5269"   # s2s
      - "5280:5280"   # http
    restart: always

  middleware:
    build: ./middleware
    volumes:
      - ./middleware:/app
    environment:
      - EJABBERD_HTTP_API=http://ejabberd:5285/api/send_message
      - EJABBERD_ADMIN=admin@example.local
      - EJABBERD_ADMIN_PASS=adminpassword
      - REDIS_URL=redis://redis:6379/0
      - LDAP_HOST=openldap
      - LDAP_PORT=389
      - LDAP_BIND_DN=cn=admin,dc=example,dc=local
      - LDAP_BIND_PASS=admin
      - LDAP_BASE_DN=dc=example,dc=local
      - GRAPH_TENANT_ID=YOUR_TENANT_ID
      - GRAPH_CLIENT_ID=YOUR_CLIENT_ID
      - GRAPH_CLIENT_SECRET=YOUR_CLIENT_SECRET
    ports:
      - "5000:5000"
    depends_on:
      - ejabberd
      - redis
      - openldap
```

---

# 2) `ldap/init.ldif` (örnek kullanıcılar)

```ldif
dn: dc=example,dc=local
objectClass: top
objectClass: dcObject
objectClass: organization
o: Example Inc
dc: example

dn: ou=users,dc=example,dc=local
objectClass: organizationalUnit
ou: users

# Ali (XMPP)
dn: uid=ali,ou=users,dc=example,dc=local
objectClass: inetOrgPerson
cn: Ali Yilmaz
sn: Yilmaz
uid: ali
userPassword: ali123
mail: ali.teams@contoso.com
displayName: Ali

# Ayse (XMPP)
dn: uid=ayse,ou=users,dc=example,dc=local
objectClass: inetOrgPerson
cn: Ayse Kaya
sn: Kaya
uid: ayse
userPassword: ayse123
mail: ayse.teams@contoso.com
displayName: Ayse
```

> `mail` attribute’ını Teams tarafındaki kullanıcı eşlemesi için kullandım. Burada `ali.teams@contoso.com` Microsoft 365/Teams e-posta adresi olacak.

---

# 3) `ejabberd/ejabberd.yml`

> Not: ejabberd’in sürümlere göre yapılandırma anahtarları farklı olabilir; kullandığın ejabberd imajının dökümantasyonuna göre `ejabberd.yml` path’ini ve modül adlarını kontrol et. Aşağıdaki temel örnek LDAP auth ve HTTP API için.

```yaml
hosts:
  - "example.local"

listen:
  -
    port: 5222
    module: ejabberd_c2s
    max_stanza_size: 65536
    shaper: c2s_shaper
    access: c2s

  -
    port: 5280
    module: ejabberd_http
    request_handlers:
      "/api": mod_http_api
      "/hooks": mod_event_pusher

auth_method: ldap

ldap:
  ldap_server: "openldap"
  ldap_port: 389
  ldap_base: "dc=example,dc=local"
  ldap_uids: "uid"
  ldap_rootdn: "cn=admin,dc=example,dc=local"
  ldap_password: "admin"
  ldap_tls: false
  user_lookup: "mail"   # optional, depends on ejabberd version

modules:
  mod_announce: {}
  mod_ping: {}
  mod_http_api: {}
  mod_event_pusher:
    webhooks:
      - url: "http://middleware:5000/xmpp-hook"
        timeout: 5
  mod_muc: {}
  mod_private: {}
  mod_offline:
    access_max_user_messages: 100

acl:
  admin:
    user:
      - "admin@example.local"
```

> Önemli: `mod_event_pusher` kullanımı ejabberd sürümüne bağlıdır. Eğer farklı bir event hook yöntemi (örn. `mod_pubsub` + custom module) kullanmanız gerekirse, ona göre adapte et.

---

# 4) `middleware/Dockerfile`

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["python", "app.py"]
```

---

# 5) `middleware/requirements.txt`

```
flask
requests
ldap3
redis
python-dotenv
msal
```

---

# 6) `middleware/user_map.py` (opsiyonel)

Bu dosya LDAP sorgularını sarmalayan, XMPP kullanıcı adından Teams e-posta adresini döndüren yardımcı fonksiyon içerir.

```python
from ldap3 import Server, Connection, ALL
import os

LDAP_HOST = os.environ.get("LDAP_HOST", "openldap")
LDAP_PORT = int(os.environ.get("LDAP_PORT", 389))
LDAP_BIND_DN = os.environ.get("LDAP_BIND_DN")
LDAP_BIND_PASS = os.environ.get("LDAP_BIND_PASS")
LDAP_BASE_DN = os.environ.get("LDAP_BASE_DN")

def get_teams_address_for_xmpp(xmpp_jid_localpart):
    """
    xmpp_jid_localpart: 'ali' (from ali@example.local)
    returns teams email like ali.teams@contoso.com or None
    """
    server = Server(LDAP_HOST, port=LDAP_PORT, get_info=ALL)
    conn = Connection(server, LDAP_BIND_DN, LDAP_BIND_PASS, auto_bind=True)
    search_filter = f"(uid={xmpp_jid_localpart})"
    conn.search(LDAP_BASE_DN, search_filter, attributes=['mail'])
    if conn.entries:
        mail = conn.entries[0].mail.value
        return mail
    return None

def get_xmpp_jid_for_teams_email(email):
    server = Server(LDAP_HOST, port=LDAP_PORT, get_info=ALL)
    conn = Connection(server, LDAP_BIND_DN, LDAP_BIND_PASS, auto_bind=True)
    search_filter = f"(mail={email})"
    conn.search(LDAP_BASE_DN, search_filter, attributes=['uid'])
    if conn.entries:
        uid = conn.entries[0].uid.value
        return f"{uid}@example.local"
    return None
```

---

# 7) `middleware/app.py` — ana bridge uygulaması

> Aşağıdaki uygulama iki endpoint sağlar:
>
> * `/xmpp-hook` — Ejabberd tarafından gelen event’leri alır (mesaj olayları) ve Teams’e iletir.
> * `/teams-hook` — Teams (Outgoing Webhook veya Bot) tarafından çağrılacak endpoint; gelen mesajı LDAP’ta eşleştirip ejabberd’e iletir.

```python
import os
import json
import requests
from flask import Flask, request, jsonify
from user_map import get_teams_address_for_xmpp, get_xmpp_jid_for_teams_email
import redis
from msal import ConfidentialClientApplication

app = Flask(__name__)

REDIS_URL = os.environ.get("REDIS_URL", "redis://localhost:6379/0")
r = redis.from_url(REDIS_URL)

# Ejabberd HTTP API
EJABBERD_HTTP_API = os.environ.get("EJABBERD_HTTP_API", "http://localhost:5285/api/send_message")
EJABBERD_ADMIN = os.environ.get("EJABBERD_ADMIN", "admin@example.local")
EJABBERD_ADMIN_PASS = os.environ.get("EJABBERD_ADMIN_PASS", "adminpassword")

# MS Graph OAuth (Confidential client)
TENANT_ID = os.environ.get("GRAPH_TENANT_ID")
CLIENT_ID = os.environ.get("GRAPH_CLIENT_ID")
CLIENT_SECRET = os.environ.get("GRAPH_CLIENT_SECRET")
SCOPES = ["https://graph.microsoft.com/.default"]
app_msal = ConfidentialClientApplication(CLIENT_ID, authority=f"https://login.microsoftonline.com/{TENANT_ID}", client_credential=CLIENT_SECRET)

def get_graph_token():
    result = app_msal.acquire_token_silent(SCOPES, account=None)
    if not result:
        result = app_msal.acquire_token_for_client(scopes=SCOPES)
    if "access_token" in result:
        return result["access_token"]
    else:
        app.logger.error("Graph token error: %s", result)
        return None

def prevent_loop_and_store(msg_id):
    """
    Basit loop engelleme: redis'te msg_id sakla. 
    Eğer zaten varsa True döndür (yani işlememe).
    """
    if not msg_id:
        return False
    key = f"msg:{msg_id}"
    if r.get(key):
        return True
    r.set(key, "1", ex=60*5)  # 5 dakika tut
    return False

@app.route("/xmpp-hook", methods=["POST"])
def xmpp_hook():
    """
    Ejabberd'den gelen JSON payload beklenir:
    {
      "type": "message",
      "from": "ali@example.local",
      "to": "ayse@example.local",
      "body": "Merhaba",
      "id": "uuid-123"
    }
    """
    data = request.get_json() or {}
    msg_id = data.get("id")
    if prevent_loop_and_store(msg_id):
        return jsonify({"status": "loop, ignored"}), 200

    sender = data.get("from")
    to = data.get("to")
    body = data.get("body", "")

    # localpart çek (ali from ali@example.local)
    to_local = to.split("@")[0]
    teams_address = get_teams_address_for_xmpp(to_local)
    if not teams_address:
        return jsonify({"error": "Teams mapping not found"}), 404

    # Graph API: create or find 1:1 chat between app and user, then send message
    token = get_graph_token()
    if not token:
        return jsonify({"error": "no graph token"}), 500

    headers = {"Authorization": f"Bearer {token}", "Content-Type": "application/json"}

    # 1) Create chat (if not exists) - here we attempt to create a oneOnOne chat with user and app
    create_chat_payload = {
      "chatType": "oneOnOne",
      "members": [
        {
          "@odata.type": "#microsoft.graph.aadUserConversationMember",
          "roles": ["owner"],
          "user@odata.bind": f"https://graph.microsoft.com/v1.0/users('{teams_address}')"
        },
        # If using application as participant, see Graph docs for app-as-member pattern (might require special setup).
      ]
    }
    # Attempt to create chat - if exists, Graph will return conflict or we need to search existing chats.
    # For simplicity, try create and then use returned chat id.
    chat_id = None
    create_resp = requests.post("https://graph.microsoft.com/v1.0/chats", headers=headers, data=json.dumps(create_chat_payload))
    if create_resp.status_code in (201, 200):
        chat_id = create_resp.json().get("id")
    else:
        # Fallback: search user's chats is more complex; we will attempt to proceed with creating message in chat via /users/{id}/chats
        # As simpler fallback, send mail (if chat send fails) or log error.
        app.logger.warning("Could not create chat: %s %s", create_resp.status_code, create_resp.text)
        return jsonify({"error": "could not create chat", "detail": create_resp.text}), 500

    # 2) Send message
    msg_payload = {
      "body": {
        "contentType": "text",
        "content": f"{sender}: {body}"
      }
    }
    send_resp = requests.post(f"https://graph.microsoft.com/v1.0/chats/{chat_id}/messages", headers=headers, data=json.dumps(msg_payload))
    if send_resp.status_code in (201, 200):
        return jsonify({"status": "sent to teams"}), 200
    else:
        app.logger.error("send message failed: %s %s", send_resp.status_code, send_resp.text)
        return jsonify({"error": "send failed", "detail": send_resp.text}), 500

@app.route("/teams-hook", methods=["POST"])
def teams_hook():
    """
    Teams tarafının outgoing webhook veya bot'dan gönderdiği JSON beklenir.
    Örnek simple outgoing webhook payload (teams outgoing webhook farklı formatlarda olabilir).
    Hedef: teams email'den xmpp_jid bulup ejabberd'e gönder.
    """
    data = request.get_json() or {}
    # Shrink different shapes: try common fields
    from_email = None
    if "from" in data and isinstance(data["from"], dict):
        from_email = data["from"].get("email")
    if not from_email:
        from_email = data.get("user", {}).get("email") or data.get("from", {}).get("user", {}).get("email")
    text = data.get("text") or data.get("message") or data.get("body", {}).get("content")

    xmpp_jid = get_xmpp_jid_for_teams_email(from_email)
    if not xmpp_jid:
        return jsonify({"error": "no xmpp mapping"}), 404

    # prevent loops using a synthetic id if present
    msg_id = data.get("id")
    if prevent_loop_and_store(msg_id):
        return jsonify({"status": "loop ignored"}), 200

    payload = {
      "type": "chat",
      "from": EJABBERD_ADMIN,
      "to": xmpp_jid,
      "body": f"[Teams] {from_email}: {text}"
    }
    resp = requests.post(EJABBERD_HTTP_API, json=payload, auth=(EJABBERD_ADMIN, EJABBERD_ADMIN_PASS))
    if resp.status_code in (200, 201):
        return jsonify({"status": "sent to xmpp"}), 200
    else:
        app.logger.error("ejabberd api error: %s %s", resp.status_code, resp.text)
        return jsonify({"error": "ejabberd send failed", "detail": resp.text}), 500

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000, debug=True)
```

> Notlar:
>
> * Microsoft Graph ile chat oluşturma / mesaj gönderme işlemleri uygulama türüne göre (application permission vs delegated) farklı yetkilendirme gerektirir. Yukarıdaki `create chat` ve `send message` akışı için Graph `Chat.Create` / `ChatMessage.Send` benzeri izinler ve doğru app registration gereklidir. Eğer bot framework kullanırsan (bot registration) farklı bir akış izlenir.

---

# 8) Önemli konular — dikkat etmen gerekenler (adım adım)

1. **Azure AD / Graph izinleri**

   * Uygulaman (Client ID / Secret) için:

     * `Chats.ReadWrite` (application scope) veya ilgili `ChatMessage.Send` izinleri (Graph) — *uygulama izinleri* (application permissions) gerekebilir.
     * `User.Read.All` (kullanıcı aramak için) veya `User.ReadBasic.All`.
   * Tenant admin tarafından onaylanması gerekiyor.
   * `msal` kullanarak client credentials flow ile token alıp `Authorization: Bearer ...` kullanıyoruz.

2. **Loop (döngü) önleme**

   * Yukarıdaki örnek Redis ile kısa süreli mesaj-id saklıyor. Gerçek uygulamada daha sağlam bir dedup mekanizması gerekir:

     * Mesaj kaynak etiketi (ör. `[Teams]` veya `via-bridge`) ve message-id ile kontrol.
     * Mesajlarda meta veri (custom stanza) koymak mümkünse.

3. **Kullanıcı eşleşmesi**

   * Örnekte LDAP `mail` attribute’u kullanıldı. Senin ortamında başka attribute (e.g. `employeeNumber`, `proxyAddresses`) olabilir; `user_map.py` içinde değiştir.
   * Eğer bazı kullanıcıların eşleşmesi yoksa fallback davranışı (örn. “Kullanıcı Teams hesabı yok”) tasarla.

4. **Yetki & impersonation**

   * Ejabberd’e mesaj gönderirken `mod_http_api` kullanıp admin kimliğiyle gönderiyoruz. Bu yöntem basit ama üretimde audit / güvenlik açısından değerlendirilmeli.
   * Teams tarafında da uygulama mesaj atarken hangi kullanıcı olarak görüneceği (app-as-user mı yoksa bot mu) konusu önemlidir.

5. **Gelişmiş: Presence / roster / vCard sync**

   * İstersen LDAP’tan roster (arkadaş listesi) senkronizasyonu veya presence (online/toplantı) senkronizasyonu eklenebilir.

---

# 9) Nasıl çalıştırırsın (quickstart)

1. Dosyaları oluştur ve `docker-compose up --build` çalıştır.
2. OpenLDAP 389 portunda ayağa kalkacak; ejabberd LDAP’ı görüp auth yapabilmeli.
3. `middleware` container’ı Redis’e bağlanacak ve 5000 portunu açacak.
4. Azure AD app bilgilerini `docker-compose.yml` environment içinde doldur. (veya `.env` kullan)

---

# 10) Sonraki adımlar — ben yardımcı olabilirim

İstersen şu konularda sana adım adım yardımcı olurum:

* Azure AD uygulama izinlerinin tam listesini ve nasıl onaylatılacağını (ben örnek API call / izinleri gösteririm).
* Middleware kodunu **canlı test edilebilir** hale getirecek örnek POST’lar (curl) ile test senaryoları.
* Kullanıcı eşlemesini LDAP yerine CSV veya bir UI ile yönetilecek hale getirme.
* Loop / idempotency için daha güvenli strateji (message headers, provenance metadata).

Hangi adımla devam edelim? (Örn. “Azure AD app ve Graph izinlerini detaylandır” veya “Benim için örnek bir curl ile xmpp→teams ve teams→xmpp testleri hazırla”)
