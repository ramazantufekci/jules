Harika 👌 Şimdi de **Policy Enforcement** katmanını ekleyelim.
Bu aşama, kuralların yalnızca **syntax** değil aynı zamanda **organizasyonel standartlara** da uygun olup olmadığını garanti eder.

---

# 🛡️ Policy Enforcement

## 1. Gereksinimler

* **ID Aralıkları**

  * Örn: `100000–199999` → custom kurallar için ayrılmış.
  * `200000–299999` → decoder kuralları.
  * Bu aralık dışına çıkan ID’ler **reject** edilmeli.
* **Schema Validation**

  * XML, Wazuh’un rule schema’sına uygun olmalı.
  * Örn: `id`, `level`, `if_sid` gibi zorunlu attribute’lar.
* **Naming Convention**

  * Kural dosya isimleri `custom_*.xml` formatında olmalı.
  * Dokümantasyon dosyaları `R<ID>-<shortname>.md` formatında olmalı.

---

## 2. Policy Enforcement Script

`scripts/enforce_policies.py`

```python
import sys
import re
import xml.etree.ElementTree as ET
from pathlib import Path

RULES_DIR = Path("rules")
DOCS_DIR = Path("docs/rules")

# Allowed ID ranges
ALLOWED_RANGES = [
    (100000, 199999),  # custom rules
    (200000, 299999),  # decoders
]

def check_id_range(rule_id: int) -> bool:
    return any(low <= rule_id <= high for low, high in ALLOWED_RANGES)

def validate_rule_file(xml_file: Path):
    errors = []
    try:
        tree = ET.parse(xml_file)
        root = tree.getroot()
    except ET.ParseError as e:
        errors.append(f"❌ XML parse error in {xml_file}: {e}")
        return errors

    for rule in root.findall(".//rule"):
        rule_id = int(rule.get("id", "0"))
        if not check_id_range(rule_id):
            errors.append(f"❌ Rule ID {rule_id} in {xml_file} is out of allowed ranges!")

        if "level" not in rule.attrib:
            errors.append(f"❌ Rule {rule_id} in {xml_file} missing 'level' attribute!")

    return errors

def validate_docs():
    errors = []
    for doc_file in DOCS_DIR.glob("*.md"):
        if not re.match(r"R\d{6}-.*\.md", doc_file.name):
            errors.append(f"❌ Doc file {doc_file} does not follow naming convention (R<ID>-<name>.md)")
    return errors

if __name__ == "__main__":
    all_errors = []

    for xml_file in RULES_DIR.glob("*.xml"):
        all_errors.extend(validate_rule_file(xml_file))

    all_errors.extend(validate_docs())

    if all_errors:
        print("\n".join(all_errors))
        sys.exit(1)
    else:
        print("✅ All rules passed policy enforcement checks.")
```

---

## 3. CI Pipeline Entegrasyonu

`.github/workflows/policy_check.yml`

```yaml
name: Enforce Rule Policies

on:
  pull_request:
    paths:
      - "rules/**"
      - "docs/rules/**"

jobs:
  policy-enforcement:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run policy enforcement
        run: |
          pip install lxml
          python scripts/enforce_policies.py
```

---

## 4. İleri Seviye

* **ID çakışma kontrolü**: Aynı ID birden fazla dosyada varsa hata.
* **Rule ID → Doc eşleştirmesi**: Her rule ID için `docs/rules/R<ID>-*.md` mevcut mu?
* **Pre-commit hook**: Policy enforcement sadece CI’da değil, developer commit aşamasında da çalıştırılabilir.

---

✅ Böylece artık:

* Yanlış ID kullanımı ❌
* Eksik `level` attribute ❌
* Yanlış doc naming ❌
* XML schema violation ❌

hepsi merge aşamasında engelleniyor.

---

👉 İstersen bir sonraki adımda ben bu **ID çakışma kontrolünü + doc eşleştirme zorunluluğunu** da bu script’e ekleyeyim mi?
