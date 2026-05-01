# Cisco ACI Integration with Ansible Event-Driven Automation (EDA)

**Date:** 2026-05-01  
**Platform:** Ansible Automation Platform (AAP) with Event-Driven Ansible (EDA)  
**ACI Version:** 5.x+ (WebSocket support; webhook available on earlier versions)

---

## Overview

Cisco ACI (Application Centric Infrastructure) can be connected to Ansible Event-Driven Automation to enable automated responses to network events — faults, configuration changes, policy violations, and more.

There are three integration patterns. Choose based on your latency requirements and operational complexity tolerance.

| Approach | Latency | Complexity | Best For |
|---|---|---|---|
| ACI Webhook → EDA | Low | Low | Fault alerts, audit events |
| WebSocket Subscription | Very low | Medium | Real-time config drift detection |
| REST API Polling | Medium | Low | Periodic compliance checks |

---

## Prerequisites

- Ansible Automation Platform 2.4+ with Event-Driven Ansible controller
- Cisco APIC access with an API service account
- `cisco.aci` collection installed in your Decision Environment
- Network access from EDA controller to APIC (port 443; port 80 for WebSocket if not using TLS)

---

## Option 1: ACI Webhook → EDA Webhook Source

The simplest integration. ACI POSTs HTTP notifications to EDA when faults or audit log events occur.

### Step 1: Create an EDA Webhook Rulebook

Save this as `rulebooks/aci_webhook.yml` in your SCM project:

```yaml
---
- name: Receive ACI webhook events
  hosts: all
  sources:
    - ansible.eda.webhook:
        host: 0.0.0.0
        port: 5000
  rules:
    - name: ACI fault detected
      condition: event.payload is defined
      action:
        run_job_template:
          name: "ACI Fault Remediation"
          organization: "Default"
          job_args:
            extra_vars:
              aci_event_payload: "{{ event.payload }}"

    - name: ACI audit log - unauthorized change
      condition: >
        event.payload.imdata[0].aaaModLR is defined and
        event.payload.imdata[0].aaaModLR.attributes.descr is defined
      action:
        run_job_template:
          name: "ACI Compliance Check"
          organization: "Default"
```

### Step 2: Configure ACI to Send Webhooks

**Via ACI REST API:**

```bash
# Authenticate first and get a token
curl -k -X POST https://<apic>/api/aaaLogin.json \
  -H "Content-Type: application/json" \
  -d '{"aaaUser": {"attributes": {"name": "admin", "pwd": "yourpassword"}}}'

# Create an HTTP callhome destination
curl -k -X POST https://<apic>/api/mo/uni/fabric/moncommon/callhome.json \
  -H "Cookie: APIC-cookie=<token>" \
  -H "Content-Type: application/json" \
  -d '{
    "callhomeSmtp": {
      "attributes": {
        "host": "<eda-controller-ip>",
        "port": "5000",
        "format": "json"
      }
    }
  }'
```

**Via APIC GUI:**

1. Navigate to `Fabric → Fabric Policies → Pod Policies → Monitoring`
2. Select your monitoring policy
3. Add a **Callhome** destination pointing to your EDA controller IP and port 5000

### Step 3: Open Firewall Port

Ensure TCP port 5000 (or your chosen port) is open from APIC to your EDA controller.

---

## Option 2: ACI WebSocket Subscription → Custom EDA Source Plugin

ACI's REST API supports real-time WebSocket subscriptions. You subscribe to a class of ACI objects and receive push notifications immediately when they change. This is the best approach for sub-second reaction time.

### Step 1: Create the EDA Source Plugin

Create the following directory structure in your Ansible collection or project:

```
my_org/
└── aci_eda/
    └── plugins/
        └── event_source/
            └── aci_websocket.py
```

Save this as `plugins/event_source/aci_websocket.py`:

