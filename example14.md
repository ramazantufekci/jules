Süper 👍 O zaman sana **canary + rollback mekanizmalı** bir `deploy_rules.yml` hazırlayayım.
Bu pipeline şu adımlarla çalışır:

1. **Canary Deploy** → Önce sadece **ilk sunucuya** kurallar yüklenir, validasyon yapılır, restart edilir.
2. Eğer canary başarılı olursa → **tüm cluster’a dağıtım** yapılır.
3. Eğer canary ya da cluster dağıtım sırasında bir sorun çıkarsa → **rollback** devreye girer.

---

# 🚀 `.github/workflows/deploy_rules.yml`

```yaml
name: Deploy Wazuh Rules (Canary + Rollback)

on:
  push:
    branches:
      - main
    paths:
      - "rules/**"
      - "decoders/**"

jobs:
  canary:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Deploy to Canary Host
        uses: appleboy/ssh-action@v1.1.0
        with:
          host: ${{ secrets.WAZUH_CANARY_HOST }}
          username: ${{ secrets.WAZUH_SERVER_USER }}
          key: ${{ secrets.WAZUH_SERVER_SSH_KEY }}
          port: ${{ secrets.WAZUH_SSH_PORT }}
          script: |
            echo "🔄 Deploying rules to Canary..."
            sudo mkdir -p /var/ossec/etc/rules_new
            sudo rm -rf /var/ossec/etc/rules_new/*
            exit_code=0

            # Copy rules
            cat > /tmp/rules.tar.gz <<'EOF'
            $(tar czf - rules/)
            EOF
            tar xzf /tmp/rules.tar.gz -C /var/ossec/etc/rules_new/

            # Validate XML
            xmllint --noout /var/ossec/etc/rules_new/*.xml || exit_code=$?

            if [ $exit_code -ne 0 ]; then
              echo "❌ Canary XML validation failed"
              exit 1
            fi

            # Backup old rules
            sudo rm -rf /var/ossec/etc/rules_bak
            sudo mv /var/ossec/etc/rules /var/ossec/etc/rules_bak

            # Switch to new rules
            sudo mv /var/ossec/etc/rules_new /var/ossec/etc/rules

            # Restart manager
            if ! sudo systemctl restart wazuh-manager; then
              echo "❌ Restart failed, rolling back..."
              sudo rm -rf /var/ossec/etc/rules
              sudo mv /var/ossec/etc/rules_bak /var/ossec/etc/rules
              sudo systemctl restart wazuh-manager
              exit 1
            fi

            echo "✅ Canary deploy successful"

  deploy-cluster:
    runs-on: ubuntu-latest
    needs: canary
    if: success()
    strategy:
      matrix:
        host: ${{ fromJSON(format('["{0}"]', join(secrets.WAZUH_HOSTS, '","'))) }}
    steps:
      - uses: actions/checkout@v4

      - name: Deploy to ${{ matrix.host }}
        uses: appleboy/ssh-action@v1.1.0
        with:
          host: ${{ matrix.host }}
          username: ${{ secrets.WAZUH_SERVER_USER }}
          key: ${{ secrets.WAZUH_SERVER_SSH_KEY }}
          port: ${{ secrets.WAZUH_SSH_PORT }}
          script: |
            echo "🔄 Deploying rules to $HOSTNAME..."
            sudo mkdir -p /var/ossec/etc/rules_new
            sudo rm -rf /var/ossec/etc/rules_new/*
            exit_code=0

            # Copy rules
            cat > /tmp/rules.tar.gz <<'EOF'
            $(tar czf - rules/)
            EOF
            tar xzf /tmp/rules.tar.gz -C /var/ossec/etc/rules_new/

            # Validate XML
            xmllint --noout /var/ossec/etc/rules_new/*.xml || exit_code=$?

            if [ $exit_code -ne 0 ]; then
              echo "❌ Validation failed on $HOSTNAME"
              exit 1
            fi

            # Backup old rules
            sudo rm -rf /var/ossec/etc/rules_bak
            sudo mv /var/ossec/etc/rules /var/ossec/etc/rules_bak

            # Switch to new rules
            sudo mv /var/ossec/etc/rules_new /var/ossec/etc/rules

            # Restart manager
            if ! sudo systemctl restart wazuh-manager; then
              echo "❌ Restart failed, rolling back..."
              sudo rm -rf /var/ossec/etc/rules
              sudo mv /var/ossec/etc/rules_bak /var/ossec/etc/rules
              sudo systemctl restart wazuh-manager
              exit 1
            fi

            echo "✅ Deploy successful on $HOSTNAME"
```

---

## 🔑 Gereken Secrets

* `WAZUH_CANARY_HOST` → Canary için kullanılacak tek bir host (örn: test ortamı).
* `WAZUH_HOSTS` → Cluster’daki diğer tüm hostlar (örn: `10.0.1.11,10.0.2.12`).
* `WAZUH_SERVER_USER`, `WAZUH_SERVER_SSH_KEY`, `WAZUH_SSH_PORT` → SSH erişim bilgileri.

---

## ✅ Özellikler

* Canary deploy → önce tek hostta test.
* XML validasyon.
* Restart başarısız olursa rollback.
* Canary başarılı olursa cluster dağıtımı başlar.
* Her host için rollback mekanizması var.

---

👉 İstersen buna bir de **Slack/Teams bildirim job’u** ekleyelim mi, böylece deploy sonucunu SOC ekibine otomatik göndersin?
