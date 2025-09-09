Süper 👍 şimdi sıradaki parçayı ekleyelim: **Observability (deploy sonrası log toplanması)**.

Şu an kuralları deploy ediyoruz, restart ediyoruz ama:

* `wazuh-manager` gerçekten düzgün başladı mı?
* XML veya rule hataları var mı?
* Deploy sonrası uyarılar loglarda gözüküyor mu?

bunları CI/CD çıktısına taşımadık. Bunun için **log toplama & artifact** mekanizması kuralım.

---

# 🔍 Deploy Sonrası Log Toplama

## 1. Deploy Workflow’a Ek Adım (Her Host İçin)

`deploy_rules.yml` içine (canary ve cluster deploy job’larına) eklenebilir:

```yaml
      - name: Collect wazuh-manager logs on ${{ matrix.host }}
        uses: appleboy/ssh-action@v1.1.0
        with:
          host: ${{ matrix.host }}
          username: ${{ secrets.WAZUH_SERVER_USER }}
          key: ${{ secrets.WAZUH_SERVER_SSH_KEY }}
          port: ${{ secrets.WAZUH_SSH_PORT }}
          script: |
            echo "📜 Collecting logs from wazuh-manager..."
            sudo journalctl -u wazuh-manager --since "-2m" --no-pager > /tmp/wazuh_${{ matrix.host }}.log
        continue-on-error: true

      - name: Copy logs as artifact
        uses: actions/upload-artifact@v4
        with:
          name: wazuh-logs-${{ matrix.host }}
          path: /tmp/wazuh_${{ matrix.host }}.log
```

---

## 2. Canary Job İçin Benzer Ek

```yaml
      - name: Collect wazuh-manager logs (canary)
        uses: appleboy/ssh-action@v1.1.0
        with:
          host: ${{ secrets.WAZUH_CANARY_HOST }}
          username: ${{ secrets.WAZUH_SERVER_USER }}
          key: ${{ secrets.WAZUH_SERVER_SSH_KEY }}
          port: ${{ secrets.WAZUH_SSH_PORT }}
          script: |
            echo "📜 Collecting canary logs..."
            sudo journalctl -u wazuh-manager --since "-2m" --no-pager > /tmp/wazuh_canary.log
        continue-on-error: true

      - name: Upload Canary Logs
        uses: actions/upload-artifact@v4
        with:
          name: wazuh-logs-canary
          path: /tmp/wazuh_canary.log
```

---

## 3. Sonuç

* Deploy sonrası her sunucunun son **2 dakikalık `wazuh-manager` logları** pipeline artifact olarak saklanıyor.
* Eğer deployment başarısız olursa ya da manager loglarında hata varsa → CI/CD pipeline çıktısından direkt indirip inceleyebilirsin.
* SOC ekibi geçmiş deploy loglarını pipeline run’larından indirebilir.

---

## 4. İleri Seviye Geliştirme

* **Log Parsing**: CI pipeline loglarını parse edip belirli keyword’ler (`ERROR`, `Invalid decoder`, `Invalid rule`) bulunduğunda job fail etsin.
* **Centralized Observability**: Artifact olarak saklamanın ötesinde, deploy logları **ELK / Grafana Loki / S3** gibi merkezi bir yere gönderilebilir.
* **Alerting**: Eğer loglarda hata varsa Slack/Teams bildirimi gönderilebilir.

---

✅ Böylece artık pipeline sadece deploy etmiyor, aynı zamanda **deploy sonrası doğrulama loglarını** topluyor → gerçek bir **observability** katmanı eklenmiş oluyor.

---

👉 İstersen ben sana bir de **log parsing + otomatik fail mekanizması** ekleyeyim mi (ör. loglarda “ERROR” görünürse pipeline direkt dursun)?
