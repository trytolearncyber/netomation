📄 05_A_Day_First_n8n_Workflow.md

🎯 Goal
n8n এ প্রথম working workflow তৈরি করা — Schedule Trigger → Weather API → Google Sheets

📝 Task 1: n8n Workflow কী এবং কীভাবে কাজ করে
Workflow এর ৩টি মূল অংশ:

Trigger → Workflow শুরু করে (Manual / Schedule / Webhook)
Process → Data নেওয়া, transform করা (HTTP Request / Code node)
Action → Data কোথাও পাঠানো বা save করা (Google Sheets / Email)
Real-life analogy:

Trigger = অ্যালার্ম বাজল
Process = কাজটা করা হলো
Action = Result কোথাও রাখা হলো
📝 Task 2: Manual Trigger → HTTP Request → Console Log
Step 1: n8n dashboard ওপেন করো → New Workflow click করো

Step 2: + বাটনে click করো → Manual Trigger node যোগ করো

Step 3: Manual Trigger এর পাশে + click করো → HTTP Request node যোগ করো

Step 4: HTTP Request node এ এই settings দাও:

Method: GET
URL: https://jsonplaceholder.typicode.com/posts/1
Step 5: Execute Workflow click করো

👀 Expected Output:

json
{
  "userId": 1,
  "id": 1,
  "title": "sunt aut facere repellat...",
  "body": "quia et suscipit..."
}
📝 Task 3: Schedule Trigger Setup (Every 5 Minutes)
Step 1: Manual Trigger node টি delete করো

Step 2: + → Schedule Trigger node যোগ করো

Step 3: Schedule Trigger settings:

Trigger Interval: Minutes
Minutes Between Triggers: 5
Step 4: HTTP Request node আগেরটাই থাকবে

Step 5: Workflow টি Save করো → Activate toggle ON করো

⚠️ Common Mistake: Activate না করলে schedule কাজ করবে না। Save করা আর Activate করা — দুটো আলাদা।

📝 Task 4 (Main Project): Schedule → Weather API → Google Sheets
Step 1: Weather API key নাও
openweathermap.org → Free account → API key copy করো
Step 2: Schedule Trigger
Every 5 minutes
Step 3: HTTP Request node (Weather API)
Method: GET
URL: https://api.openweathermap.org/data/2.5/weather
Query Parameters:
  q: Dhaka
  appid: YOUR_API_KEY
  units: metric
Step 4: Google Sheets node যোগ করো
Operation: Append Row
Sheet: তোমার Google Sheet এর নাম
Columns Mapping:
  City: {{ $json.name }}
  Temp: {{ $json.main.temp }}
  Weather: {{ $json.weather[0].description }}
  Time: {{ new Date().toISOString() }}
Step 5: Google Sheets Credential
n8n এ Google Sheets node → Credential → OAuth2 দিয়ে connect করো
Step 6: Save → Activate
👀 Expected Output (Google Sheets এ):

City	Temp	Weather	Time
Dhaka	31.5	clear sky	2026-06-17T...
🔧 Practice
q: Dhaka এর জায়গায় q: Helsinki দিয়ে test করো → দুটো শহরের data আলাদা row তে save হয় কিনা দেখো

⚠️ Common Mistakes
https:// বাদ দিলে HTTP Request fail করবে
Google Sheets credential সঠিকভাবে connect না করলে "Authentication Error" আসবে
API key invalid হলে Weather API 401 error দেবে
Workflow Activate না করলে Schedule কাজ করবে না
💡 Pro Tips
HTTP Request node এ "Include Headers in Output" enable রাখো — debug করতে সহজ হবে
Google Sheets এর পরিবর্তে প্রথমে n8n এর Debug Output দিয়ে data ঠিক আছে কিনা confirm করো, তারপর Sheets যোগ করো
📁 Log File: 05_A_Day_Log.md

markdown
# Day 05A Log
Date: 2026-06-17

## Checklist
- [ ] Manual Trigger → HTTP Request → Output দেখা
- [ ] Schedule Trigger (5 min) setup করা
- [ ] Weather API call করা
- [ ] Google Sheets এ data save করা

## Summary (One Line)
n8n এ প্রথম schedule-based workflow তৈরি করে Weather API data Google Sheets এ auto-save করা।

🎯 Goal: Build first working workflow in n8n