```python
"""
aci_websocket.py - EDA source plugin for Cisco ACI WebSocket subscriptions

Arguments:
    apic_host:    APIC hostname or IP (required)
    username:     APIC username (required)
    password:     APIC password (required)
    query_filter: ACI object class to subscribe to (default: faultRecord)
    verify_ssl:   Verify SSL certificate (default: false)
"""

import asyncio
import json
import aiohttp
import ssl


async def main(queue, args):
    apic = args["apic_host"]
    username = args["username"]
    password = args["password"]
    query_filter = args.get("query_filter", "faultRecord")
    verify_ssl = args.get("verify_ssl", False)

    ssl_ctx = ssl.create_default_context()
    if not verify_ssl:
        ssl_ctx.check_hostname = False
        ssl_ctx.verify_mode = ssl.CERT_NONE

    async with aiohttp.ClientSession() as session:
        # Authenticate to APIC
        login_data = {
            "aaaUser": {
                "attributes": {"name": username, "pwd": password}
            }
        }
        async with session.post(
            f"https://{apic}/api/aaaLogin.json",
            json=login_data,
            ssl=ssl_ctx
        ) as resp:
            data = await resp.json()
            token = data["imdata"][0]["aaaLogin"]["attributes"]["token"]

        # Subscribe to ACI object class
        async with session.get(
            f"https://{apic}/api/class/{query_filter}.json?subscription=yes",
            headers={"APIC-cookie": token},
            ssl=ssl_ctx
        ) as resp:
            result = await resp.json()
            subscription_id = result.get("subscriptionId")

        # Stream events over WebSocket
        async with session.ws_connect(
            f"wss://{apic}/socket{token}",
            ssl=ssl_ctx
        ) as ws:
            async for msg in ws:
                if msg.type == aiohttp.WSMsgType.TEXT:
                    event_data = json.loads(msg.data)
                    await queue.put({
                        "aci_event": event_data,
                        "subscription_id": subscription_id,
                        "source": f"aci_websocket:{apic}"
                    })
                elif msg.type == aiohttp.WSMsgType.ERROR:
                    break


if __name__ == "__main__":
    asyncio.run(main())
```

### Step 2: Create the Rulebook

Save as `rulebooks/aci_realtime.yml`:

```yaml
---
- name: ACI real-time event stream
  hosts: all
  sources:
    - my_org.aci_eda.aci_websocket:
        apic_host: apic.example.com
        username: ansible-svc
        password: "{{ lookup('env', 'ACI_PASSWORD') }}"
        query_filter: faultRecord
        verify_ssl: false

  rules:
    - name: Critical fault - trigger remediation
      condition: >
        event.aci_event.imdata[0].faultRecord.attributes.severity == "critical"
      action:
        run_job_template:
          name: "ACI Fault Remediation"
          organization: "Default"
          job_args:
            extra_vars:
              fault_dn: "{{ event.aci_event.imdata[0].faultRecord.attributes.dn }}"
              fault_code: "{{ event.aci_event.imdata[0].faultRecord.attributes.code }}"
              fault_description: "{{ event.aci_event.imdata[0].faultRecord.attributes.descr }}"

    - name: Tenant config changed - run compliance check
      condition: event.aci_event.imdata[0].fvTenant is defined
      action:
        run_job_template:
          name: "ACI Compliance Check"
          organization: "Default"
          job_args:
            extra_vars:
              tenant_dn: "{{ event.aci_event.imdata[0].fvTenant.attributes.dn }}"

    - name: EPG configuration changed
      condition: event.aci_event.imdata[0].fvAEPg is defined
      action:
        run_job_template:
          name: "ACI EPG Validation"
          organization: "Default"
```

### Common ACI Object Classes to Subscribe To

| Object Class | Description |
|---|---|
| `faultRecord` | All faults (critical, major, minor, warning) |
| `fvTenant` | Tenant create/modify/delete |
| `fvAEPg` | Endpoint Group changes |
| `fvBD` | Bridge Domain changes |
| `l3extOut` | L3 external network changes |
| `vzBrCP` | Contract changes |
| `aaaModLR` | Audit log (all user-initiated changes) |
| `fabricNode` | Fabric node status changes (leaf/spine) |

---

## Option 3: REST API Polling

The lowest-complexity option. EDA polls the ACI REST API on a schedule and fires rules when new data is found. Good for compliance checks that don't require immediate response.

### Rulebook with URL Check Source

Save as `rulebooks/aci_poll.yml`:

```yaml
---
- name: Poll ACI REST API for faults
  hosts: all
  sources:
    - ansible.eda.url_check:
        urls:
          - url: "https://apic.example.com/api/class/faultRecord.json?query-target-filter=and(eq(faultRecord.severity,\"critical\"))"
            headers:
              Cookie: "APIC-cookie={{ lookup('env', 'APIC_TOKEN') }}"
        delay: 60
  rules:
    - name: Critical fault found
      condition: event.status == "up" and event.body.imdata | length > 0
      action:
        run_job_template:
          name: "ACI Fault Remediation"
          organization: "Default"
```

> **Note:** For production polling, manage the APIC session token refresh in a separate job template or pre-flight task, as APIC tokens expire after 300 seconds by default.

---

## AAP / EDA Controller Configuration

### Step 1: Build a Custom Decision Environment

Create a `Containerfile` for your Decision Environment:

```dockerfile
FROM registry.redhat.io/ansible-automation-platform/de-minimal-rhel8:latest

RUN ansible-galaxy collection install cisco.aci
RUN pip3 install aiohttp
```

Build and push to your private automation hub or registry:

```bash
ansible-builder build -t my-registry.example.com/aci-eda-de:latest
podman push my-registry.example.com/aci-eda-de:latest
```

