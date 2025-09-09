Harika 👍 Şimdi sıra geldi **Rule Documentation + MITRE Mapping** kısmına. Bu kısım, Detection-as-Code yaklaşımını gerçekten olgunlaştırıyor çünkü her kural sadece XML değil, aynı zamanda **neden yazıldığı** ve **hangi MITRE ATT\&CK tekniğini kapsadığı** da açıkça belgelenmiş oluyor.

---

# 📚 Rule Documentation + MITRE Mapping

## 1. Repo Yapısına `docs/rules/` Dizini Ekle

```
DaC/
├── rules/
│   └── custom_apache_rules.xml
├── docs/
│   └── rules/
│       └── R100001-apache_failed_login.md
```

---

## 2. Markdown Dokümantasyon Formatı

Her kural için bir **Markdown dosyası** (`docs/rules/R<ID>-<shortname>.md`) eklenmeli.
Örnek:

```markdown
# Rule 100001 – Apache Failed Login

## Description
Detects failed login attempts in Apache access logs.  
Triggered when multiple failed logins occur from the same source IP.

## Rule Details
- **Rule ID:** 100001
- **Level:** 8
- **Category:** Web Application Security
- **Source:** Apache access.log

## MITRE ATT&CK Mapping
- **Technique:** Brute Force
- **Technique ID:** T1110
- **Tactic:** Credential Access

## Sample Log
```

192.168.1.50 - - \[09/Mar/2025:12:45:01 +0000] "POST /login HTTP/1.1" 401 458

```

## Expected Behavior
- This log should trigger **Rule ID 100001**
- Alert level: **8**
```

---

## 3. PR Template (Zorunlu Alanlar)

`.github/pull_request_template.md` içine:

```markdown
## Rule Documentation

- [ ] Added/Updated rule XML (`rules/`)
- [ ] Added/Updated documentation (`docs/rules/`)
- [ ] Linked MITRE ATT&CK Technique (Txxxx)
- [ ] Added sample log in `tests/sample_logs/`

### Rule Information
- **Rule ID(s):**
- **MITRE ATT&CK Mapping:** Technique + ID
- **Description:**
```

👉 Böylece her PR açıldığında contributor bu bilgileri doldurmak zorunda olacak.

---

## 4. CI Kontrolü – Dokümantasyon Eksikse Fail

`.github/workflows/check_docs.yml` → Her PR’da dokümantasyon dosyası kontrol edilir.

```yaml
name: Check Rule Documentation

on:
  pull_request:
    paths:
      - "rules/**"

jobs:
  check-docs:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Ensure each rule has documentation
        run: |
          missing=0
          for rule in rules/*.xml; do
            rule_id=$(grep -oP 'id="\K[0-9]+' "$rule" | head -1)
            if [ ! -f "docs/rules/R${rule_id}-"*.md ]; then
              echo "❌ Missing documentation for Rule ID: $rule_id"
              missing=1
            fi
          done
          if [ $missing -eq 1 ]; then
            exit 1
          fi
```

---

## 5. Gelişmiş: MITRE Matrix Otomatik Raporlama

* Tüm dokümantasyon dosyaları taranarak **hangi MITRE teknikleri kapsanıyor** raporu çıkarılabilir.
* Örn: `scripts/generate_mitre_report.py` → `docs/mitre_matrix.md` oluşturur.

```bash
python scripts/generate_mitre_report.py > docs/mitre_matrix.md
```

Sonuç: Taktik bazında hangi teknikler kapsanıyor → coverage görünür.

---

# ✅ Özet

* Her kural için **Markdown dokümantasyonu** zorunlu.
* Dokümantasyon: Açıklama, MITRE mapping, örnek log, expected rule ID.
* PR template → contributor bu bilgileri doldurmalı.
* CI → eksik dokümantasyon varsa merge engelleniyor.
* (Opsiyonel) MITRE coverage raporu otomatik oluşturuluyor.

---

👉 İstersen bir sonraki adımda sana **MITRE coverage raporu üreten Python script’i** hazırlayayım mı?
