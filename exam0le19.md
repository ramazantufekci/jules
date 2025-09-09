Süper 👌 O zaman deploy sonrası sadece logları toplamakla kalmayalım, aynı zamanda **otomatik analiz edelim**:

* `wazuh-manager` logları toplandıktan sonra
* Eğer loglarda `ERROR`, `Invalid decoder`, `Invalid rule` gibi kritik hatalar varsa → **pipeline fail etsin**
* Böylece sorunlu deploy production’a ulaşmaz.

---

# ⚙️ Log Parsing + Fail Mekanizması

## 1. Workflow’a Ek Adım (Cluster & Canary Deploy)

`deploy_rules.yml` içine log toplama sonrası şu step eklenebilir:

```yaml
      - name: Analyze Wazuh logs for errors
        run: |
          logfile="/tmp/wazuh_${{ matrix.host }}.log"
          if grep -E "ERROR|Invalid decoder|Invalid rule" "$logfile"; then
            echo "❌ Found critical errors in wazuh-manager logs!"
            exit 1
          else
            echo "✅ No critical errors found in wazuh-manager logs."
          fi
```

Canary için:

```yaml
      - name: Analyze Canary logs
        run: |
          logfile="/tmp/wazuh_canary.log"
          if grep -E "ERROR|Invalid decoder|Invalid rule" "$logfile"; then
            echo "❌ Found errors in canary wazuh-manager logs!"
            exit 1
          else
            echo "✅ Canary logs are clean."
          fi
```

---

## 2. Gelişmiş: Hata Türüne Göre Aksiyon

İstersen hataya göre farklı davranabiliriz:

* `Invalid decoder` → Canary rollback
* `Invalid rule` → Deploy fail
* `ERROR` → sadece warning

Örn. daha akıllı analiz için küçük bir script:

```bash
#!/bin/bash
logfile="$1"
fail=0

if grep -q "Invalid decoder" "$logfile"; then
  echo "❌ Invalid decoder detected!"
  fail=1
fi

if grep -q "Invalid rule" "$logfile"; then
  echo "❌ Invalid rule detected!"
  fail=1
fi

if grep -q "ERROR" "$logfile"; then
  echo "⚠️  Generic ERROR found in logs."
fi

exit $fail
```

Bunu `scripts/check_wazuh_logs.sh` olarak koyup CI’da şöyle çağırabiliriz:

```yaml
      - name: Run log analyzer
        run: bash scripts/check_wazuh_logs.sh /tmp/wazuh_${{ matrix.host }}.log
```

---

## 3. Sonuç

* Deploy sonrası loglar **artifact** olarak saklanıyor ✅
* Aynı zamanda CI log analizi ile **otomatik kontrol** yapılıyor ✅
* Eğer kritik bir hata varsa → job fail oluyor, rollback tetikleniyor ✅

---

👉 İstersen ben sana bu adımı daha da geliştireyim ve **Slack/Teams entegrasyonu** ekleyeyim: yani loglarda hata bulunduğunda SOC ekibine otomatik bildirim gitsin. Bunu ister misin?
