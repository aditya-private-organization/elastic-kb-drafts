## Publishing Metadata (Innovation Hub — support.elastic.dev/knowledge/create)

| Field | Value |
|---|---|
| **Type** | Break-Fix |
| **Solution** | Observability |
| **Platform** | Self-managed / On-premise |
| **Deployment Type** | Elastic self-managed |
| **Deployment Versions** | _(leave blank — not version-gated)_ |
| **Product** | Logstash |
| **Product Versions** | 9.x (confirmed 9.4) |
| **Component** | SNMP input (logstash-integration-snmp) |

---

**Type:** Break-Fix

# HPE iLO SNMP sensor temperatures incorrect in Logstash

**Summary:** When Logstash polls HPE iLO via SNMP OID `cpqHeTemperatureCelsius`, sensors that have expandable sub-entries in the iLO WebUI return a fixed placeholder value instead of the actual measured temperature. Switching to the HPE Redfish API resolves this.

---

## Problem

When polling HPE server temperature sensors via Logstash using the SNMP input plugin, some sensors consistently report an incorrect or fixed temperature value (commonly `40°C`) in Elasticsearch, while the HPE iLO WebUI shows the correct actual temperature.

This affects sensors that have an expandable `>` dropdown in the iLO WebUI's **Thermal and Cooling** table — for example, XLR8R GPU accelerator sensors. Sensors without sub-entries report correctly through SNMP.

Confirming with `snmpwalk` directly against the iLO host returns the same incorrect value, ruling out Logstash as the source of the mismatch.

**Affected OID:** `.1.3.6.1.4.1.232.6.2.6.8.1.4` (`cpqHeTemperatureCelsius`, `cpqHeTemperatureTable` in the HP Compaq Health MIB)

---

## Environment

- **Product**: Logstash (SNMP input plugin)
- **Version**: 9.x (observed on 9.4; likely affects earlier versions)
- **Hardware**: HPE ProLiant Gen12 servers with HPE iLO 7
- **Protocol**: SNMPv3 (authPriv, SHA/AES)
- **Deployment**: Self-managed Logstash polling HPE iLO SNMP endpoint

---

## Cause

Logstash is working correctly — it ingests exactly what HPE's SNMP subagent reports.

The root cause is how HPE structures composite sensors in `cpqHeTemperatureTable`. For sensors that have sub-entries (shown with a `>` expandable dropdown in the iLO WebUI), the **parent row** in the table does not carry the actual measured temperature. Instead, it returns a fixed or placeholder value. The real per-component readings are stored in child index entries that are not exposed through a simple walk of the `cpqHeTemperatureCelsius` column alone.

This is a characteristic of the HPE SNMP MIB design for composite sensors such as GPU accelerators (XLR8R), certain PCI slots, and other multi-component devices. The `>` indicator in the iLO WebUI is a reliable signal: any sensor row that displays it will return a non-actual temperature at the parent SNMP level.

---

## Resolution

Switch from SNMP to the **HPE Redfish API** to collect temperature data. The Redfish endpoint returns structured JSON with one entry per sensor — including all sub-component readings — matching what the iLO WebUI displays. This bypasses the SNMP parent/child hierarchy entirely.

**Endpoint:** `GET /redfish/v1/Chassis/1/Thermal`

The response includes a `Temperatures` array. Use Logstash's `http_poller` input with the `split` filter to produce one event per sensor:

```
input {
  http_poller {
    urls => {
      ilo_thermal => {
        method => "get"
        url => "https://<ilo-ip>/redfish/v1/Chassis/1/Thermal"
        user => "<ilo-user>"
        password => "<ilo-password>"
        ssl_verification_mode => "none"   # or configure your CA cert
      }
    }
    schedule => { every => "60s" }
    codec => "json"
  }
}

filter {
  split { field => "Temperatures" }
}
```

The `split` filter produces one Logstash document per entry in the `Temperatures` array, giving a separate event for each sensor with its actual reading. This approach was confirmed to resolve the issue for an HPE ProLiant DL380a Gen12 running iLO 7.

**Note on SSL**: If you have a trusted CA certificate for iLO, configure `ssl_certificate_authorities` instead of disabling verification. See [Logstash HTTP Poller Input Certificate error](https://support.elastic.co/knowledge/0dafc0b3) if you encounter certificate errors.

---

## Workaround

If switching to Redfish is not immediately possible, perform a broader SNMP walk of the full `cpqHe` temperature subtree (`.1.3.6.1.4.1.232.6.2.6`) to locate the correct child OID columns for the sensors with sub-entries. Consult the HPE SNMP Management MIB Kit (available from the HPE Support Portal) for the authoritative table definitions. Note that Elastic does not provide OID lists for third-party hardware.

---

## References

- [HPE Redfish API reference — Thermal schema](https://support.hpe.com/hpesc/public/docDisplay?docId=sd00002007en_us&page=GUID-D7147C7F-2016-0901-06D0-0000000016AA.html&docLocale=en_US)
- [General Approach for Sending HPE iLO / Non-OS Management Logs to Elasticsearch](https://support.elastic.co/knowledge/939d1c0e)
- [Logstash HTTP Poller Input Certificate error](https://support.elastic.co/knowledge/0dafc0b3)

{elastic-private-context}
- Case: 02154981 — Home Team Science & Technology Agency (HTX), Logstash 9.4, HPE ProLiant DL380a Gen12 / iLO 7 firmware 1.24.00
- Customer confirmed fix working on 2026-09-30: "Thank you for the help I am able to retrieve what I want to now."
- snmpwalk output confirmed indices 48, 50, 52, 54, 56, 58, 60, 62 (XLR8R sensors) all return INTEGER: 40 from the SNMP device directly
- Related prior case: 02152022 — "Require assistance for MIB files for logstash SNMP" (Closed, same customer)
{/elastic-private-context}
