Harika 🎯 O zaman sıradaki eksik olan kısmı tamamlıyoruz: **log testleri (wazuh-logtest)**.

Amaç:

* Yeni eklenen kural sadece XML valid değil, aynı zamanda **gerçek log örnekleri** ile test edilmeli.
* Yanlış çalışıyorsa veya hiç tetiklenmiyorsa, merge olmadan yakalanmalı.

---

# 🧪 Log Test Altyapısı

## 1. Repo Yapısına `tests/` Dizini Ekle

```
DaC/
├── rules/
├── decoders/
├── tests/
│   ├── sample_logs/
│   │   └── apache_access.log
│   ├── test_rules.py
│   └── conftest.py
```

* `sample_logs/` → her kural için örnek log dosyaları
* `test_rules.py` → pytest tabanlı testler
* `conftest.py` → ortak fixture’lar

---

## 2. `test_rules.py` Örneği

```python
import subprocess
import pytest
from pathlib import Path

LOGTEST_CMD = "/var/ossec/bin/wazuh-logtest"

@pytest.mark.parametrize("logfile", Path("tests/sample_logs").glob("*.log"))
def test_logs_against_rules(logfile):
    """
    Run wazuh-logtest against sample logs and check if rules trigger.
    """
    with open(logfile, "r") as f:
        log_lines = f.readlines()

    for line in log_lines:
        proc = subprocess.run(
            [LOGTEST_CMD, "-q"], input=line.encode(),
            stdout=subprocess.PIPE, stderr=subprocess.PIPE
        )

        output = proc.stdout.decode()

        # Basic check: the rule ID should appear in the logtest output
        assert "rule:" in output, f"No rule triggered for log: {line.strip()}"
```

---

## 3. GitHub Actions Workflow (`.github/workflows/test_rules.yml`)

```yaml
name: Test Wazuh Rules with Logs

on:
  pull_request:
    paths:
      - "rules/**"
      - "decoders/**"
      - "tests/**"

jobs:
  logtest:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Wazuh logtest
        run: |
          curl -s https://packages.wazuh.com/4.x/wazuh-install.sh | bash -s -- -a
          sudo cp /var/ossec/bin/wazuh-logtest /usr/local/bin/wazuh-logtest

      - name: Run pytest log tests
        run: |
          pip install pytest
          pytest -v tests/
```

> Burada `wazuh-logtest` kurulumu için basit bir Wazuh agent setup script kullanıyoruz. İstersen **Docker içinde logtest çalıştıracak bir test container** da ayarlayabilirim.

---

## 4. Nasıl Çalışır?

1. PR açıldığında → `test_rules.yml` devreye girer.
2. `pytest` → `tests/sample_logs/*.log` dosyalarındaki satırları `wazuh-logtest` ile işler.
3. Eğer kural tetiklenmezse test fail olur → PR merge edilemez.
4. Böylece **yanlış yazılmış, gereksiz veya çalışmayan kurallar** production’a girmeden yakalanır.

---

## 5. İleri Seviye Geliştirme

* Her log için **hangi rule ID** tetiklenmesi gerektiğini `.expected` dosyaları ile tanımlayabiliriz.
  Örn: `apache_access.log.expected` içinde `100001` yazarsa → sadece bu kural beklenir.
* `pytest` bu expected ID’lerle karşılaştırma yapar.

---

👉 İstersen sana şimdi bir de **expected rule ID’leriyle çalışan gelişmiş pytest test iskeleti** hazırlayayım mı? (Yani log + beklenen rule eşleşmesi).
