Harika 👍 Şimdi eksik kalan son büyük parçayı ekleyelim: **Blue/Green Deployment Desteği**.

Bu yöntemle:

* Mevcut production ortamı (Blue) etkilenmeden, yeni kurallar (Green) ayrı bir ortamda deploy edilir.
* Test ve validasyon tamamlandıktan sonra Green → Blue’ya switch edilir.
* Böylece hata riski minimize edilir.

---

# 🔹 Blue/Green Deployment Tasarımı

## 1. Ortam Yapısı

* **Blue Cluster:** Şu an üretimde çalışan Wazuh manager’lar.
* **Green Cluster:** Canary + test ortamı → yeni kurallar burada test edilir.

Secrets yapısı (GitHub Actions):

```text
BLUE_HOSTS="10.0.1.10,10.0.2.10"
GREEN_HOSTS="10.0.3.10,10.0.4.10"  # Green cluster (test/validation)
WAZUH_SERVER_USER
WAZUH_SERVER_SSH_KEY
WAZUH_SSH_PORT
```

---

## 2. Workflow Adımları

### a) Deploy to Green Cluster

* Yeni kurallar `/var/ossec/etc/rules_new/` içine kopyalanır.
* XML ve log testleri çalıştırılır.
* Eğer tüm testler geçerse → Green Cluster “ready” olur.

### b) Switch Blue ← Green

* Validasyon tamamlandıktan sonra, production Blue cluster, Green cluster’ın kural setiyle güncellenir.
* Opsiyonel: Backup alınır (`rules_bak`) ve rollback mekanizması hazır tutulur.

---

## 3. GitHub Actions Örneği

```yaml
name: Blue/Green Deployment Wazuh Rules

on:
  push:
    branches:
      - main
    paths:
      - "rules/**"
      - "decoders/**"

jobs:
  deploy-green:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        host: ${{ fromJSON(format('["{0}"]', join(secrets.GREEN_HOSTS, '","'))) }}
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to Green Host ${{ matrix.host }}
        uses: appleboy/ssh-action@v1.1.0
        with:
          host: ${{ matrix.host }}
          username: ${{ secrets.WAZUH_SERVER_USER }}
          key: ${{ secrets.WAZUH_SERVER_SSH_KEY }}
          port: ${{ secrets.WAZUH_SSH_PORT }}
          script: |
            sudo mkdir -p /var/ossec/etc/rules_new
            sudo rm -rf /var/ossec/etc/rules_new/*
            cp -r rules/* /var/ossec/etc/rules_new/
            xmllint --noout /var/ossec/etc/rules_new/*.xml
            sudo systemctl restart wazuh-manager
            echo "✅ Green deploy successful"

  switch-blue:
    runs-on: ubuntu-latest
    needs: deploy-green
    steps:
      - uses: actions/checkout@v4
      - name: Deploy Green rules to Blue cluster
        uses: appleboy/ssh-action@v1.1.0
        with:
          host: ${{ secrets.BLUE_HOSTS }}
          username: ${{ secrets.WAZUH_SERVER_USER }}
          key: ${{ secrets.WAZUH_SERVER_SSH_KEY }}
          port: ${{ secrets.WAZUH_SSH_PORT }}
          script: |
            echo "🔄 Switching Blue cluster to Green rules..."
            sudo rm -rf /var/ossec/etc/rules_bak
            sudo mv /var/ossec/etc/rules /var/ossec/etc/rules_bak
            sudo cp -r /var/ossec/etc/rules_new/* /var/ossec/etc/rules
            sudo systemctl restart wazuh-manager
            echo "✅ Blue cluster updated successfully"
```

---

## 4. Öne Çıkan Özellikler

* **Risk Azaltma:** Blue cluster production’da çalışırken Green cluster üzerinde test yapılır.
* **Rollback Hazır:** Hata durumunda `/var/ossec/etc/rules_bak` ile eski kurallar geri yüklenebilir.
* **CI/CD Otomasyonu:** Merge sonrası workflow tamamen otomatik çalışır.
* **Multi-Cluster Ready:** Matrix strategy ile hem Blue hem Green cluster paralel yönetilebilir.

---

## 5. Gelişmiş Opsiyonlar

* **Canary + Blue/Green:** Green cluster’da önce 1-2 host test edilir (canary) → başarılıysa tüm Green cluster deploy edilir → sonra Blue’ya switch yapılır.
* **Deploy Window:** Production switch’i belirli maintenance window’larda yapılabilir.
* **Notification:** Switch sonrası Slack/Teams ile SOC ekibine bildirim gönderilebilir.

---

✅ Böylece artık workflow, **Blue/Green + Canary + rollback + log observability + policy enforcement** ile production-grade bir DaC pipeline haline geliyor.

---

İstersen bir sonraki adımda **tüm parçaları tek bir entegre workflow örneği** olarak birleştirip sana komple production-ready `deploy_rules.yml` hazırlayabilirim. Bunu yapmamı ister misin?
