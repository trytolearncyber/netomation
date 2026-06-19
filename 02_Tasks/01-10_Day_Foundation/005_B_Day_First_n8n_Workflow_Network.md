📄 05_B_Day_First_Network_n8n_Workflow.md

🎯 Goal
Network device (Cisco + MikroTik + Fortinet) poll করে unified Google Sheets dashboard তৈরি করা

📝 Task 1: Manual Trigger → Cisco RESTCONF → Google Sheets
Cisco RESTCONF কী? (Quick Reminder)

RESTCONF = Cisco device এর REST API
HTTP দিয়ে router/switch এর data নেওয়া যায়
Normal show command এর বদলে API call করা হয়

Step 1: ContainerLab এ Cisco device চালু করো
bash# ContainerLab topology file (cisco-lab.yml)
name: cisco-lab
topology:
  nodes:
    router1:
      kind: cisco_iosv
      image: vios-adventerprisek9-m.vmdk.SPA.157-3.M3
bash# Terminal এ run করো
sudo containerlab deploy -t cisco-lab.yml
Step 2: Cisco এ RESTCONF enable করো
Router> enable
Router# conf t
Router(config)# restconf
Router(config)# ip http server
Router(config)# ip http secure-server
Router(config)# ip http authentication local
Router(config)# username admin privilege 15 secret admin123
Step 3: n8n এ Manual Trigger → HTTP Request node
HTTP Request node settings:
Method: GET
URL: https://192.168.1.1/restconf/data/ietf-interfaces:interfaces
Authentication: Basic Auth
Username: admin
Password: admin123
Headers:
  Accept: application/yang-data+json
⚠️ SSL issue fix:
SSL Certificate Verification: OFF (lab environment এ)
👀 Expected Output:
json{
  "ietf-interfaces:interfaces": {
    "interface": [
      {
        "name": "GigabitEthernet0/0",
        "type": "iana-if-type:ethernetCsmacd",
        "enabled": true
      }
    ]
  }
}
Step 4: Google Sheets node
Operation: Append Row
Columns:
  Device: Cisco-Router1
  Interface: {{ $json["ietf-interfaces:interfaces"]["interface"][0]["name"] }}
  Status: {{ $json["ietf-interfaces:interfaces"]["interface"][0]["enabled"] }}
  Time: {{ new Date().toISOString() }}

📝 Task 2: Schedule Trigger → MikroTik API → Bandwidth Check
MikroTik REST API (Quick Reminder)

MikroTik RouterOS 7+ এ built-in REST API আছে
Port: 80 বা 443
Endpoint: /rest/interface

Step 1: MikroTik এ API enable করো
/ip service set www enabled=yes port=80
/user add name=apiuser password=api123 group=read
Step 2: n8n Schedule Trigger (Every 5 min) → HTTP Request
HTTP Request node settings:
Method: GET
URL: http://192.168.1.2/rest/interface
Authentication: Basic Auth
Username: apiuser
Password: api123
👀 Expected Output:
json[
  {
    "name": "ether1",
    "type": "ether",
    "running": true,
    "rx-byte": "1234567",
    "tx-byte": "9876543"
  }
]
Step 3: IF node যোগ করো (Bandwidth Alert Logic)
Condition:
  Value 1: {{ $json["rx-byte"] }}
  Operation: Greater Than
  Value 2: 1000000000  (1GB = bandwidth high)

True branch → Telegram alert পাঠাও
False branch → Google Sheets এ log করো


📝 Task 3: Webhook Trigger → Syslog Receive → Parse → Save
Syslog Webhook কীভাবে কাজ করে
Cisco Device → Syslog UDP 514 → n8n Webhook → Parse → Google Sheets
Step 1: n8n এ Webhook node যোগ করো
HTTP Method: POST
Path: /syslog
Response Mode: Last Node
Step 2: ngrok দিয়ে public URL বানাও
bashngrok http 5678
# Result: https://abc123.ngrok.io → তোমার n8n
Step 3: Cisco এ syslog config
Router(config)# logging host 192.168.1.100  (n8n server IP)
Router(config)# logging trap informational
Router(config)# logging on
Step 4: Code node → Syslog Parse করো
javascript// Syslog message parse করা
const message = $input.first().json.body;
const timestamp = new Date().toISOString();

