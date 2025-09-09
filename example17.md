Süper 👍 O zaman sana **MITRE coverage raporu üreten Python script’i** hazırlıyorum.
Bu script, `docs/rules/` klasöründeki **Markdown dosyalarını tarar**, içindeki **MITRE Technique ID** değerlerini çıkarır ve **MITRE tactics → teknikler** şeklinde bir rapor üretir.

---

# 📜 `scripts/generate_mitre_report.py`

```python
import os
import re
from collections import defaultdict
from pathlib import Path

DOCS_DIR = Path("docs/rules")

# Regex to find MITRE Technique IDs (örn: T1110, T1059.001)
TECHNIQUE_REGEX = re.compile(r"T\d{4}(?:\.\d{3})?")

def parse_docs():
    mitre_mapping = defaultdict(list)

    for doc_file in DOCS_DIR.glob("*.md"):
        with open(doc_file, "r", encoding="utf-8") as f:
            content = f.read()

        # Extract Rule ID from filename (R<ID>-...)
        match = re.match(r"R(\d+)-", doc_file.name)
        rule_id = match.group(1) if match else "UNKNOWN"

        # Extract technique IDs
        techniques = TECHNIQUE_REGEX.findall(content)
        if not techniques:
            continue

        # Extract tactic if available
        tactic_match = re.search(r"Tactic:\s*(.+)", content)
        tactic = tactic_match.group(1).strip() if tactic_match else "Unspecified"

        for tech in techniques:
            mitre_mapping[tactic].append((rule_id, tech, doc_file.name))

    return mitre_mapping


def generate_report(mitre_mapping):
    lines = []
    lines.append("# MITRE ATT&CK Coverage Report\n")
    for tactic, items in mitre_mapping.items():
        lines.append(f"## {tactic}\n")
        for rule_id, technique, doc in items:
            lines.append(f"- **Rule {rule_id}** → {technique} (📄 {doc})")
        lines.append("")  # boş satır
    return "\n".join(lines)


if __name__ == "__main__":
    mapping = parse_docs()
    report = generate_report(mapping)
    report_path = Path("docs/mitre_matrix.md")
    with open(report_path, "w", encoding="utf-8") as f:
        f.write(report)
    print(f"✅ MITRE coverage report generated at {report_path}")
```

---

# 📊 Örnek Çıktı (`docs/mitre_matrix.md`)

```markdown
# MITRE ATT&CK Coverage Report

## Credential Access
- **Rule 100001** → T1110 (📄 R100001-apache_failed_login.md)

## Execution
- **Rule 100002** → T1059.001 (📄 R100002-powershell_exec.md)
```

---

# ⚙️ CI Pipeline Entegrasyonu

`.github/workflows/generate_mitre.yml`

```yaml
name: Generate MITRE Coverage Report

on:
  push:
    branches:
      - main
    paths:
      - "docs/rules/**"

jobs:
  mitre-report:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run MITRE report generator
        run: |
          pip install pathlib
          python scripts/generate_mitre_report.py
      - name: Commit MITRE coverage report
        uses: stefanzweifel/git-auto-commit-action@v5
        with:
          commit_message: "Update MITRE coverage report"
          file_pattern: docs/mitre_matrix.md
```

---

✅ Artık her yeni kural eklendiğinde PR merge edilince **MITRE coverage raporu** otomatik güncellenir.
SOC ekibi direkt `docs/mitre_matrix.md` dosyasından hangi teknikleri kapsadığınızı görebilir.

---

👉 İstersen bir sonraki adımda sana **Slack/Teams bildirim entegrasyonunu** da ekleyeyim mi, böylece deploy ve test sonuçları otomatik bildirilir?
