import os, re, requests
from flask import Flask, request, jsonify

# === CONFIG ===
ASANA_TOKEN = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpYXQiOjE3NTM0Nzg0NjMsInNjb3BlIjoicHJvamVjdF90ZW1wbGF0ZXM6cmVhZCBwcm9qZWN0czpyZWFkIHRhc2tzOnJlYWQgdGFza3M6d3JpdGUgdXNlcnM6cmVhZCB3ZWJob29rczpyZWFkIHdlYmhvb2tzOndyaXRlIHdvcmtzcGFjZXM6cmVhZCB3b3Jrc3BhY2VzLnR5cGVhaGVhZDpyZWFkIiwic3ViIjoxMjEwNTc0ODcwNDc0NTYyLCJyZWZyZXNoX3Rva2VuIjoxMjEwODkzNDkzNDQ1NDkzLCJ2ZXJzaW9uIjoyLCJhcHAiOjEyMTA4OTM0ODAwMzU0MTksImV4cCI6MTc1MzQ4MjA2M30.tZAJLaQZpDc0KOBTD91plAUDMPk5XF9omIzLrLyXUUA"
PROJECT_GID = "1206922077345173"
ASANA_API = "https://app.asana.com/api/1.0"
HEADERS = {"Authorization": f"Bearer {ASANA_TOKEN}"}

app = Flask(__name__)

def get_next_mkt_id():
    """Find the next sequential MKT ID by scanning existing tasks"""
    tasks = requests.get(f"{ASANA_API}/projects/{PROJECT_GID}/tasks", headers=HEADERS).json()['data']
    max_id = 0
    for t in tasks:
        match = re.search(r"MKT-(\d+)", t['name'])
        if match:
            max_id = max(max_id, int(match.group(1)))
    return max_id + 1

def rename_task(task_gid, task_name):
    """Rename new task to append - MKT-####"""
    next_id = get_next_mkt_id()
    new_name = f"{task_name} - MKT-{next_id:04d}"
    requests.put(f"{ASANA_API}/tasks/{task_gid}", headers=HEADERS, json={"data": {"name": new_name}})
    print(f"✅ Renamed: {new_name}")

@app.route("/webhook", methods=["POST"])
def webhook():
    # Handle Asana webhook verification handshake
    if "X-Hook-Secret" in request.headers:
        return "", 200, {"X-Hook-Secret": request.headers["X-Hook-Secret"]}

    payload = request.json
    for event in payload.get("events", []):
        if event["resource"]["resource_type"] == "task" and event["action"] == "added":
            task_gid = event["resource"]["gid"]
            task_data = requests.get(f"{ASANA_API}/tasks/{task_gid}", headers=HEADERS).json()["data"]
            rename_task(task_gid, task_data["name"])

    return jsonify({"status": "ok"})

@app.route("/")
def home():
    return "✅ Asana Webhook Listener is running!"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=int(os.environ.get("PORT", 5000)))