return [{
  json: {
    raw: message,
    timestamp: timestamp,
    severity: message.includes("DOWN") ? "CRITICAL" : "INFO",
    device: "Cisco-Router1"
  }
}];
👀 Expected Output:
json{
  "raw": "%LINK-3-UPDOWN: Interface Gi0/0, changed state to down",
  "timestamp": "2026-06-17T10:00:00Z",
  "severity": "CRITICAL",
  "device": "Cisco-Router1"
}

📝 Task 4 (Main Project): Multi-Device Poller → Unified Dashboard
Workflow Architecture
Schedule Trigger (5 min)
        ↓
   ┌────┴────┐
   ↓         ↓
Cisco      MikroTik
RESTCONF   REST API
   ↓         ↓
   └────┬────┘
        ↓
  Merge node
        ↓
  Code node (normalize)
        ↓
  Google Sheets (unified dashboard)
Step 1: Schedule Trigger → 3টা আলাদা HTTP Request node
Cisco HTTP Request:
URL: https://192.168.1.1/restconf/data/ietf-interfaces:interfaces
Auth: Basic (admin/admin123)
Header: Accept: application/yang-data+json
MikroTik HTTP Request:
URL: http://192.168.1.2/rest/interface
Auth: Basic (apiuser/api123)
Fortinet HTTP Request:
URL: https://192.168.1.3/api/v2/monitor/system/interface
Header: Authorization: Bearer YOUR_API_TOKEN
Step 2: Merge node যোগ করো
Mode: Merge By Position
Step 3: Code node → Normalize করো
javascript// সব vendor এর data একই format এ আনো
const items = $input.all();

return items.map(item => {
  return {
    json: {
      vendor: item.json.vendor || "Unknown",
      device_ip: item.json.ip || "N/A",
      interface_count: item.json.interfaces?.length || 0,
      timestamp: new Date().toISOString(),
      status: "OK"
    }
  };
});
Step 4: Google Sheets → Unified Dashboard
VendorDevice IPInterfacesStatusTimeCisco192.168.1.14OK2026-06-17...MikroTik192.168.1.26OK2026-06-17...Fortinet192.168.1.33OK2026-06-17...

🔧 Practice
Fortinet এর পরিবর্তে আরেকটি MikroTik device যোগ করো এবং দুটো MikroTik এর interface count compare করো

⚠️ Common Mistakes

Cisco RESTCONF এ Accept: application/yang-data+json header না দিলে 406 error আসবে
MikroTik REST API শুধু RouterOS 7+ এ available — পুরোনো version এ কাজ করবে না
ngrok free plan এ session 2 ঘণ্টা পর expire হয় — lab এ মাথায় রাখো
Merge node তে সব branch এর data আসতে হবে, একটা fail করলে workflow আটকে যাবে


💡 Pro Tips

ContainerLab এ real hardware না থাকলে 192.168.1.x এর বদলে mock API (mockoon.com) ব্যবহার করতে পারো
Fortinet API token এভাবে নাও: System → Admin → REST API Admin → New
Google Sheets এ প্রতিটি vendor এর জন্য আলাদা Sheet tab রাখো — data clean থাকবে


📁 Log File: 05_B_Day_Log.md
markdown# Day 05B Log
Date: 2026-06-17

## Checklist
- [ ] Cisco RESTCONF থেকে interface data pull করা
- [ ] MikroTik REST API থেকে bandwidth data নেওয়া
- [ ] Webhook দিয়ে syslog receive করা
- [ ] Multi-device poller unified dashboard তৈরি করা

## Summary (One Line)
Cisco, MikroTik, Fortinet তিনটি device poll করে n8n দিয়ে unified Google Sheets dashboard তৈরি করা।

🎯 Goal: Poll network devices and display data in dashboardYou said: ContainerLab not always gns3



