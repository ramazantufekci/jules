Got it 👍 — you want to **catch Windows Event ID 1100** in Wazuh.

---

## 🔎 What Event ID 1100 Means

* **1100** – *The event logging service has shut down.*
  This comes from the **EventLog** source in the **EventLog channel**.
  ⚠️ It’s a **highly critical event**, because if the logging service is stopped, Windows will not generate or store events (attackers sometimes do this to cover their tracks).

---

## 🛠️ Steps to Catch Event ID 1100 in Wazuh

### 1. Make Sure the Logs Are Collected

Add this to the Windows agent config (`ossec.conf`):

```xml
<localfile>
  <location>System</location>
  <log_format>eventchannel</log_format>
</localfile>

<localfile>
  <location>Security</location>
  <log_format>eventchannel</log_format>
</localfile>
```

The `System` log is where **Event ID 1100** usually shows up.

---

### 2. Create a Custom Rule

On the Wazuh manager, edit:

```
/var/ossec/etc/rules/local_rules.xml
```

Add:

```xml
<rule id="100200" level="12">
  <if_group>windows,</if_group>
  <field name="win.system.eventID">1100</field>
  <description>Windows Event Logging Service was stopped (Event ID 1100)</description>
  <group>eventlog_shutdown,windows,</group>
</rule>
```

* **level="12"** makes it a high-severity alert.
* You can add an `<options>alert_by_email</options>` if you have email alerts configured.

---

### 3. Restart Wazuh Manager

After saving the rule, restart:

```bash
sudo systemctl restart wazuh-manager
```

---

### 4. Test the Rule

* Simulate or check a real 1100 event in Windows:

  * Stop the *Windows Event Log* service (`wevtsvc`) — though Windows usually protects this service.
  * Or search for past 1100 events in Event Viewer → **System log**.

If the agent sends the event, you should see a Wazuh alert like:

```
Windows Event Logging Service was stopped (Event ID 1100)
```

---

👉 Question:
Do you want me to also show you how to **auto-respond** (e.g., restart the service if 1100 is detected) using **Wazuh active responses**?