### Step 2: Create a Custom Credential Type for APIC

In AAP: `Credentials → Credential Types → Add`

**Input configuration:**
```yaml
fields:
  - id: apic_host
    type: string
    label: APIC Host
  - id: apic_username
    type: string
    label: Username
  - id: apic_password
    type: string
    label: Password
    secret: true
required:
  - apic_host
  - apic_username
  - apic_password
```

**Injector configuration:**
```yaml
env:
  ACI_HOST: "{{ apic_host }}"
  ACI_USERNAME: "{{ apic_username }}"
  ACI_PASSWORD: "{{ apic_password }}"
```

### Step 3: Create a Rulebook Activation

In the EDA Controller:

1. Navigate to `Rulebook Activations → Create rulebook activation`
2. Fill in:
   - **Name:** `Cisco ACI Event Stream`
   - **Project:** your SCM project containing the rulebooks
   - **Rulebook:** `rulebooks/aci_realtime.yml` (or whichever you chose)
   - **Decision Environment:** your custom ACI EDA DE
   - **Credential:** your APIC credential
   - **Restart policy:** `Always` (keeps the WebSocket alive)

### Step 4: Link EDA to AAP Controller

In the EDA Controller settings, ensure the AAP Controller connection is configured:

`Settings → Ansible Automation Controller`
- URL: `https://aap-controller.example.com`
- Token: (create an AAP token under `Users → Tokens`)

This allows EDA rules using `run_job_template` to trigger jobs on the AAP controller.

---

## Example Remediation Playbook

Save as `playbooks/aci_fault_remediate.yml` in your SCM project:

```yaml
---
- name: Remediate ACI fault
  hosts: localhost
  gather_facts: false
  vars:
    fault_dn: "{{ fault_dn | default('') }}"
    fault_code: "{{ fault_code | default('') }}"

  tasks:
    - name: Get fault details from ACI
      cisco.aci.aci_rest:
        host: "{{ lookup('env', 'ACI_HOST') }}"
        username: "{{ lookup('env', 'ACI_USERNAME') }}"
        password: "{{ lookup('env', 'ACI_PASSWORD') }}"
        validate_certs: false
        method: get
        path: "/api/mo/{{ fault_dn }}.json"
      register: fault_detail

    - name: Log fault to ticketing system
      ansible.builtin.uri:
        url: "https://servicenow.example.com/api/now/table/incident"
        method: POST
        headers:
          Authorization: "Bearer {{ lookup('env', 'SNOW_TOKEN') }}"
        body_format: json
        body:
          short_description: "ACI Fault: {{ fault_code }} - {{ fault_dn }}"
          description: "{{ fault_detail | to_nice_json }}"
          category: "network"
          urgency: "1"
      when: fault_code != ""

    - name: Notify team via messaging
      ansible.builtin.uri:
        url: "{{ lookup('env', 'SLACK_WEBHOOK_URL') }}"
        method: POST
        body_format: json
        body:
          text: ":alert: ACI Critical Fault Detected\n*DN:* {{ fault_dn }}\n*Code:* {{ fault_code }}"
```

---

## Troubleshooting

| Symptom | Likely Cause | Resolution |
|---|---|---|
| WebSocket disconnects frequently | APIC session token expiring (default 300s) | Set `aaaRefreshTicker` on APIC or refresh token in source plugin |
| No events received | ACI subscription not created | Check APIC logs under `Operations → Event Analytics` |
| EDA rulebook won't activate | Missing `aiohttp` in DE | Rebuild DE with `pip3 install aiohttp` |
| SSL errors | Self-signed cert on APIC | Set `verify_ssl: false` or import APIC CA cert into DE |
| `run_job_template` fails | EDA not linked to AAP controller | Configure controller connection in EDA settings |

---

## Security Recommendations

- Use a dedicated APIC service account with read-only access for EDA
- Store APIC credentials in AAP Credential Vault, never in plaintext rulebooks
- Enable SSL verification in production (`verify_ssl: true` with proper CA bundle)
- Rotate the APIC service account password on a schedule
- Restrict EDA controller network access to APIC management network only

---

## References

- [Cisco APIC REST API Guide](https://www.cisco.com/c/en/us/td/docs/switches/datacenter/aci/apic/sw/all/apic-rest-api-configuration-guide.html)
- [Event-Driven Ansible Documentation](https://access.redhat.com/documentation/en-us/red_hat_ansible_automation_platform/2.4/html/event-driven_ansible_controller_user_guide)
- [cisco.aci Ansible Collection](https://galaxy.ansible.com/ui/repo/published/cisco/aci/)
- [Ansible EDA Source Plugins Guide](https://ansible.readthedocs.io/projects/rulebook/en/latest/sources.html)