📄 05_B_Day_First_Network_n8n_Workflow.md (Mockoon Version)

🎯 Goal
Mockoon দিয়ে fake Cisco + MikroTik + Fortinet API বানিয়ে n8n workflow তৈরি করা — real device ছাড়াই

📝 Task 1: Mockoon Setup (15 মিনিট)
Step 1: Mockoon install করো
https://mockoon.com/download → Windows installer download → Install
Step 2: Mockoon ওপেন করো → New Environment
Environment Name: Network-Lab
Port: 3001
Step 3: তিনটা Route বানাও

Route 1 — Cisco RESTCONF Mock
Method: GET
Endpoint: /restconf/data/ietf-interfaces:interfaces
Status: 200
Response Body:
json{
  "ietf-interfaces:interfaces": {
    "interface": [
      {
        "name": "GigabitEthernet0/0",
        "type": "ethernetCsmacd",
        "enabled": true,
        "oper-status": "up"
      },
      {
        "name": "GigabitEthernet0/1",
        "type": "ethernetCsmacd",
        "enabled": false,
        "oper-status": "down"
      }
    ]
  }
}

Route 2 — MikroTik REST API Mock
Method: GET
Endpoint: /rest/interface
Status: 200
Response Body:
json[
  {
    "name": "ether1",
    "type": "ether",
    "running": true,
    "rx-byte": "524288000",
    "tx-byte": "104857600"
  },
  {
    "name": "ether2",
    "type": "ether",
    "running": false,
    "rx-byte": "0",
    "tx-byte": "0"
  }
]

Route 3 — Fortinet API Mock
Method: GET
Endpoint: /api/v2/monitor/system/interface
Status: 200
Response Body:
json{
  "results": [
    {
      "name": "port1",
      "link": true,
      "speed": 1000,
      "rx_bytes": 209715200,
      "tx_bytes": 52428800
    },
    {
      "name": "port2",
      "link": false,
      "speed": 0,
      "rx_bytes": 0,
      "tx_bytes": 0
    }
  ]
}
Step 4: Mockoon এ ▶️ Start Server button click করো
✅ Verify:
Browser এ যাও:
http://localhost:3001/rest/interface
→ MikroTik JSON দেখাবে

📝 Task 2: n8n এ Manual Trigger → Cisco Mock API
Step 1: n8n → New Workflow → Manual Trigger
Step 2: HTTP Request node
Method: GET
URL: http://localhost:3001/restconf/data/ietf-interfaces:interfaces
⚠️ n8n Docker এ চললে localhost কাজ করবে না।

এই URL ব্যবহার করো:
http://host.docker.internal:3001/restconf/data/ietf-interfaces:interfaces
Step 3: Execute করো
👀 Expected Output:
json{
  "ietf-interfaces:interfaces": {
    "interface": [
      { "name": "GigabitEthernet0/0", "enabled": true },
      { "name": "GigabitEthernet0/1", "enabled": false }
    ]
  }
}

📝 Task 3: Schedule Trigger → MikroTik Mock → IF node → Alert
Step 1: Schedule Trigger (Every 5 min)
Step 2: HTTP Request node
URL: http://host.docker.internal:3001/rest/interface
Step 3: Code node → rx-byte MB তে convert করো
javascriptconst interfaces = $input.first().json;

return interfaces.map(iface => {
  return {
    json: {
      name: iface.name,
      running: iface.running,
      rx_mb: (parseInt(iface["rx-byte"]) / 1024 / 1024).toFixed(2),
      tx_mb: (parseInt(iface["tx-byte"]) / 1024 / 1024).toFixed(2)
    }
  };
});
Step 4: IF node
Condition:
  Value 1: {{ $json.rx_mb }}
  Operation: Greater Than
  Value 2: 400

True → "High Bandwidth" label দাও
False → "Normal" label দাও

👀 Expected Output:
ether1 → rx_mb: 500.00 → HIGH BANDWIDTH ⚠️
ether2 → rx_mb: 0.00   → NORMAL ✅

