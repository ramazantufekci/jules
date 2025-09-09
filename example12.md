Harika 👍 O zaman mevcut geliştirilmiş yapıyı **multi-cluster** desteği ile genişletelim. Böylece aynı kural seti birden fazla **Wazuh manager**’a (örneğin prod, dr, test ortamlarına) otomatik dağıtılabilecek.

---

# 🔥 Multi-Cluster Destekli DaC Yapısı

## 1. GitHub Secrets Düzeni

Tek sunucu yerine birden fazla host için `secrets` ayarlıyoruz:

| Secret Adı             | Açıklama                                                                          |
| ---------------------- | --------------------------------------------------------------------------------- |
| `WAZUH_HOSTS`          | Virgülle ayrılmış IP veya hostname listesi (örn: `10.0.1.10,10.0.2.10,10.0.3.10`) |
| `WAZUH_SERVER_USER`    | SSH kullanıcı adı                                                                 |
| `WAZUH_SERVER_SSH_KEY` | Özel SSH anahtar içeriği                                                          |
| `WAZUH_SSH_PORT`       | SSH portu (default: 22)                                                           |

---

## 2. Multi-Cluster Deploy Workflow

```yaml
# .github/workflows/deploy_rules.yml
name: Deploy Wazuh Rules (Multi-Cluster)

on:
  push:
    branches:
      - main
    paths:
      - "rules/**"
      - "decoders/**"

jobs:
  deploy:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        host: ${{ fromJSON(format('["{0}"]', join(secrets.WAZUH_HOSTS, '","'))) }}
    steps:
      - uses: actions/checkout@v4

      - name: Copy rules to staging dir on ${{ matrix.host }}
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ matrix.host }}
          username: ${{ secrets.WAZUH_SERVER_USER }}
          key: ${{ secrets.WAZUH_SERVER_SSH_KEY }}
          port: ${{ secrets.WAZUH_SSH_PORT }}
          source: "rules/*"
          target: "/var/ossec/etc/rules_new/"

      - name: Validate & Switch Directories on ${{ matrix.host }}
        uses: appleboy/ssh-action@v1.1.0
        with:
          host: ${{ matrix.host }}
          username: ${{ secrets.WAZUH_SERVER_USER }}
          key: ${{ secrets.WAZUH_SERVER_SSH_KEY }}
          port: ${{ secrets.WAZUH_SSH_PORT }}
          script: |
            echo "🔍 Validating rules on $HOSTNAME..."
            xmllint --noout /var/ossec/etc/rules_new/*.xml
            if [ $? -eq 0 ]; then
              echo "✅ Validation passed on $HOSTNAME"
              sudo rm -rf /var/ossec/etc/rules_bak
              sudo mv /var/ossec/etc/rules /var/ossec/etc/rules_bak
              sudo mv /var/ossec/etc/rules_new /var/ossec/etc/rules
              sudo systemctl restart wazuh-manager
              echo "🚀 Deployment finished on $HOSTNAME"
            else
              echo "❌ XML Validation failed on $HOSTNAME"
              exit 1
            fi
```

---

## 3. Çalışma Prensibi

1. `WAZUH_HOSTS` → Örn: `10.0.1.10,10.0.2.10,10.0.3.10` şeklinde secrets’a girilir.
2. Workflow `matrix` ile bu listeyi ayırır ve her sunucuya **paralel job** başlatır.
3. Her sunucuda:

   * `rules_new/` klasörüne kopyalanır.
   * XML validasyon yapılır.
   * Başarılıysa `rules/` ile değiştirilir ve `wazuh-manager` restart edilir.
   * Hata varsa sadece o sunucuda job fail olur.

---

## 4. Ekstra Güvenlik İyileştirmeleri

* **Canary Deploy:** Önce tek bir manager’a dağıtıp test ettikten sonra diğerlerine yaymak için pipeline’a `needs: [canary]` eklenebilir.
* **Slack/Teams Bildirimi:** Başarılı / başarısız deploy sonrası DevOps kanalına bildirim göndermek için ek job eklenebilir.
* **Rollback:** Eğer restart sonrası hata varsa, `rules_bak/` klasörü geri yüklenebilir.

---

👉 Böylece artık aynı repo → aynı pipeline üzerinden **birden fazla Wazuh cluster** güncelleniyor.

İstersen ben sana bunun bir de **canary + rollback mekanizmalı versiyonunu** hazırlayayım mı, yoksa şimdilik paralel dağıtım yeterli mi?