📝 Task 4 (Main Project): Multi-Device Poller → Unified Google Sheets Dashboard
Workflow Architecture
Schedule Trigger (5 min)
         ↓
  ┌──────┼──────┐
  ↓      ↓      ↓
Cisco  MikroTik  Fortinet
Mock   Mock      Mock
  ↓      ↓      ↓
  └──────┼──────┘
         ↓
    Code node
  (normalize data)
         ↓
   Google Sheets
  (unified dashboard)
Step 1: তিনটা HTTP Request node
Cisco:
URL: http://host.docker.internal:3001/restconf/data/ietf-interfaces:interfaces
Name: Cisco-Router1
MikroTik:
URL: http://host.docker.internal:3001/rest/interface
Name: MikroTik-Core
Fortinet:
URL: http://host.docker.internal:3001/api/v2/monitor/system/interface
Name: Fortinet-FW1
Step 2: তিনটা আলাদা Code node → Normalize
Cisco Normalize:
javascriptconst data = $input.first().json;
const interfaces = data["ietf-interfaces:interfaces"]["interface"];

return interfaces.map(iface => ({
  json: {
    vendor: "Cisco",
    device: "Router1",
    interface: iface.name,
    status: iface.enabled ? "UP" : "DOWN",
    timestamp: new Date().toISOString()
  }
}));
MikroTik Normalize:
javascriptconst interfaces = $input.all();

return interfaces.map(item => ({
  json: {
    vendor: "MikroTik",
    device: "Core-SW",
    interface: item.json.name,
    status: item.json.running ? "UP" : "DOWN",
    timestamp: new Date().toISOString()
  }
}));
Fortinet Normalize:
javascriptconst data = $input.first().json;
const interfaces = data.results;

return interfaces.map(iface => ({
  json: {
    vendor: "Fortinet",
    device: "FW1",
    interface: iface.name,
    status: iface.link ? "UP" : "DOWN",
    timestamp: new Date().toISOString()
  }
}));
Step 3: Merge node
Mode: Append
Step 4: Google Sheets node
Operation: Append Row
Columns:
  Vendor:    {{ $json.vendor }}
  Device:    {{ $json.device }}
  Interface: {{ $json.interface }}
  Status:    {{ $json.status }}
  Time:      {{ $json.timestamp }}
👀 Google Sheets Expected Output:
VendorDeviceInterfaceStatusTimeCiscoRouter1GigabitEthernet0/0UP2026-06-17...CiscoRouter1GigabitEthernet0/1DOWN2026-06-17...MikroTikCore-SWether1UP2026-06-17...MikroTikCore-SWether2DOWN2026-06-17...FortinetFW1port1UP2026-06-17...FortinetFW1port2DOWN2026-06-17...

🔧 Practice
Mockoon এ ether1 এর "running": true → false করো → n8n workflow আবার run করো → Sheets এ status change হয় কিনা দেখো

⚠️ Common Mistakes

n8n Docker এ চললে localhost কাজ করে না → host.docker.internal ব্যবহার করো
Mockoon server Start না করলে n8n connection refused error দেবে
Merge node এ সব branch connect না করলে incomplete data আসবে
Code node এ $input.first().json vs $input.all() — array response এ all() ব্যবহার করো

💡 Pro Tips

Mockoon এ "Latency" add করো (200-500ms) → real network device behavior simulate হবে
Mockoon export করা যায় JSON হিসেবে → GitHub এ save রাখো → যেকোনো PC তে import করো
Real device পেলে শুধু URL change করলেই workflow কাজ করবে — বাকি সব same থাকবে


📁 Log File: 05_B_Day_Log.md
markdown# Day 05B Log
Date: 2026-06-17

## Checklist
- [ ] Mockoon install ও 3টা route তৈরি করা
- [ ] n8n থেকে Cisco mock API call করা
- [ ] MikroTik mock → IF node → bandwidth alert
- [ ] Multi-device poller → Google Sheets dashboard

## Summary (One Line)
Mockoon দিয়ে Cisco/MikroTik/Fortinet API mock করে n8n unified dashboard তৈরি করা।

🎯 Goal: Poll network devices and display data in dashboard