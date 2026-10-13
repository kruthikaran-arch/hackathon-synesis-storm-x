# ⚡ Sales Pipeline Autopilot - Complete Source Code Archive

> **Complete standalone bundle of all source code files used to build the Sales Pipeline Autopilot.**

Project Path: `/Users/kruthikaran/.gemini/antigravity/scratch/sales-pipeline-autopilot`

---

## 📄 `app.py`
**Description**: Backend Flask Web Server & REST API  
**File Path**: `/Users/kruthikaran/.gemini/antigravity/scratch/sales-pipeline-autopilot/app.py`

```python
import json
import random
from datetime import datetime, timedelta
from flask import Flask, render_template, request, jsonify, session
from database import get_connection, init_db, seed_database, log_activity
from engine import SalesAutopilotEngine

app = Flask(__name__)
app.secret_key = "sales_autopilot_secret_key_demo"

init_db()
seed_database()

# Current user session tracker (default to Client or Sales Exec)
CURRENT_SESSION = {
    "user_id": 2 # Default to Sarah Jenkins (Client)
}

@app.route("/")
def index():
    return render_template("index.html")

# -------------------------------------------------------------
# AUTH & USER PROFILES
# -------------------------------------------------------------
@app.route("/api/auth/current-user", methods=["GET"])
def get_current_user():
    conn = get_connection()
    cursor = conn.cursor()

    user_id = session.get("user_id", CURRENT_SESSION["user_id"])
    cursor.execute("SELECT * FROM users WHERE id = ?", (user_id,))
    user = cursor.fetchone()

    if not user:
        cursor.execute("SELECT * FROM users LIMIT 1")
        user = cursor.fetchone()

    user_dict = dict(user)
    user_dict.pop("password", None)

    # Fetch assigned partner (Sales Exec or Client)
    partner = None
    if user_dict.get("assigned_partner_id"):
        cursor.execute("SELECT id, name, email, role, title, avatar FROM users WHERE id = ?", (user_dict["assigned_partner_id"],))
        p = cursor.fetchone()
        if p:
            partner = dict(p)
    elif user_dict["role"] == "sales_executive":
        # Find clients assigned to this sales exec
        cursor.execute("SELECT id, name, email, company_name, title, avatar FROM users WHERE assigned_partner_id = ?", (user_dict["id"],))
        p = cursor.fetchone()
        if p:
            partner = dict(p)

    user_dict["counterpart"] = partner
    conn.close()
    return jsonify(user_dict)

@app.route("/api/auth/login", methods=["POST"])
def login():
    data = request.json or {}
    email = data.get("email", "").strip().lower()
    password = data.get("password", "").strip()

    conn = get_connection()
    cursor = conn.cursor()

    cursor.execute("SELECT * FROM users WHERE LOWER(email) = ?", (email,))
    user = cursor.fetchone()

    if not user or (password and user["password"] != password):
        conn.close()
        return jsonify({"error": "Invalid email or password"}), 401

    CURRENT_SESSION["user_id"] = user["id"]
    session["user_id"] = user["id"]

    user_dict = dict(user)
    user_dict.pop("password", None)
    conn.close()
    return jsonify({"success": True, "user": user_dict})

@app.route("/api/auth/switch-role", methods=["POST"])
def switch_role():
    data = request.json or {}
    target_role = data.get("role", "client")

    conn = get_connection()
    cursor = conn.cursor()

    cursor.execute("SELECT * FROM users WHERE role = ? LIMIT 1", (target_role,))
    user = cursor.fetchone()

    if not user:
        conn.close()
        return jsonify({"error": "User for role not found"}), 404

    CURRENT_SESSION["user_id"] = user["id"]
    session["user_id"] = user["id"]

    user_dict = dict(user)
    user_dict.pop("password", None)
    conn.close()
    return jsonify({"success": True, "user": user_dict})

# -------------------------------------------------------------
# CLIENT REQUIREMENTS & SALES EXECUTIVE ASSIGNMENT
# -------------------------------------------------------------
@app.route("/api/client/requirements", methods=["GET"])
def get_client_requirements():
    conn = get_connection()
    cursor = conn.cursor()

    cursor.execute("""
        SELECT r.*, u.name as client_name, u.email as client_email
        FROM client_requirements r
        JOIN users u ON r.client_user_id = u.id
        ORDER BY r.id DESC LIMIT 1
    """)
    row = cursor.fetchone()
    conn.close()

    if not row:
        return jsonify({"error": "No requirements found"}), 404

    return jsonify(dict(row))

@app.route("/api/client/requirements", methods=["POST"])
def update_client_requirements():
    data = request.json or {}

    conn = get_connection()
    cursor = conn.cursor()

    now = datetime.now().strftime("%Y-%m-%d %H:%M:%S")

    # Fetch active requirement record
    cursor.execute("SELECT id FROM client_requirements ORDER BY id DESC LIMIT 1")
    req = cursor.fetchone()

    if req:
        cursor.execute("""
            UPDATE client_requirements
            SET company_name = ?,
                product_name = ?,
                value_proposition = ?,
                target_industries = ?,
                target_company_size = ?,
                target_geography = ?,
                key_buying_signals = ?,
                pain_points_solved = ?,
                pricing_tier = ?,
                notes_for_sales_exec = ?,
                updated_at = ?
            WHERE id = ?
        """, (
            data.get("company_name", "Apex Enterprise Systems"),
            data.get("product_name", "Enterprise Laptop Fleets & Cloud MDM"),
            data.get("value_proposition", ""),
            data.get("target_industries", ""),
            data.get("target_company_size", ""),
            data.get("target_geography", "North America & Remote EU"),
            data.get("key_buying_signals", ""),
            data.get("pain_points_solved", ""),
            data.get("pricing_tier", ""),
            data.get("notes_for_sales_exec", ""),
            now,
            req["id"]
        ))
    else:
        cursor.execute("""
            INSERT INTO client_requirements (
                client_user_id, company_name, product_name, value_proposition,
                target_industries, target_company_size, target_geography,
                key_buying_signals, pain_points_solved, pricing_tier,
                assigned_sales_exec_id, assigned_sales_exec_name, assigned_sales_exec_email,
                notes_for_sales_exec, updated_at
            ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
        """, (
            2,
            data.get("company_name", "Apex Enterprise Systems"),
            data.get("product_name", "Enterprise Laptop Fleets & Cloud MDM"),
            data.get("value_proposition", ""),
            data.get("target_industries", ""),
            data.get("target_company_size", ""),
            data.get("target_geography", "North America & Remote EU"),
            data.get("key_buying_signals", ""),
            data.get("pain_points_solved", ""),
            data.get("pricing_tier", ""),
            1, "Alex Morgan", "alex.morgan@salesautopilot.io",
            data.get("notes_for_sales_exec", ""),
            now
        ))

    # Also synchronize settings table so autopilot discovers/scores matching these criteria
    if data.get("value_proposition"):
        cursor.execute("INSERT OR REPLACE INTO settings (key, value) VALUES ('company_pitch', ?)", (data["value_proposition"],))
    if data.get("target_industries"):
        cursor.execute("INSERT OR REPLACE INTO settings (key, value) VALUES ('target_industries', ?)", (data["target_industries"],))
    if data.get("target_company_size"):
        cursor.execute("INSERT OR REPLACE INTO settings (key, value) VALUES ('target_icp_min_size', ?)", (data["target_company_size"],))

    log_activity(None, "icp_scored", "Client Strategy & Requirements Updated",
                 f"Client updated target profile. Industries: {data.get('target_industries')[:50]}... Autopilot synchronized.", conn=conn)

    conn.commit()
    conn.close()
    return jsonify({"success": True, "message": "Client requirements and sales strategy synchronized with Autopilot."})

@app.route("/api/sales-execs", methods=["GET"])
def get_sales_execs():
    conn = get_connection()
    cursor = conn.cursor()

    cursor.execute("SELECT id, name, email, company_name, title, avatar FROM users WHERE role = 'sales_executive'")
    execs = [dict(r) for r in cursor.fetchall()]
    conn.close()
    return jsonify(execs)

# -------------------------------------------------------------
# CORE DASHBOARD STATS
# -------------------------------------------------------------
@app.route("/api/stats", methods=["GET"])
def get_stats():
    conn = get_connection()
    cursor = conn.cursor()

    cursor.execute("SELECT COUNT(*) as total FROM leads")
    total_leads = cursor.fetchone()["total"]

    cursor.execute("SELECT COUNT(*) as high_icp FROM leads WHERE overall_score >= 80")
    high_icp = cursor.fetchone()["high_icp"]

    cursor.execute("SELECT COUNT(*) as booked FROM leads WHERE stage = 'meeting_booked'")
    meetings_booked = cursor.fetchone()["booked"]

    cursor.execute("SELECT COUNT(*) as replied FROM leads WHERE stage IN ('replied', 'meeting_booked', 'follow_up_scheduled')")
    replied_count = cursor.fetchone()["replied"]

    cursor.execute("SELECT COUNT(*) as contacted FROM leads WHERE stage IN ('in_sequence', 'replied', 'meeting_booked', 'follow_up_scheduled', 'opted_out')")
    contacted_count = cursor.fetchone()["contacted"]

    cursor.execute("SELECT COUNT(*) as pending_review FROM outreach_drafts WHERE status = 'pending_approval'")
    pending_review = cursor.fetchone()["pending_review"]

    cursor.execute("SELECT COUNT(*) as opted_out FROM leads WHERE is_opted_out = 1")
    opted_out_count = cursor.fetchone()["opted_out"]

    response_rate = round((replied_count / max(1, contacted_count)) * 100, 1)
    pipeline_val = (high_icp * 120000) + (meetings_booked * 85000)

    conn.close()
    return jsonify({
        "total_leads": total_leads,
        "high_icp_leads": high_icp,
        "meetings_booked": meetings_booked,
        "response_rate": response_rate,
        "pending_human_review": pending_review,
        "opted_out_safeguards": opted_out_count,
        "pipeline_value_formatted": f"${pipeline_val:,}"
    })

@app.route("/api/leads", methods=["GET"])
def get_leads():
    stage = request.args.get("stage")
    priority = request.args.get("priority")
    search = request.args.get("search", "").strip().lower()
    sort = request.args.get("sort", "score_desc")

    conn = get_connection()
    cursor = conn.cursor()

    query = "SELECT * FROM leads WHERE 1=1"
    params = []

    if stage and stage != "all":
        query += " AND stage = ?"
        params.append(stage)

    if priority and priority != "all":
        query += " AND priority = ?"
        params.append(priority)

    if search:
        query += " AND (LOWER(company_name) LIKE ? OR LOWER(contact_name) LIKE ? OR LOWER(industry) LIKE ? OR LOWER(tech_stack) LIKE ?)"
        term = f"%{search}%"
        params.extend([term, term, term, term])

    if sort == "score_desc":
        query += " ORDER BY overall_score DESC"
    elif sort == "score_asc":
        query += " ORDER BY overall_score ASC"
    elif sort == "name_asc":
        query += " ORDER BY company_name ASC"
    else:
        query += " ORDER BY updated_at DESC"

    cursor.execute(query, params)
    leads = [dict(row) for row in cursor.fetchall()]

    for l in leads:
        if l.get("scoring_breakdown"):
            try:
                l["scoring_breakdown"] = json.loads(l["scoring_breakdown"])
            except Exception:
                pass

    conn.close()
    return jsonify(leads)

@app.route("/api/leads/<int:lead_id>", methods=["GET"])
def get_lead_detail(lead_id):
    conn = get_connection()
    cursor = conn.cursor()

    cursor.execute("SELECT * FROM leads WHERE id = ?", (lead_id,))
    lead = cursor.fetchone()
    if not lead:
        conn.close()
        return jsonify({"error": "Lead not found"}), 404

    lead_data = dict(lead)
    if lead_data.get("scoring_breakdown"):
        try:
            lead_data["scoring_breakdown"] = json.loads(lead_data["scoring_breakdown"])
        except Exception:
            pass

    cursor.execute("SELECT * FROM outreach_drafts WHERE lead_id = ? ORDER BY id DESC", (lead_id,))
    drafts = [dict(r) for r in cursor.fetchall()]

    cursor.execute("SELECT * FROM replies WHERE lead_id = ? ORDER BY id DESC", (lead_id,))
    replies = [dict(r) for r in cursor.fetchall()]

    cursor.execute("SELECT * FROM activity_logs WHERE lead_id = ? ORDER BY id DESC", (lead_id,))
    activities = [dict(r) for r in cursor.fetchall()]

    conn.close()
    return jsonify({
        "lead": lead_data,
        "drafts": drafts,
        "replies": replies,
        "activities": activities
    })

@app.route("/api/leads", methods=["POST"])
def create_lead():
    data = request.json or {}
    company_name = data.get("company_name", "").strip()
    contact_name = data.get("contact_name", "").strip()
    email = data.get("email", "").strip()
    industry = data.get("industry", "Technology").strip()
    company_size = data.get("company_size", "50-100 employees").strip()
    tech_stack = data.get("tech_stack", "Cloud, Slack, Laptops").strip()
    buying_signals = data.get("buying_signals", "Hiring tech talent and evaluating infrastructure refresh").strip()

    if not company_name or not contact_name or not email:
        return jsonify({"error": "Company name, contact name, and email are required"}), 400

    conn = get_connection()
    cursor = conn.cursor()

    cursor.execute("SELECT id FROM leads WHERE email = ? OR company_name = ?", (email, company_name))
    existing = cursor.fetchone()
    if existing:
        conn.close()
        return jsonify({"error": f"Duplicate safeguard: Lead already exists with id #{existing['id']}"}), 409

    lead_sample = {
        "company_name": company_name,
        "contact_name": contact_name,
        "contact_title": data.get("contact_title", "Director of IT"),
        "company_size": company_size,
        "industry": industry,
        "tech_stack": tech_stack,
        "buying_signals": buying_signals,
        "estimated_budget": data.get("estimated_budget", "$50,000 - $100,000")
    }
    score_res = SalesAutopilotEngine.calculate_lead_score(lead_sample)
    now = datetime.now().strftime("%Y-%m-%d %H:%M:%S")

    cursor.execute("""
        INSERT INTO leads (
            company_name, contact_name, contact_title, email, industry, company_size,
            estimated_budget, tech_stack, buying_signals, icp_score, urgency_score,
            overall_score, scoring_breakdown, stage, priority, created_at, updated_at
        ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, 'scored', ?, ?, ?)
    """, (
        company_name, contact_name, lead_sample["contact_title"], email, industry, company_size,
        lead_sample["estimated_budget"], tech_stack, buying_signals,
        score_res["icp_score"], score_res["urgency_score"], score_res["overall_score"],
        json.dumps(score_res["scoring_breakdown"]), score_res["priority"], now, now
    ))
    new_id = cursor.lastrowid

    log_activity(new_id, "lead_discovered", f"Discovered & Enriched: {company_name}",
                 f"AI enriched telemetry: ICP {score_res['icp_score']}/100, Urgency {score_res['urgency_score']}/100.", conn=conn)

    conn.commit()
    conn.close()

    return jsonify({"success": True, "lead_id": new_id, "score": score_res})

@app.route("/api/leads/discover-batch", methods=["POST"])
def discover_batch():
    discovered_pool = [
        {
            "company_name": "Synthetix Health",
            "contact_name": "Dr. Maya Lin",
            "contact_title": "Head of Clinical IT & Hardware Systems",
            "email": f"mlin.{random.randint(100,999)}@synthetixhealth.com",
            "industry": "Healthcare AI",
            "company_size": "140 employees",
            "estimated_budget": "$110,000",
            "tech_stack": "AWS HealthOmics, HIPAA macOS, Zero Trust",
            "buying_signals": "Expanding telemetry engineering squad; posted procurement request for HIPAA-hardened mobile workstations"
        },
        {
            "company_name": "KiteFin Capital",
            "contact_name": "Julian Mercer",
            "contact_title": "Chief Technology Officer",
            "email": f"jmercer.{random.randint(100,999)}@kitefincap.com",
            "industry": "Fintech & Algorithmic Trading",
            "company_size": "210 employees",
            "estimated_budget": "$190,000",
            "tech_stack": "Linux Dev Stations, Bloomberg Terminals, Low Latency Network",
            "buying_signals": "Completed Series B; replacing 80+ aging ThinkPad laptops with high-clock-rate engineering hardware"
        },
        {
            "company_name": "EchoLogistics Automation",
            "contact_name": "Rachel Adams",
            "contact_title": "VP of Supply Chain Systems",
            "email": f"radams.{random.randint(100,999)}@echologistics.io",
            "industry": "Logistics Software",
            "company_size": "95 employees",
            "estimated_budget": "$70,000",
            "tech_stack": "Google Workspace, Dell Latitude, Zebra Handhelds",
            "buying_signals": "Opening 2 regional dispatch facilities; procurement RFP active for fleet hardware"
        }
    ]

    conn = get_connection()
    cursor = conn.cursor()
    created_leads = []
    now = datetime.now().strftime("%Y-%m-%d %H:%M:%S")

    for item in discovered_pool:
        cursor.execute("SELECT id FROM leads WHERE company_name LIKE ?", (f"%{item['company_name']}%",))
        if cursor.fetchone():
            continue

        score_res = SalesAutopilotEngine.calculate_lead_score(item)
        cursor.execute("""
            INSERT INTO leads (
                company_name, contact_name, contact_title, email, industry, company_size,
                estimated_budget, tech_stack, buying_signals, icp_score, urgency_score,
                overall_score, scoring_breakdown, stage, priority, created_at, updated_at
            ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, 'scored', ?, ?, ?)
        """, (
            item["company_name"], item["contact_name"], item["contact_title"], item["email"],
            item["industry"], item["company_size"], item["estimated_budget"], item["tech_stack"],
            item["buying_signals"], score_res["icp_score"], score_res["urgency_score"],
            score_res["overall_score"], json.dumps(score_res["scoring_breakdown"]),
            score_res["priority"], now, now
        ))
        new_id = cursor.lastrowid
        log_activity(new_id, "lead_discovered", f"Autonomous Discovery: {item['company_name']}",
                     f"Scored {score_res['overall_score']}/100. Signal: {item['buying_signals'][:60]}...", conn=conn)
        created_leads.append(new_id)

    conn.commit()
    conn.close()
    return jsonify({"success": True, "discovered_count": len(created_leads), "lead_ids": created_leads})

@app.route("/api/leads/<int:lead_id>/generate-outreach", methods=["POST"])
def generate_outreach(lead_id):
    data = request.json or {}
    tone = data.get("tone", "consultative")
    touch_number = int(data.get("touch_number", 1))

    conn = get_connection()
    cursor = conn.cursor()

    cursor.execute("SELECT * FROM leads WHERE id = ?", (lead_id,))
    lead = cursor.fetchone()
    if not lead:
        conn.close()
        return jsonify({"error": "Lead not found"}), 404

    lead_dict = dict(lead)
    if lead_dict["is_opted_out"]:
        conn.close()
        return jsonify({"error": "Cannot generate outreach: Lead is opted out due to safeguard suppression."}), 400

    draft_data = SalesAutopilotEngine.generate_personalized_outreach(lead_dict, tone=tone, touch_number=touch_number)
    now = datetime.now().strftime("%Y-%m-%d %H:%M:%S")

    cursor.execute("""
        INSERT INTO outreach_drafts (lead_id, channel, subject, body, personalization_hooks, tone, status, touch_number, created_at)
        VALUES (?, ?, ?, ?, ?, ?, 'pending_approval', ?, ?)
    """, (
        lead_id, draft_data["channel"], draft_data["subject"], draft_data["body"],
        draft_data["personalization_hooks"], tone, touch_number, now
    ))
    draft_id = cursor.lastrowid

    if lead_dict["stage"] in ["discovered", "scored"]:
        cursor.execute("UPDATE leads SET stage = 'outreach_drafted', updated_at = ? WHERE id = ?", (now, lead_id))

    log_activity(lead_id, "outreach_generated", f"Personalized Draft ({tone.capitalize()}) Generated",
                 f"Subject: '{draft_data['subject']}'. Placed in Human Review Queue.", conn=conn)

    conn.commit()
    conn.close()
    return jsonify({"success": True, "draft_id": draft_id, "draft": draft_data})

@app.route("/api/autopilot/run-cycle", methods=["POST"])
def run_autopilot_cycle():
    result = SalesAutopilotEngine.run_autopilot_cycle()
    return jsonify({"success": True, **result})

@app.route("/api/outreach/pending", methods=["GET"])
def get_pending_outreach():
    conn = get_connection()
    cursor = conn.cursor()

    cursor.execute("""
        SELECT d.*, l.company_name, l.contact_name, l.contact_title, l.overall_score, l.buying_signals
        FROM outreach_drafts d
        JOIN leads l ON d.lead_id = l.id
        WHERE d.status = 'pending_approval'
        ORDER BY d.id DESC
    """)
    drafts = [dict(r) for r in cursor.fetchall()]

    for d in drafts:
        if d.get("personalization_hooks"):
            try:
                d["personalization_hooks"] = json.loads(d["personalization_hooks"])
            except Exception:
                pass

    conn.close()
    return jsonify(drafts)

@app.route("/api/outreach/<int:draft_id>/approve", methods=["POST"])
def approve_outreach(draft_id):
    conn = get_connection()
    cursor = conn.cursor()

    cursor.execute("SELECT * FROM outreach_drafts WHERE id = ?", (draft_id,))
    draft = cursor.fetchone()
    if not draft:
        conn.close()
        return jsonify({"error": "Draft not found"}), 404

    lead_id = draft["lead_id"]
    now = datetime.now().strftime("%Y-%m-%d %H:%M:%S")

    cursor.execute("""
        UPDATE outreach_drafts
        SET status = 'sent', approved_at = ?, sent_at = ?
        WHERE id = ?
    """, (now, now, draft_id))

    cursor.execute("""
        UPDATE leads
        SET stage = 'in_sequence', updated_at = ?
        WHERE id = ?
    """, (now, lead_id))

    log_activity(lead_id, "human_approved", "Outreach Approved & Sent",
                 f"Approved Touch {draft['touch_number']} to {draft['subject']}. Lead moved to 'In Sequence'.", conn=conn)

    conn.commit()
    conn.close()
    return jsonify({"success": True, "status": "sent"})

@app.route("/api/outreach/<int:draft_id>/reject", methods=["POST"])
def reject_outreach(draft_id):
    conn = get_connection()
    cursor = conn.cursor()

    cursor.execute("SELECT * FROM outreach_drafts WHERE id = ?", (draft_id,))
    draft = cursor.fetchone()
    if not draft:
        conn.close()
        return jsonify({"error": "Draft not found"}), 404

    lead_id = draft["lead_id"]
    cursor.execute("UPDATE outreach_drafts SET status = 'rejected' WHERE id = ?", (draft_id,))
    log_activity(lead_id, "outreach_rejected", "Outreach Draft Rejected",
                 "Sales rep declined to send message draft.", conn=conn)

    conn.commit()
    conn.close()
    return jsonify({"success": True, "status": "rejected"})

@app.route("/api/outreach/<int:draft_id>/edit", methods=["POST"])
def edit_outreach(draft_id):
    data = request.json or {}
    subject = data.get("subject", "").strip()
    body = data.get("body", "").strip()

    if not subject or not body:
        return jsonify({"error": "Subject and body cannot be empty"}), 400

    conn = get_connection()
    cursor = conn.cursor()

    cursor.execute("""
        UPDATE outreach_drafts
        SET subject = ?, body = ?
        WHERE id = ?
    """, (subject, body, draft_id))

    conn.commit()
    conn.close()
    return jsonify({"success": True})

@app.route("/api/replies/simulate", methods=["POST"])
def simulate_reply():
    data = request.json or {}
    lead_id = data.get("lead_id")
    reply_text = data.get("reply_text", "").strip()

    if not lead_id or not reply_text:
        return jsonify({"error": "lead_id and reply_text are required"}), 400

    result = SalesAutopilotEngine.process_reply_and_update_crm(lead_id, reply_text)
    if "error" in result:
        return jsonify(result), 400

    return jsonify(result)

@app.route("/api/activities", methods=["GET"])
def get_activities():
    conn = get_connection()
    cursor = conn.cursor()

    cursor.execute("""
        SELECT a.*, l.company_name
        FROM activity_logs a
        LEFT JOIN leads l ON a.lead_id = l.id
        ORDER BY a.id DESC
        LIMIT 50
    """)
    logs = [dict(r) for r in cursor.fetchall()]
    conn.close()
    return jsonify(logs)

@app.route("/api/cadence", methods=["GET"])
def get_cadence():
    conn = get_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM cadence_steps ORDER BY step_number ASC")
    steps = [dict(r) for r in cursor.fetchall()]
    conn.close()
    return jsonify(steps)

@app.route("/api/safeguards/suppression-list", methods=["GET", "POST"])
def suppression_list():
    conn = get_connection()
    cursor = conn.cursor()

    if request.method == "POST":
        data = request.json or {}
        email = data.get("email", "").strip()
        reason = data.get("reason", "Manual user addition to DNC list").strip()
        if not email:
            conn.close()
            return jsonify({"error": "Email is required"}), 400

        now = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        cursor.execute("""
            UPDATE leads
            SET is_opted_out = 1, opt_out_reason = ?, stage = 'opted_out', updated_at = ?
            WHERE email = ?
        """, (reason, now, email))

        log_activity(None, "safeguard_triggered", f"Suppression Added: {email}",
                     f"Manually added to Do-Not-Contact list. Reason: {reason}", conn=conn)
        conn.commit()
        conn.close()
        return jsonify({"success": True})

    cursor.execute("""
        SELECT id, company_name, contact_name, email, opt_out_reason, updated_at
        FROM leads
        WHERE is_opted_out = 1
        ORDER BY id DESC
    """)
    items = [dict(r) for r in cursor.fetchall()]
    conn.close()
    return jsonify(items)

@app.route("/api/settings/icp", methods=["GET", "POST"])
def icp_settings():
    conn = get_connection()
    cursor = conn.cursor()

    if request.method == "POST":
        data = request.json or {}
        for k, v in data.items():
            cursor.execute("INSERT OR REPLACE INTO settings (key, value) VALUES (?, ?)", (k, str(v)))
        conn.commit()
        conn.close()
        return jsonify({"success": True})

    cursor.execute("SELECT key, value FROM settings")
    settings = dict(cursor.fetchall())
    conn.close()
    return jsonify(settings)

@app.route("/api/reset-demo", methods=["POST"])
def reset_demo():
    import os
    from database import DB_PATH
    if os.path.exists(DB_PATH):
        os.remove(DB_PATH)
    init_db()
    seed_database()
    return jsonify({"success": True, "message": "Demo data reset to initial state."})

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5001, debug=False)
```

---

## 📄 `engine.py`
**Description**: AI Autopilot Intelligence Engine (Scoring, Outreach, NLP)  
**File Path**: `/Users/kruthikaran/.gemini/antigravity/scratch/sales-pipeline-autopilot/engine.py`

```python
import re
import json
from datetime import datetime, timedelta
from database import get_connection, log_activity

class SalesAutopilotEngine:
    """
    AI-powered Sales Pipeline Autopilot Engine.
    Handles enrichment, scoring, multi-tone outreach generation, response intelligence,
    cadence automation, and automated CRM pipeline state transitions.
    """

    @staticmethod
    def calculate_lead_score(lead, weights=None):
        """
        Calculates ICP Fit (0-100), Urgency/Buying Signals (0-100), Authority (0-100),
        and overall score with detailed AI reasoning.
        """
        if weights is None:
            weights = {"icp": 0.40, "urgency": 0.35, "authority": 0.25}

        signals = (lead.get("buying_signals") or "").lower()
        title = (lead.get("contact_title") or "").lower()
        size = str(lead.get("company_size") or "")
        industry = (lead.get("industry") or "").lower()
        budget = (lead.get("estimated_budget") or "").lower()

        # 1. Authority Score
        authority_score = 50
        if any(w in title for w in ["chief", "cio", "cto", "ceo", "vp", "vice president", "head of"]):
            authority_score = 95
        elif any(w in title for w in ["director", "lead", "principal"]):
            authority_score = 80
        elif any(w in title for w in ["manager", "supervisor"]):
            authority_score = 65
        elif any(w in title for w in ["owner", "partner", "founder"]):
            authority_score = 90
        elif any(w in title for w in ["analyst", "engineer", "associate", "intern"]):
            authority_score = 35

        # 2. ICP Fit Score
        icp_score = 55
        if any(s in size for s in ["100", "150", "200", "250", "300", "350", "400", "500"]):
            icp_score += 35
        elif any(s in size for s in ["50", "60", "70", "80", "65", "85", "95"]):
            icp_score += 25
        elif any(s in size for s in ["1,000", "1200", "5000", "enterprise"]):
            icp_score += 20
        elif any(s in size for s in ["5", "10", "12", "15", "micro", "1-10"]):
            icp_score -= 30

        # Industry alignment
        if any(ind in industry for ind in ["software", "ai", "cloud", "saas", "biotech", "cybersecurity", "tech", "fintech"]):
            icp_score += 15
        elif any(ind in industry for ind in ["retail", "ecommerce", "media", "finance", "logistics"]):
            icp_score += 8
        elif any(ind in industry for ind in ["accounting", "local", "plumbing"]):
            icp_score -= 15

        icp_score = max(10, min(100, icp_score))

        # 3. Urgency / Buying Signal Score
        urgency_score = 40
        reasons = []

        if any(w in signals for w in ["rfp", "procurement", "renewal in", "active", "complained", "degradation"]):
            urgency_score += 45
            reasons.append("Active procurement intent or immediate dissatisfaction with current vendor")
        if any(w in signals for w in ["hiring", "hires", "expanding", "opened", "opening"]):
            urgency_score += 30
            reasons.append("Team scaling creates immediate hardware & onboarding demand")
        if any(w in signals for w in ["funding", "series a", "series b", "series c", "raised"]):
            urgency_score += 25
            reasons.append("Recent capital injection indicates fresh operational budget")
        if any(w in signals for w in ["delay", "budget freeze", "audit", "next quarter"]):
            urgency_score -= 20
            reasons.append("Timing friction detected in buying signals")
        if any(w in signals for w in ["no expansion", "low-budget", "maintenance only"]):
            urgency_score -= 30
            reasons.append("No active expansion or capital expenditure")

        urgency_score = max(10, min(100, urgency_score))

        # Overall weighted score
        w_icp = float(weights.get("icp", 0.40))
        w_urg = float(weights.get("urgency", 0.35))
        w_auth = float(weights.get("authority", 0.25))
        overall = int(round((icp_score * w_icp) + (urgency_score * w_urg) + (authority_score * w_auth)))
        overall = max(5, min(100, overall))

        if overall >= 85:
            summary = f"Tier 1 Target ({overall}/100): High decision authority ({authority_score}) combined with urgent buying signals ({urgency_score}). Prioritize for immediate outreach."
        elif overall >= 65:
            summary = f"Tier 2 Qualified ({overall}/100): Good ICP fit ({icp_score}) and moderate urgency ({urgency_score}). Proceed with consultative engagement."
        elif overall >= 45:
            summary = f"Tier 3 Nurture ({overall}/100): Meets basic criteria but lacks immediate buying urgency or budget clarity."
        else:
            summary = f"Disqualified / Low Fit ({overall}/100): Headcount or budget constraints below minimum criteria."

        breakdown = {
            "icp_fit": icp_score,
            "buying_signals": urgency_score,
            "authority_level": authority_score,
            "summary": summary,
            "signal_highlights": reasons
        }

        priority = "high" if overall >= 80 else ("medium" if overall >= 55 else "low")

        return {
            "icp_score": icp_score,
            "urgency_score": urgency_score,
            "overall_score": overall,
            "priority": priority,
            "scoring_breakdown": breakdown
        }

    @staticmethod
    def generate_personalized_outreach(lead, tone="consultative", touch_number=1):
        """
        Creates contextual, hyper-personalized 1:1 outreach drafts tailored to company signals,
        tech stack, contact role, selected tone, and cadence touch step.
        """
        if lead.get("is_opted_out"):
            return {
                "error": "Lead is opted out. Autopilot safeguard prevents message generation."
            }

        first_name = lead.get("contact_name", "there").split()[0]
        company = lead.get("company_name", "").split("(")[0].strip()
        signals = lead.get("buying_signals", "")
        tech = lead.get("tech_stack", "")
        title = lead.get("contact_title", "Leader")

        # Touch 2: Technical & Case Study
        if touch_number == 2:
            subject = f"Fleet zero-touch provisioning workflow for {company}"
            body = (
                f"Hi {first_name},\n\n"
                f"Following up on my previous note regarding {company}'s hardware infrastructure.\n\n"
                f"For engineering teams operating on {tech.split(',')[0] if tech else 'hybrid stacks'}, "
                f"our platform enables fully automated MDM deployment—meaning new hires open their laptop box, "
                f"sign in with SSO, and have their full development toolchain configured in 15 minutes.\n\n"
                f"I've attached our 2-page benchmark comparison against conventional IT leasing. "
                f"Would 10 minutes next Wednesday work for a quick look?\n\n"
                f"Best,\nSales Autopilot"
            )
            hook = "Touch 2: Technical architecture & MDM workflow"
            return {
                "channel": "email",
                "subject": subject,
                "body": body,
                "tone": tone,
                "touch_number": 2,
                "personalization_hooks": json.dumps({"signal_cited": hook, "tone": tone})
            }

        # Touch 3: Executive ROI / Peer Proof
        if touch_number == 3:
            subject = f"How peer engineering teams cut device downtime by 90%"
            body = (
                f"Hi {first_name},\n\n"
                f"I wanted to share a quick data point: similar high-growth teams at your scale ({lead.get('company_size', '100+')} staff) "
                f"reduced IT ticket volume by 42% after switching from fragmented device procurement to our automated fleet model.\n\n"
                f"If hardware logistics isn't a current bottleneck at {company}, no worries at all—happy to reconnect next quarter. "
                f"Otherwise, let me know if you'd like our pricing matrix.\n\n"
                f"Best regards,\nSales Autopilot"
            )
            hook = "Touch 3: Peer ROI & downtime reduction metrics"
            return {
                "channel": "email",
                "subject": subject,
                "body": body,
                "tone": tone,
                "touch_number": 3,
                "personalization_hooks": json.dumps({"signal_cited": hook, "tone": tone})
            }

        # Touch 1 with Tone Variations
        if tone == "executive":
            if "hiring" in signals.lower() or "hires" in signals.lower():
                subject = f"{company}'s engineering expansion: fleet CapEx & SLA"
                body = (
                    f"Hi {first_name},\n\n"
                    f"With {company} expanding technical headcount this quarter, ensuring fast, friction-free onboarding "
                    f"without tied-up CapEx is critical.\n\n"
                    f"We provide enterprise laptop fleets with zero upfront hardware costs, 48-hour global replacement SLAs, "
                    f"and automated Jamf/Intune provisioning.\n\n"
                    f"Do you have 10 minutes next Tuesday for an executive briefing?\n\n"
                    f"Best regards,\nSales Autopilot"
                )
            else:
                subject = f"Optimizing {company}'s IT fleet lifecycle & budget"
                body = (
                    f"Hi {first_name},\n\n"
                    f"I wanted to share how we help companies in {lead.get('industry', 'your space')} reduce hardware procurement costs "
                    f"by 30% while guaranteeing zero engineering downtime.\n\n"
                    f"Could we connect for a brief 10-minute overview next week?\n\n"
                    f"Best regards,\nSales Autopilot"
                )
            hook = "Executive CapEx optimization & SLA guarantees"

        elif tone == "challenger":
            subject = f"Is IT hardware onboarding slowing down {company}?"
            body = (
                f"Hi {first_name},\n\n"
                f"Most growing teams in {lead.get('industry', 'tech')} lose an average of 12 productive hours per new developer hire "
                f"just wrestling with device imaging, delayed shipments, and battery issues.\n\n"
                f"Given {company}'s recent moves ({signals[:60]}...), our zero-touch pre-configured fleet eliminates this overhead completely.\n\n"
                f"Are you open to seeing how our SLA benchmarks compare to your current vendor?\n\n"
                f"Best regards,\nSales Autopilot"
            )
            hook = "Challenger: eliminating new-hire onboarding latency"

        elif tone == "casual":
            subject = f"Quick note for {first_name} re: {company} team hardware"
            body = (
                f"Hey {first_name},\n\n"
                f"Came across {company} and loved what you're building in {lead.get('industry', 'your field')}. "
                f"Saw you're scaling up the team recently!\n\n"
                f"We help fast-moving engineering teams get custom M3 & high-spec laptops delivered next-day with zero setup hassle.\n\n"
                f"Worth a quick 5-min chat sometime this week?\n\n"
                f"Cheers,\nSales Autopilot"
            )
            hook = "Casual founder-to-leader friendly outreach"

        else: # consultative (default)
            if "hiring" in signals.lower() or "hires" in signals.lower():
                subject = f"Supporting {company}'s engineering onboarding & fleet scale"
                body = (
                    f"Hi {first_name},\n\n"
                    f"I saw that {company} is actively scaling its technical team right now. "
                    f"As engineering headcount grows, IT procurement and custom zero-touch laptop provisioning "
                    f"often turn into an unexpected bottleneck for leadership.\n\n"
                    f"We help high-growth teams equip new hires on Day 1 with pre-configured, high-performance hardware, "
                    f"guaranteed next-day hardware swaps, and unified MDM integration.\n\n"
                    f"Would you be open to a 10-minute intro this week to compare our fleet SLA and pricing against traditional leasing?\n\n"
                    f"Best regards,\nSales Autopilot"
                )
                hook = "Accelerated hiring & Day-1 onboarding relief"
            elif "series" in signals.lower() or "funding" in signals.lower():
                subject = f"Scaling distributed workstations post-{company}'s funding milestone"
                body = (
                    f"Hi {first_name},\n\n"
                    f"Huge congratulations on the recent funding round at {company}!\n\n"
                    f"With rapid post-raise expansion, ensuring distributed employees receive dependable, secure laptops "
                    f"without tied-up capital or procurement delays is critical.\n\n"
                    f"We offer full fleet lifecycle management with zero upfront CapEx, saving companies like yours 30%+ "
                    f"on device maintenance while ensuring SOC2-level compliance out of the box.\n\n"
                    f"Could we connect for a brief 12-minute review next Tuesday?\n\n"
                    f"Best regards,\nSales Autopilot"
                )
                hook = "Funding announcement & CapEx optimization"
            elif "renewal" in signals.lower() or "rfp" in signals.lower() or "complained" in signals.lower():
                subject = f"Fleet renewal alternative for {company}"
                body = (
                    f"Hi {first_name},\n\n"
                    f"Given {company}'s current review of enterprise hardware and upcoming fleet milestones, "
                    f"I wanted to reach out regarding our enterprise workstation program.\n\n"
                    f"Unlike standard vendor contracts locked into rigid 3-year refresh cycles, our agile fleet model "
                    f"delivers 99.9% uptime guarantees, instant battery/hardware swaps, and seamless {tech.split(',')[0] if tech else 'cloud'} compatibility.\n\n"
                    f"Do you have 15 minutes this Thursday or Friday to inspect our benchmark specs and enterprise discount tiers?\n\n"
                    f"Best regards,\nSales Autopilot"
                )
                hook = "Upcoming contract renewal & vendor dissatisfaction"
            else:
                subject = f"Upgrading hardware productivity for {company}'s team"
                body = (
                    f"Hi {first_name},\n\n"
                    f"I've been following {company}'s work in {lead.get('industry', 'your industry')} and noticed your tech team "
                    f"operates on {tech or 'high-performance tech stacks'}.\n\n"
                    f"We provide enterprise laptop fleets designed specifically for demanding workflows, featuring zero-touch "
                    f"remote deployment and 24/7 dedicated hardware support.\n\n"
                    f"Would you have 10 minutes next week to see how we compare to your current IT setup?\n\n"
                    f"Best regards,\nSales Autopilot"
                )
                hook = "Tech stack alignment & operational efficiency"

        return {
            "channel": "email",
            "subject": subject,
            "body": body,
            "tone": tone,
            "touch_number": 1,
            "personalization_hooks": json.dumps({
                "signal_cited": hook,
                "recipient_role": title,
                "tone": tone
            })
        }

    @staticmethod
    def analyze_reply_intent(reply_text):
        """
        AI Response Intelligence:
        Classifies incoming prospect replies into structured intent categories,
        extracts dates/delays, detects sentiment, and formulates automated CRM actions.
        """
        text = reply_text.strip().lower()

        if any(w in text for w in ["unsubscribe", "remove me", "stop emailing", "take me off", "do not contact", "opt out", "spam"]):
            return {
                "sentiment": "opt_out",
                "intent": "opt_out",
                "confidence": 0.99,
                "delay_days": 0,
                "referral_name": None,
                "ai_reasoning": "Explicit opt-out or suppression request detected. Autopilot safeguard triggers immediate Do-Not-Contact compliance.",
                "recommended_action": "Mark lead as Opted Out, cancel all pending sequences, add domain to suppression list.",
                "crm_target_stage": "opted_out",
                "should_opt_out": True
            }

        meeting_keywords = ["demo", "call", "meet", "calendar", "thursday", "friday", "monday", "tuesday", "wednesday",
                            "next week", "zoom", "15-min", "20-min", "30-min", "discuss", "chat", "send invite", "schedule"]
        positive_keywords = ["interested", "sounds good", "sounds promising", "looks great", "yes", "sure", "love to", "happy to"]

        is_meeting_request = (
            any(w in text for w in ["demo", "calendar", "send an invite", "schedule a call", "set up a call", "jump on a call"]) or
            (any(p in text for p in positive_keywords) and any(m in text for m in ["call", "meet", "thursday", "friday", "next week", "time"]))
        )

        if is_meeting_request:
            return {
                "sentiment": "positive",
                "intent": "meeting_request",
                "confidence": 0.96,
                "delay_days": 0,
                "referral_name": None,
                "ai_reasoning": "Prospect confirmed interest and requested an introductory meeting/demo or suggested specific meeting times.",
                "recommended_action": "Advance deal stage to 'Demo / Meeting Booked', prepare calendar invite, and dispatch account executive alert.",
                "crm_target_stage": "meeting_booked",
                "should_opt_out": False
            }

        timing_keywords = ["next month", "next quarter", "in a few weeks", "q3", "q4", "later this year", "not right now",
                           "busy right now", "check back", "contact me in", "follow up in", "reach back out", "revisit"]
        if any(t in text for t in timing_keywords):
            delay_days = 30
            if "next week" in text:
                delay_days = 7
            elif "two weeks" in text or "2 weeks" in text:
                delay_days = 14
            elif "next month" in text:
                delay_days = 30
            elif "two months" in text or "2 months" in text:
                delay_days = 60
            elif "quarter" in text or "q3" in text or "q4" in text:
                delay_days = 90

            return {
                "sentiment": "neutral",
                "intent": "follow_up_later",
                "confidence": 0.94,
                "delay_days": delay_days,
                "referral_name": None,
                "ai_reasoning": f"Prospect expressed latent interest or budget timing block, asking to re-engage in approximately {delay_days} days.",
                "recommended_action": f"Transition CRM stage to 'Follow-Up Scheduled', auto-set task reminder for +{delay_days} days, pause active sequences until date.",
                "crm_target_stage": "follow_up_scheduled",
                "should_opt_out": False
            }

        referral_keywords = ["not the right person", "speak with", "talk to", "reach out to", "loop in", "refer you to", "contact our"]
        if any(r in text for r in referral_keywords):
            match_email = re.search(r'[\w\.-]+@[\w\.-]+\.\w+', text)
            referral_contact = match_email.group(0) if match_email else "Referred Colleague"

            return {
                "sentiment": "neutral",
                "intent": "referral",
                "confidence": 0.92,
                "delay_days": 1,
                "referral_name": referral_contact,
                "ai_reasoning": "Contact indicated they are not the primary decision maker and pointed toward a relevant internal stakeholder.",
                "recommended_action": f"Create new referral lead card for {referral_contact}, archive current contact as referred, draft warm intro outreach.",
                "crm_target_stage": "replied",
                "should_opt_out": False
            }

        objection_keywords = ["expensive", "cost", "budget", "pricing", "too much", "frozen", "no budget", "current contract"]
        if any(b in text for b in objection_keywords):
            return {
                "sentiment": "objection",
                "intent": "pricing_objection",
                "confidence": 0.89,
                "delay_days": 7,
                "referral_name": None,
                "ai_reasoning": "Prospect raised pricing, financial constraints, or budget freeze.",
                "recommended_action": "Flag for Human Sales Rep intervention with custom ROI calculator and flexible leasing tiers.",
                "crm_target_stage": "replied",
                "should_opt_out": False
            }

        info_keywords = ["send more info", "more information", "case study", "pricing sheet", "whitepaper", "specs", "documentation"]
        if any(i in text for i in info_keywords):
            return {
                "sentiment": "positive",
                "intent": "info_request",
                "confidence": 0.91,
                "delay_days": 2,
                "referral_name": None,
                "ai_reasoning": "Prospect requested collateral, technical specifications, or compliance documents.",
                "recommended_action": "Auto-generate Touch 2 email draft with product one-pager & customer case study.",
                "crm_target_stage": "replied",
                "should_opt_out": False
            }

        if any(n in text for n in ["not interested", "no thanks", "pass for now", "we are happy with"]):
            return {
                "sentiment": "negative",
                "intent": "not_interested",
                "confidence": 0.90,
                "delay_days": 0,
                "referral_name": None,
                "ai_reasoning": "Prospect declined engagement without a specific timing re-engagement window.",
                "recommended_action": "Move lead to Disqualified / Closed-Lost stage to preserve sales team focus.",
                "crm_target_stage": "disqualified",
                "should_opt_out": False
            }

        return {
            "sentiment": "neutral",
            "intent": "info_request",
            "confidence": 0.75,
            "delay_days": 3,
            "referral_name": None,
            "ai_reasoning": "General reply received. Requires contextual follow-up.",
            "recommended_action": "Review reply and draft customized follow-up response.",
            "crm_target_stage": "replied",
            "should_opt_out": False
        }

    @staticmethod
    def process_reply_and_update_crm(lead_id, reply_text):
        conn = get_connection()
        cursor = conn.cursor()

        cursor.execute("SELECT * FROM leads WHERE id = ?", (lead_id,))
        lead = cursor.fetchone()
        if not lead:
            conn.close()
            return {"error": "Lead not found"}

        lead_dict = dict(lead)
        analysis = SalesAutopilotEngine.analyze_reply_intent(reply_text)

        now_dt = datetime.now()
        now = now_dt.strftime("%Y-%m-%d %H:%M:%S")

        cursor.execute("""
            INSERT INTO replies (
                lead_id, reply_text, sentiment, intent, confidence,
                extracted_delay_days, extracted_referral_name,
                ai_reasoning, recommended_action, created_at
            ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
        """, (
            lead_id, reply_text, analysis["sentiment"], analysis["intent"],
            analysis["confidence"], analysis["delay_days"], analysis["referral_name"],
            analysis["ai_reasoning"], analysis["recommended_action"], now
        ))
        reply_id = cursor.lastrowid

        new_follow_up = None
        if analysis["delay_days"] > 0:
            new_follow_up = (now_dt + timedelta(days=analysis["delay_days"])).strftime("%Y-%m-%d")

        target_stage = analysis["crm_target_stage"]
        is_opted_out = 1 if analysis["should_opt_out"] else lead_dict["is_opted_out"]
        opt_out_reason = "Opt-out requested via inbound reply" if analysis["should_opt_out"] else lead_dict["opt_out_reason"]

        cursor.execute("""
            UPDATE leads
            SET stage = ?,
                follow_up_date = ?,
                is_opted_out = ?,
                opt_out_reason = ?,
                updated_at = ?
            WHERE id = ?
        """, (target_stage, new_follow_up, is_opted_out, opt_out_reason, now, lead_id))

        action_type = "safeguard_triggered" if analysis["should_opt_out"] else "intent_classified"
        title = f"AI Classified Reply: {analysis['intent'].replace('_', ' ').title()}"
        desc = (
            f"Sentiment: {analysis['sentiment'].capitalize()} | "
            f"Stage updated to: {target_stage.replace('_', ' ').title()} | "
            f"{analysis['ai_reasoning']}"
        )
        if new_follow_up:
            desc += f" [Follow-up reminder set for: {new_follow_up}]"

        log_activity(lead_id, action_type, title, desc, conn=conn)

        if analysis["should_opt_out"]:
            cursor.execute("""
                UPDATE outreach_drafts
                SET status = 'rejected'
                WHERE lead_id = ? AND status = 'pending_approval'
            """, (lead_id,))
            log_activity(lead_id, "safeguard_triggered", "Pending Sequences Cancelled",
                         "Automatic suppression safeguard blocked future outreach to opted-out recipient.", conn=conn)

        conn.commit()
        conn.close()

        return {
            "success": True,
            "reply_id": reply_id,
            "analysis": analysis,
            "updated_stage": target_stage,
            "follow_up_date": new_follow_up,
            "is_opted_out": bool(is_opted_out)
        }

    @staticmethod
    def run_autopilot_cycle():
        """
        Runs one complete autonomous cycle:
        1. Scores all unscored or discovered leads.
        2. Evaluates high-fit leads (score >= 60) in 'scored' stage and generates personalized outreach drafts.
        3. Logs activity trail.
        """
        conn = get_connection()
        cursor = conn.cursor()

        # Fetch settings
        cursor.execute("SELECT key, value FROM settings")
        settings = dict(cursor.fetchall())
        weights = {
            "icp": float(settings.get("icp_weight", 40)) / 100.0,
            "urgency": float(settings.get("urgency_weight", 35)) / 100.0,
            "authority": float(settings.get("authority_weight", 25)) / 100.0
        }
        preferred_tone = settings.get("default_tone", "consultative")

        # 1. Score unscored leads
        cursor.execute("SELECT * FROM leads WHERE stage = 'discovered'")
        discovered_leads = [dict(r) for r in cursor.fetchall()]
        scored_count = 0

        for l in discovered_leads:
            score_res = SalesAutopilotEngine.calculate_lead_score(l, weights)
            now = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
            cursor.execute("""
                UPDATE leads
                SET icp_score = ?, urgency_score = ?, overall_score = ?,
                    scoring_breakdown = ?, stage = 'scored', priority = ?, updated_at = ?
                WHERE id = ?
            """, (score_res["icp_score"], score_res["urgency_score"], score_res["overall_score"],
                  json.dumps(score_res["scoring_breakdown"]), score_res["priority"], now, l["id"]))
            log_activity(l["id"], "icp_scored", f"Autopilot Scored: {l['company_name']}",
                         f"Assigned score {score_res['overall_score']}/100 based on active ICP weights.", conn=conn)
            scored_count += 1

        # 2. Generate drafts for qualified leads in 'scored' stage without existing drafts
        cursor.execute("""
            SELECT l.* FROM leads l
            WHERE l.stage = 'scored' AND l.overall_score >= 60 AND l.is_opted_out = 0
            AND NOT EXISTS (SELECT 1 FROM outreach_drafts d WHERE d.lead_id = l.id)
        """)
        ready_leads = [dict(r) for r in cursor.fetchall()]
        drafts_generated = 0

        for lead in ready_leads:
            draft_data = SalesAutopilotEngine.generate_personalized_outreach(lead, tone=preferred_tone)
            now = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
            cursor.execute("""
                INSERT INTO outreach_drafts (lead_id, channel, subject, body, personalization_hooks, tone, status, touch_number, created_at)
                VALUES (?, ?, ?, ?, ?, ?, 'pending_approval', 1, ?)
            """, (lead["id"], draft_data["channel"], draft_data["subject"], draft_data["body"],
                  draft_data["personalization_hooks"], preferred_tone, now))

            cursor.execute("UPDATE leads SET stage = 'outreach_drafted', updated_at = ? WHERE id = ?", (now, lead["id"]))
            log_activity(lead["id"], "outreach_generated", f"Autopilot Outreach Drafted: {lead['company_name']}",
                         f"Generated 1:1 message ({preferred_tone} tone). Placed in Human Review Queue.", conn=conn)
            drafts_generated += 1

        conn.commit()
        conn.close()

        return {
            "scored_count": scored_count,
            "drafts_generated": drafts_generated,
            "summary": f"Autopilot cycle finished: Scored {scored_count} leads, Generated {drafts_generated} personalized drafts."
        }
```

---

## 📄 `database.py`
**Description**: SQLite Database Layer, Schemas & Seed Data  
**File Path**: `/Users/kruthikaran/.gemini/antigravity/scratch/sales-pipeline-autopilot/database.py`

```python
import sqlite3
import json
import os
from datetime import datetime, timedelta

DB_PATH = os.path.join(os.path.dirname(__file__), "autopilot.db")

def get_connection():
    conn = sqlite3.connect(DB_PATH, timeout=20.0)
    conn.row_factory = sqlite3.Row
    return conn

def init_db():
    conn = get_connection()
    cursor = conn.cursor()

    cursor.executescript("""
    CREATE TABLE IF NOT EXISTS users (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL,
        email TEXT NOT NULL UNIQUE,
        password TEXT NOT NULL,
        role TEXT NOT NULL,
        company_name TEXT,
        title TEXT,
        avatar TEXT,
        assigned_partner_id INTEGER,
        created_at TEXT NOT NULL
    );

    CREATE TABLE IF NOT EXISTS client_requirements (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        client_user_id INTEGER NOT NULL,
        company_name TEXT NOT NULL,
        product_name TEXT NOT NULL,
        value_proposition TEXT NOT NULL,
        target_industries TEXT NOT NULL,
        target_company_size TEXT NOT NULL,
        target_geography TEXT,
        key_buying_signals TEXT NOT NULL,
        pain_points_solved TEXT NOT NULL,
        pricing_tier TEXT,
        assigned_sales_exec_id INTEGER,
        assigned_sales_exec_name TEXT,
        assigned_sales_exec_email TEXT,
        notes_for_sales_exec TEXT,
        updated_at TEXT NOT NULL,
        FOREIGN KEY (client_user_id) REFERENCES users (id)
    );

    CREATE TABLE IF NOT EXISTS leads (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        company_name TEXT NOT NULL,
        contact_name TEXT NOT NULL,
        contact_title TEXT NOT NULL,
        email TEXT NOT NULL UNIQUE,
        industry TEXT NOT NULL,
        company_size TEXT NOT NULL,
        estimated_budget TEXT,
        tech_stack TEXT,
        buying_signals TEXT,
        icp_score INTEGER DEFAULT 0,
        urgency_score INTEGER DEFAULT 0,
        overall_score INTEGER DEFAULT 0,
        scoring_breakdown TEXT,
        stage TEXT NOT NULL DEFAULT 'discovered',
        priority TEXT DEFAULT 'medium',
        follow_up_date TEXT,
        is_opted_out INTEGER DEFAULT 0,
        opt_out_reason TEXT,
        created_at TEXT NOT NULL,
        updated_at TEXT NOT NULL
    );

    CREATE TABLE IF NOT EXISTS outreach_drafts (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        lead_id INTEGER NOT NULL,
        channel TEXT DEFAULT 'email',
        subject TEXT NOT NULL,
        body TEXT NOT NULL,
        personalization_hooks TEXT,
        tone TEXT DEFAULT 'consultative',
        status TEXT DEFAULT 'pending_approval',
        touch_number INTEGER DEFAULT 1,
        scheduled_for TEXT,
        created_at TEXT NOT NULL,
        approved_at TEXT,
        sent_at TEXT,
        FOREIGN KEY (lead_id) REFERENCES leads (id) ON DELETE CASCADE
    );

    CREATE TABLE IF NOT EXISTS replies (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        lead_id INTEGER NOT NULL,
        reply_text TEXT NOT NULL,
        sentiment TEXT NOT NULL,
        intent TEXT NOT NULL,
        confidence REAL DEFAULT 0.95,
        extracted_delay_days INTEGER DEFAULT 0,
        extracted_referral_name TEXT,
        ai_reasoning TEXT,
        recommended_action TEXT,
        created_at TEXT NOT NULL,
        FOREIGN KEY (lead_id) REFERENCES leads (id) ON DELETE CASCADE
    );

    CREATE TABLE IF NOT EXISTS activity_logs (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        lead_id INTEGER,
        action_type TEXT NOT NULL,
        title TEXT NOT NULL,
        description TEXT NOT NULL,
        created_at TEXT NOT NULL
    );

    CREATE TABLE IF NOT EXISTS cadence_steps (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        step_number INTEGER NOT NULL,
        delay_days INTEGER NOT NULL,
        channel TEXT DEFAULT 'email',
        title TEXT NOT NULL,
        purpose TEXT NOT NULL,
        template_focus TEXT NOT NULL
    );

    CREATE TABLE IF NOT EXISTS settings (
        key TEXT PRIMARY KEY,
        value TEXT
    );
    """)

    default_settings = {
        "company_pitch": "Next-Gen Enterprise High-Performance Laptops & Fleet Lifecycle Management",
        "target_icp_min_size": "50-500 employees",
        "target_industries": "Software, AI Infrastructure, Cloud SaaS, Biotech, Cybersecurity, Fintech",
        "icp_weight": "40",
        "urgency_weight": "35",
        "authority_weight": "25",
        "default_tone": "consultative",
        "auto_approval_mode": "false",
        "safeguard_duplicate_check": "true",
        "safeguard_dnc_active": "true"
    }

    for k, v in default_settings.items():
        cursor.execute("INSERT OR IGNORE INTO settings (key, value) VALUES (?, ?)", (k, v))

    cursor.execute("SELECT COUNT(*) as count FROM cadence_steps")
    if cursor.fetchone()["count"] == 0:
        default_cadence = [
            (1, 0, "email", "Touch 1: Signal-Anchored Value Prop", "Hook prospect on immediate expansion pain point & Day-1 hardware readiness.", "Hiring / Funding trigger"),
            (2, 3, "email", "Touch 2: Technical Deep Dive & Fleet SLA", "Demonstrate zero-touch MDM deployment and 99.9% hardware swap SLA.", "SOC2 compliance & Jamf integration"),
            (3, 7, "email", "Touch 3: Executive ROI & CapEx Case Study", "Share how similar peers cut IT onboarding lag by 85% and optimized leasing CapEx.", "Peer ROI benchmarks"),
            (4, 14, "email", "Touch 4: Low-Friction Check-in / Nurture Transition", "Courteous closing note giving them permission to delay if timing is off.", "Timing deferral permission")
        ]
        cursor.executemany("""
            INSERT INTO cadence_steps (step_number, delay_days, channel, title, purpose, template_focus)
            VALUES (?, ?, ?, ?, ?, ?)
        """, default_cadence)

    conn.commit()
    conn.close()

def log_activity(lead_id, action_type, title, description, conn=None):
    should_close = False
    if conn is None:
        conn = get_connection()
        should_close = True
    cursor = conn.cursor()
    now = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    cursor.execute("""
        INSERT INTO activity_logs (lead_id, action_type, title, description, created_at)
        VALUES (?, ?, ?, ?, ?)
    """, (lead_id, action_type, title, description, now))
    if should_close:
        conn.commit()
        conn.close()

def seed_database():
    conn = get_connection()
    cursor = conn.cursor()

    cursor.execute("SELECT COUNT(*) as count FROM leads")
    count = cursor.fetchone()["count"]

    now_dt = datetime.now()
    now = now_dt.strftime("%Y-%m-%d %H:%M:%S")

    # Seed users if empty
    cursor.execute("SELECT COUNT(*) as count FROM users")
    user_count = cursor.fetchone()["count"]
    if user_count == 0:
        # 1. Sales Executive
        cursor.execute("""
            INSERT INTO users (name, email, password, role, company_name, title, avatar, created_at)
            VALUES (?, ?, ?, ?, ?, ?, ?, ?)
        """, (
            "Alex Morgan", "alex.morgan@salesautopilot.io", "password123", "sales_executive",
            "Sales Pipeline Autopilot", "Senior Enterprise Account Executive", "👨‍💼", now
        ))
        sales_exec_id = cursor.lastrowid

        # 2. Client / Business Owner
        cursor.execute("""
            INSERT INTO users (name, email, password, role, company_name, title, avatar, assigned_partner_id, created_at)
            VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)
        """, (
            "Sarah Jenkins", "client@apextech.io", "password123", "client",
            "Apex Enterprise Systems", "VP of Business Development & Founder", "👩‍💼", sales_exec_id, now
        ))
        client_id = cursor.lastrowid

        # Seed Client Requirements
        cursor.execute("""
            INSERT INTO client_requirements (
                client_user_id, company_name, product_name, value_proposition,
                target_industries, target_company_size, target_geography,
                key_buying_signals, pain_points_solved, pricing_tier,
                assigned_sales_exec_id, assigned_sales_exec_name, assigned_sales_exec_email,
                notes_for_sales_exec, updated_at
            ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
        """, (
            client_id,
            "Apex Enterprise Systems",
            "Zero-Touch Enterprise Laptop Fleets & Cloud MDM Lifecycle",
            "We supply pre-configured, high-performance developer workstations (M3 Mac & Linux) with next-business-day swap guarantees, saving high-growth tech firms 8+ hours per employee onboarding.",
            "Software & AI, Cloud SaaS, Biotech & Genomics, Cybersecurity",
            "50-500 employees (High-Growth Mid-Market)",
            "North America & Remote EU",
            "Expanding engineering headcount (20+ hires), Recent Series A/B funding, Posted hardware procurement RFP, Complained of battery/hardware degradation",
            "Procurement lead time lag, onboarding delays, tied-up CapEx leasing agreements",
            "$120 - $180/device/month (All-Inclusive Fleet SLA)",
            sales_exec_id,
            "Alex Morgan",
            "alex.morgan@salesautopilot.io",
            "Alex, prioritize fast-growing Series A/B tech startups who just raised or posted tech hiring surges. Pitch our zero-touch remote provisioning.",
            now
        ))

    if count > 0:
        conn.commit()
        conn.close()
        return

    next_month = (now_dt + timedelta(days=30)).strftime("%Y-%m-%d")

    sample_leads = [
        {
            "company_name": "ABC Company (Apex Dynamics)",
            "contact_name": "Marcus Vance",
            "contact_title": "VP of Engineering & IT Operations",
            "email": "marcus.vance@apexdynamics.io",
            "industry": "Software & AI Infrastructure",
            "company_size": "150-200 employees",
            "estimated_budget": "$120,000 - $180,000",
            "tech_stack": "AWS, Kubernetes, Apple M3 fleet, Linux Devs",
            "buying_signals": "Hiring 35+ engineers this quarter; posted RFP for fleet hardware refresh; tech stack expansion",
            "icp_score": 96,
            "urgency_score": 94,
            "overall_score": 95,
            "scoring_breakdown": json.dumps({
                "icp_fit": 98,
                "buying_signals": 95,
                "authority_level": 92,
                "summary": "Perfect ICP match: High engineering headcount growth with active procurement RFP for laptop upgrades."
            }),
            "stage": "in_sequence",
            "priority": "high",
            "follow_up_date": (now_dt + timedelta(days=2)).strftime("%Y-%m-%d"),
            "is_opted_out": 0,
            "opt_out_reason": None
        },
        {
            "company_name": "XYZ Company (NexaCloud)",
            "contact_name": "Elena Rostova",
            "contact_title": "Head of People & IT Procurement",
            "email": "elena.r@nexacloud.tech",
            "industry": "Cloud SaaS",
            "company_size": "65 employees",
            "estimated_budget": "$40,000 - $60,000",
            "tech_stack": "Google Workspace, Slack, Dell Laptops, Jamf MDM",
            "buying_signals": "Recent Series A round ($12M); planning distributed hybrid workstation refresh across US & EU",
            "icp_score": 72,
            "urgency_score": 64,
            "overall_score": 68,
            "scoring_breakdown": json.dumps({
                "icp_fit": 75,
                "buying_signals": 68,
                "authority_level": 70,
                "summary": "Solid growth signals post-funding, but evaluating multiple vendors for mid-size fleet."
            }),
            "stage": "outreach_drafted",
            "priority": "medium",
            "follow_up_date": (now_dt + timedelta(days=5)).strftime("%Y-%m-%d"),
            "is_opted_out": 0,
            "opt_out_reason": None
        },
        {
            "company_name": "Starlight BioMed",
            "contact_name": "Dr. Aris Thorne",
            "contact_title": "Chief Information Officer",
            "email": "athorne@starlightbiomed.org",
            "industry": "Biotech & Genomics",
            "company_size": "320 employees",
            "estimated_budget": "$250,000+",
            "tech_stack": "High-Performance Compute, HIPAA compliant OS, ThinkPad P-Series",
            "buying_signals": "Opened two new bioinformatics wet-labs; heavy compute laptops needed for remote genomics analysis",
            "icp_score": 94,
            "urgency_score": 88,
            "overall_score": 91,
            "scoring_breakdown": json.dumps({
                "icp_fit": 95,
                "buying_signals": 90,
                "authority_level": 88,
                "summary": "Urgent compute requirement with strong compliance budgets allocated."
            }),
            "stage": "replied",
            "priority": "high",
            "follow_up_date": (now_dt + timedelta(days=1)).strftime("%Y-%m-%d"),
            "is_opted_out": 0,
            "opt_out_reason": None
        },
        {
            "company_name": "CyberShield Defense",
            "contact_name": "David Sterling",
            "contact_title": "Director of Security Engineering",
            "email": "dsterling@cybershielddef.com",
            "industry": "Cybersecurity Consulting",
            "company_size": "110 employees",
            "estimated_budget": "$90,000",
            "tech_stack": "Linux, Zero Trust Architecture, Hardware-level TPM 2.0",
            "buying_signals": "Complained on LinkedIn about battery degradation on current enterprise fleet; contract up for renewal in 45 days",
            "icp_score": 86,
            "urgency_score": 82,
            "overall_score": 84,
            "scoring_breakdown": json.dumps({
                "icp_fit": 88,
                "buying_signals": 84,
                "authority_level": 80,
                "summary": "Active dissatisfaction with incumbent vendor; renewal window approaching."
            }),
            "stage": "meeting_booked",
            "priority": "high",
            "follow_up_date": (now_dt + timedelta(days=1)).strftime("%Y-%m-%d"),
            "is_opted_out": 0,
            "opt_out_reason": None
        },
        {
            "company_name": "OmniRetail Global",
            "contact_name": "Claire Dupont",
            "contact_title": "VP of Retail Field Technology",
            "email": "cdupont@omniretail.com",
            "industry": "Retail & E-commerce",
            "company_size": "1,200 employees",
            "estimated_budget": "$300,000+",
            "tech_stack": "Windows Enterprise, Microsoft Intune, Legacy HP Fleet",
            "buying_signals": "Expanding retail store leadership; expressed interest but delayed budget review to next month",
            "icp_score": 79,
            "urgency_score": 55,
            "overall_score": 67,
            "scoring_breakdown": json.dumps({
                "icp_fit": 85,
                "buying_signals": 50,
                "authority_level": 82,
                "summary": "High deal value potential, but long decision cycles. Currently in delayed follow-up cadence."
            }),
            "stage": "follow_up_scheduled",
            "priority": "medium",
            "follow_up_date": next_month,
            "is_opted_out": 0,
            "opt_out_reason": None
        },
        {
            "company_name": "PQR Solutions",
            "contact_name": "Arthur Pendelton",
            "contact_title": "Owner / Managing Partner",
            "email": "arthur@pqrsolutions.biz",
            "industry": "Local Accounting & Tax",
            "company_size": "12 employees",
            "estimated_budget": "< $5,000",
            "tech_stack": "QuickBooks Desktop, Mixed refurb laptops",
            "buying_signals": "No current expansion; strictly low-budget maintenance",
            "icp_score": 25,
            "urgency_score": 15,
            "overall_score": 20,
            "scoring_breakdown": json.dumps({
                "icp_fit": 20,
                "buying_signals": 15,
                "authority_level": 30,
                "summary": "Below minimum headcount criteria; no procurement budget for enterprise hardware."
            }),
            "stage": "disqualified",
            "priority": "low",
            "follow_up_date": None,
            "is_opted_out": 0,
            "opt_out_reason": None
        },
        {
            "company_name": "Vanguard Media Group",
            "contact_name": "Chloe Bennett",
            "contact_title": "Head of Production",
            "email": "chloe.b@vanguardmedia.co",
            "industry": "Digital Media & Video Production",
            "company_size": "85 employees",
            "estimated_budget": "$75,000",
            "tech_stack": "Adobe Creative Cloud, DaVinci Resolve, Final Cut Pro",
            "buying_signals": "Recent hiring wave of 4K video editors; expressed 'Unsubscribe / Not interested at this time'",
            "icp_score": 60,
            "urgency_score": 40,
            "overall_score": 50,
            "scoring_breakdown": json.dumps({
                "icp_fit": 70,
                "buying_signals": 30,
                "authority_level": 65,
                "summary": "Opted out from automated outreach. Safeguard active."
            }),
            "stage": "opted_out",
            "priority": "low",
            "follow_up_date": None,
            "is_opted_out": 1,
            "opt_out_reason": "Recipient requested 'Unsubscribe' via inbound reply."
        }
    ]

    for lead in sample_leads:
        cursor.execute("""
            INSERT INTO leads (
                company_name, contact_name, contact_title, email, industry, company_size,
                estimated_budget, tech_stack, buying_signals, icp_score, urgency_score,
                overall_score, scoring_breakdown, stage, priority, follow_up_date, is_opted_out,
                opt_out_reason, created_at, updated_at
            ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
        """, (
            lead["company_name"], lead["contact_name"], lead["contact_title"], lead["email"],
            lead["industry"], lead["company_size"], lead["estimated_budget"], lead["tech_stack"],
            lead["buying_signals"], lead["icp_score"], lead["urgency_score"], lead["overall_score"],
            lead["scoring_breakdown"], lead["stage"], lead["priority"], lead["follow_up_date"],
            lead["is_opted_out"], lead["opt_out_reason"], now, now
        ))
        lead_id = cursor.lastrowid

        log_activity(lead_id, "lead_discovered", f"Lead Discovered: {lead['company_name']}",
                     f"Enriched profile with company size {lead['company_size']} and buying signals.", conn=conn)

        if lead["company_name"].startswith("ABC Company"):
            cursor.execute("""
                INSERT INTO outreach_drafts (lead_id, channel, subject, body, personalization_hooks, tone, status, touch_number, created_at, approved_at, sent_at)
                VALUES (?, 'email', ?, ?, ?, 'consultative', 'sent', 1, ?, ?, ?)
            """, (
                lead_id,
                "Scaling engineering fleet for Apex Dynamics's 35+ new hires",
                "Hi Marcus,\n\nI noticed Apex Dynamics is accelerating engineering hiring this quarter and recently opened an RFP for fleet hardware upgrades.\n\nWith distributed teams requiring high-performance M3/Linux workstation reliability, our automated zero-touch provisioning and next-business-day swap program saves engineering managers 8+ hours per onboarding.\n\nWould you be open to a quick 12-minute intro this Thursday to see our fleet pricing and specs?\n\nBest regards,\nAlex Morgan | Dedicated Enterprise Sales Executive",
                json.dumps({"signal_cited": "35+ engineering hires", "tone": "Consultative Engineering"}),
                now, now, now
            ))
            log_activity(lead_id, "outreach_sent", "Touch 1 Email Sent", "Personalized message citing engineering growth delivered to Marcus Vance.", conn=conn)

        elif lead["company_name"].startswith("XYZ Company"):
            cursor.execute("""
                INSERT INTO outreach_drafts (lead_id, channel, subject, body, personalization_hooks, tone, status, touch_number, created_at)
                VALUES (?, 'email', ?, ?, ?, 'executive', 'pending_approval', 1, ?)
            """, (
                lead_id,
                "Streamlining hardware onboarding post NexaCloud's Series A",
                "Hi Elena,\n\nCongratulations on NexaCloud's $12M Series A round! Scaling distributed teams across US and EU often creates IT logistical friction when deploying secure work laptops.\n\nWe provide automated global hardware deployment with Jamf-ready pre-configurations, reducing procurement lag from 3 weeks to 48 hours.\n\nWould next Tuesday at 2 PM work for a brief overview?\n\nBest,\nAlex Morgan | Dedicated Enterprise Sales Executive",
                json.dumps({"signal_cited": "$12M Series A funding", "tone": "Executive Direct"}),
                now
            ))
            log_activity(lead_id, "outreach_generated", "Draft Generated (Awaiting Approval)", "AI generated customized message citing Series A expansion. Queued for human review.", conn=conn)

        elif lead["company_name"].startswith("Starlight BioMed"):
            cursor.execute("""
                INSERT INTO replies (lead_id, reply_text, sentiment, intent, confidence, extracted_delay_days, ai_reasoning, recommended_action, created_at)
                VALUES (?, ?, 'positive', 'meeting_request', 0.96, 0, ?, ?, ?)
            """, (
                lead_id,
                "Thanks for reaching out! We are indeed setting up two new bioinformatics compute stations. Can you send over compliance documentation and jump on a 20-min demo this Friday at 11 AM EST?",
                "Prospect validated buying signal and actively requested a 20-min product demonstration with compliance verification.",
                "Book calendar invite for Friday 11:00 AM EST, attach SOC2 & HIPAA specs, notify account executive.",
                now
            ))
            log_activity(lead_id, "reply_received", "High-Intent Reply Classified", "Dr. Thorne requested demo for Friday 11:00 AM. Stage moved to Replied.", conn=conn)

        elif lead["company_name"].startswith("OmniRetail Global"):
            cursor.execute("""
                INSERT INTO replies (lead_id, reply_text, sentiment, intent, confidence, extracted_delay_days, ai_reasoning, recommended_action, created_at)
                VALUES (?, ?, 'neutral', 'follow_up_later', 0.94, 30, ?, ?, ?)
            """, (
                lead_id,
                "Hello, this looks promising for our field supervisors, but we are in the middle of our Q2 inventory audit. Please contact me next month when our hardware budget opens.",
                "Clear timing deferral. Positive latent interest, but action blocked until next month.",
                "Automatically scheduled smart nurture follow-up in 30 days. CRM stage set to Follow-Up Scheduled.",
                now
            ))
            log_activity(lead_id, "follow_up_scheduled", "Follow-up Scheduled (+30 Days)", "Detected delay request in reply. Calendar reminder set for next month.", conn=conn)

    conn.commit()
    conn.close()

if __name__ == "__main__":
    init_db()
    seed_database()
    print("Database updated and seeded successfully with users and requirements.")
```

---

## 📄 `test_autopilot.py`
**Description**: Automated Test Suite (11/11 Passing Tests)  
**File Path**: `/Users/kruthikaran/.gemini/antigravity/scratch/sales-pipeline-autopilot/test_autopilot.py`

```python
import unittest
import json
import os
from database import init_db, seed_database, get_connection, DB_PATH
from engine import SalesAutopilotEngine
from app import app

class TestSalesPipelineAutopilot(unittest.TestCase):
    @classmethod
    def setUpClass(cls):
        init_db()
        seed_database()
        cls.client = app.test_client()

    def test_01_stats_endpoint(self):
        res = self.client.get("/api/stats")
        self.assertEqual(res.status_code, 200)
        data = res.get_json()
        self.assertIn("total_leads", data)
        self.assertGreater(data["total_leads"], 0)
        self.assertIn("high_icp_leads", data)
        self.assertIn("pipeline_value_formatted", data)
        print("✓ Stats Endpoint OK:", data)

    def test_02_leads_endpoint(self):
        res = self.client.get("/api/leads")
        self.assertEqual(res.status_code, 200)
        leads = res.get_json()
        self.assertGreaterEqual(len(leads), 5)
        abc = next((l for l in leads if "ABC Company" in l["company_name"]), None)
        self.assertIsNotNone(abc)
        self.assertGreaterEqual(abc["overall_score"], 85)
        print("✓ Leads Listing & Scoring OK:", abc["company_name"], "Score:", abc["overall_score"])

    def test_03_ai_scoring_engine(self):
        high_lead = {
            "company_name": "Test High Fit",
            "contact_name": "Sarah Connor",
            "contact_title": "Chief Information Officer",
            "company_size": "200 employees",
            "industry": "Enterprise Cloud Software",
            "buying_signals": "Hiring 40 engineers; active procurement RFP posted"
        }
        score_res = SalesAutopilotEngine.calculate_lead_score(high_lead)
        self.assertGreaterEqual(score_res["overall_score"], 85)
        self.assertEqual(score_res["priority"], "high")

        low_lead = {
            "company_name": "Test Low Fit",
            "contact_name": "Bob Small",
            "contact_title": "Intern",
            "company_size": "5 employees",
            "industry": "Local Plumbing",
            "buying_signals": "No expansion, low-budget maintenance only"
        }
        score_res2 = SalesAutopilotEngine.calculate_lead_score(low_lead)
        self.assertLess(score_res2["overall_score"], 45)
        self.assertEqual(score_res2["priority"], "low")
        print("✓ AI Scoring Logic Verified (High:", score_res["overall_score"], "Low:", score_res2["overall_score"], ")")

    def test_04_personalized_outreach_generation(self):
        lead = {
            "company_name": "Apex Cloud Systems",
            "contact_name": "David Miller",
            "contact_title": "Head of Engineering",
            "industry": "Software",
            "tech_stack": "macOS M3, AWS",
            "buying_signals": "Hiring 30 engineers this quarter"
        }
        draft = SalesAutopilotEngine.generate_personalized_outreach(lead)
        self.assertIn("engineering", draft["subject"].lower())
        self.assertIn("David", draft["body"])
        self.assertIn("Apex Cloud Systems", draft["body"])
        print("✓ Personalized Outreach Generation OK:", draft["subject"])

    def test_05_response_intelligence_delay_intent(self):
        reply = "I'm interested, but contact me next month."
        analysis = SalesAutopilotEngine.analyze_reply_intent(reply)
        self.assertEqual(analysis["intent"], "follow_up_later")
        self.assertEqual(analysis["delay_days"], 30)
        self.assertEqual(analysis["crm_target_stage"], "follow_up_scheduled")
        print("✓ Response Intelligence (Delay 30d) OK:", analysis["ai_reasoning"])

    def test_06_response_intelligence_demo_request(self):
        reply = "Sounds interesting, can we do a 15-min demo this Thursday at 2 PM?"
        analysis = SalesAutopilotEngine.analyze_reply_intent(reply)
        self.assertEqual(analysis["intent"], "meeting_request")
        self.assertEqual(analysis["crm_target_stage"], "meeting_booked")
        print("✓ Response Intelligence (Demo Request) OK:", analysis["intent"])

    def test_07_response_intelligence_opt_out_safeguard(self):
        reply = "Please unsubscribe me and remove our company from your list."
        analysis = SalesAutopilotEngine.analyze_reply_intent(reply)
        self.assertEqual(analysis["intent"], "opt_out")
        self.assertTrue(analysis["should_opt_out"])
        self.assertEqual(analysis["crm_target_stage"], "opted_out")
        print("✓ Response Intelligence (Opt-out Safeguard) OK")

    def test_08_full_reply_simulation_api(self):
        res = self.client.post("/api/replies/simulate", json={
            "lead_id": 2,
            "reply_text": "I'm interested, but contact me next month."
        })
        self.assertEqual(res.status_code, 200)
        data = res.get_json()
        self.assertTrue(data["success"])
        self.assertEqual(data["updated_stage"], "follow_up_scheduled")
        self.assertIsNotNone(data["follow_up_date"])
        print("✓ Full Reply Simulation API OK. Stage updated to:", data["updated_stage"], "Date:", data["follow_up_date"])

    def test_09_duplicate_safeguard(self):
        res = self.client.post("/api/leads", json={
            "company_name": "ABC Company (Apex Dynamics)",
            "contact_name": "Marcus Vance",
            "email": "marcus.vance@apexdynamics.io",
            "industry": "Software"
        })
        self.assertEqual(res.status_code, 409)
        self.assertIn("Duplicate safeguard", res.get_json()["error"])
        print("✓ Duplicate Safeguard OK")

    def test_10_auth_and_roles(self):
        res = self.client.get("/api/auth/current-user")
        self.assertEqual(res.status_code, 200)
        user = res.get_json()
        self.assertIn("name", user)
        self.assertIn("role", user)
        print("✓ Current User OK:", user["name"], "(Role:", user["role"], ")")

        res = self.client.post("/api/auth/switch-role", json={"role": "sales_executive"})
        self.assertEqual(res.status_code, 200)
        switched = res.get_json()
        self.assertTrue(switched["success"])
        self.assertEqual(switched["user"]["role"], "sales_executive")
        print("✓ Switch Role OK:", switched["user"]["name"], "(Role:", switched["user"]["role"], ")")

    def test_11_client_requirements_sync(self):
        res = self.client.get("/api/client/requirements")
        self.assertEqual(res.status_code, 200)
        req = res.get_json()
        self.assertIn("product_name", req)
        self.assertIn("assigned_sales_exec_name", req)
        print("✓ Client Requirements OK:", req["product_name"], "-> Exec:", req["assigned_sales_exec_name"])

        res = self.client.post("/api/client/requirements", json={
            "company_name": "Apex Enterprise Systems",
            "product_name": "Zero-Touch M3 Workstation Fleets",
            "value_proposition": "Automated laptop provisioning with 24/7 SLA",
            "target_industries": "AI, Software, Cloud SaaS",
            "target_company_size": "50-500 employees",
            "key_buying_signals": "Hiring 20+ devs, Series A funding",
            "pain_points_solved": "Eliminating IT lag",
            "pricing_tier": "$150/dev/mo",
            "notes_for_sales_exec": "Target AI startups"
        })
        self.assertEqual(res.status_code, 200)
        self.assertTrue(res.get_json()["success"])
        print("✓ Update Client Requirements & Sync OK")

if __name__ == "__main__":
    unittest.main()
```

---

## 📄 `run.sh`
**Description**: One-Click Executable Launch Script  
**File Path**: `/Users/kruthikaran/.gemini/antigravity/scratch/sales-pipeline-autopilot/run.sh`

```bash
#!/usr/bin/env bash
cd "$(dirname "$0")"
echo "=========================================================="
echo " Starting Sales Pipeline Autopilot Prototype"
echo " Address: http://localhost:5001"
echo "=========================================================="
python3 app.py
```

---

## 📄 `templates/index.html`
**Description**: Frontend HTML Layout & Cockpit Structure  
**File Path**: `/Users/kruthikaran/.gemini/antigravity/scratch/sales-pipeline-autopilot/templates/index.html`

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Sales Pipeline Autopilot | Autonomous Revenue & CRM Engine</title>
  <link rel="stylesheet" href="/static/css/style.css">
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
  <link href="https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@400;500;600;700;800&family=JetBrains+Mono:wght@400;500;600&display=swap" rel="stylesheet">
</head>
<body class="app-layout">

  <!-- LEFT SIDEBAR WITH ALL OPTIONS DOWNWARDS -->
  <aside class="sidebar">
    <!-- Brand / Logo Header -->
    <div class="sidebar-brand">
      <div class="logo-icon">⚡</div>
      <div class="brand-text">
        <span class="brand-title">Sales Pipeline</span>
        <span class="brand-sub gradient-text">Autopilot v2.0</span>
      </div>
    </div>

    <!-- Logged In User Profile & Counterpart Card -->
    <div class="sidebar-user-card" onclick="openLoginModal()" title="Click to Switch User / Role">
      <div class="user-avatar" id="sidebar-user-avatar">👩‍💼</div>
      <div class="user-info">
        <div class="user-name-row">
          <span class="user-name" id="sidebar-user-name">Sarah Jenkins</span>
          <span class="role-pill" id="sidebar-user-role-pill">Client</span>
        </div>
        <div class="user-sub" id="sidebar-user-company">Apex Enterprise Systems</div>
        <div class="user-counterpart" id="sidebar-counterpart-text">
          Sales Exec: <strong>Alex Morgan</strong>
        </div>
      </div>
      <div class="user-switch-icon" title="Switch User">⇄</div>
    </div>

    <!-- Live Autopilot Status Indicator -->
    <div class="sidebar-status-box">
      <div class="status-top">
        <span class="pulse-dot"></span>
        <span class="status-text">Autopilot Active</span>
      </div>
      <div class="status-sub">AI Pipeline Sync • 1:1 Personalization</div>
    </div>

    <!-- MAIN NAVIGATION OPTIONS (VERTICALLY DOWNWARDS) -->
    <nav class="sidebar-nav">
      <div class="nav-section-label">Client & Strategy</div>

      <button class="nav-item" data-tab="requirements" onclick="switchTab('requirements')">
        <span class="nav-icon">🏢</span>
        <span class="nav-label">Client Requirements Hub</span>
        <span class="nav-badge badge-teal">Client</span>
      </button>

      <div class="nav-section-label">Pipeline & Execution</div>

      <button class="nav-item active" data-tab="kanban" onclick="switchTab('kanban')">
        <span class="nav-icon">📊</span>
        <span class="nav-label">Pipeline Kanban</span>
      </button>

      <button class="nav-item" data-tab="list" onclick="switchTab('list')">
        <span class="nav-icon">📋</span>
        <span class="nav-label">Lead Roster & Scoring</span>
      </button>

      <button class="nav-item" data-tab="review" onclick="switchTab('review')">
        <span class="nav-icon">✍️</span>
        <span class="nav-label">Human Review Queue</span>
        <span class="nav-badge badge-amber" id="review-badge-count">0</span>
      </button>

      <button class="nav-item highlight-item" data-tab="simulator" onclick="switchTab('simulator')">
        <span class="nav-icon">🧪</span>
        <span class="nav-label">Response Simulator</span>
        <span class="nav-badge badge-purple">Core</span>
      </button>

      <div class="nav-section-label">Automation & Guardrails</div>

      <button class="nav-item" data-tab="cadence" onclick="switchTab('cadence')">
        <span class="nav-icon">🚀</span>
        <span class="nav-label">Multi-Touch Sequences</span>
      </button>

      <button class="nav-item" data-tab="icp" onclick="switchTab('icp')">
        <span class="nav-icon">🎯</span>
        <span class="nav-label">ICP Rules & Scoring Weights</span>
      </button>

      <button class="nav-item" data-tab="safeguards" onclick="switchTab('safeguards')">
        <span class="nav-icon">🛡️</span>
        <span class="nav-label">Compliance & DNC List</span>
        <span class="nav-badge badge-teal" id="nav-dnc-count">0</span>
      </button>

      <button class="nav-item" data-tab="activity" onclick="switchTab('activity')">
        <span class="nav-icon">⚡</span>
        <span class="nav-label">Autopilot Audit Logs</span>
      </button>
    </nav>

    <!-- LEFT SIDEBAR ACTIONS (AT THE BOTTOM) -->
    <div class="sidebar-bottom-actions">
      <div class="nav-section-label">Quick Actions</div>

      <button class="sidebar-action-btn btn-autopilot-run" onclick="triggerAutopilotCycle()">
        <span class="action-icon">⚡</span>
        <div class="action-text">
          <span class="action-title">Run Autopilot Cycle</span>
          <span class="action-desc">Auto-score & draft outreach</span>
        </div>
      </button>

      <button class="sidebar-action-btn" onclick="discoverBatchLeads()">
        <span class="action-icon">🔍</span>
        <div class="action-text">
          <span class="action-title">AI Discover Leads</span>
          <span class="action-desc">Scan market for ICP matches</span>
        </div>
      </button>

      <button class="sidebar-action-btn" onclick="openAddLeadModal()">
        <span class="action-icon">➕</span>
        <div class="action-text">
          <span class="action-title">Add Prospect</span>
          <span class="action-desc">Enter custom account</span>
        </div>
      </button>

      <button class="sidebar-action-btn btn-reset" onclick="resetDemoData()">
        <span class="action-icon">↺</span>
        <div class="action-text">
          <span class="action-title">Reset Demo Data</span>
          <span class="action-desc">Restore prototype state</span>
        </div>
      </button>
    </div>
  </aside>

  <!-- MAIN APP CONTAINER -->
  <div class="app-main">

    <!-- TOP HEADER WITH KPI BANNER -->
    <header class="top-header">
      <div class="header-banner">
        <div class="stat-card">
          <div class="stat-label">Pipeline Value</div>
          <div class="stat-value" id="kpi-pipeline-val">$0</div>
          <div class="stat-trend positive">↑ Weighted High-Fit</div>
        </div>
        <div class="stat-card">
          <div class="stat-label">Total Prospects</div>
          <div class="stat-value" id="kpi-total-leads">0</div>
          <div class="stat-desc">Enriched Telemetry</div>
        </div>
        <div class="stat-card highlight">
          <div class="stat-label">High ICP Matches (80+)</div>
          <div class="stat-value" id="kpi-high-icp">0</div>
          <div class="stat-desc">Ideal Profile Fit</div>
        </div>
        <div class="stat-card">
          <div class="stat-label">Response Rate</div>
          <div class="stat-value" id="kpi-response-rate">0%</div>
          <div class="stat-trend positive">↑ 1:1 Personalization</div>
        </div>
        <div class="stat-card">
          <div class="stat-label">Meetings Booked</div>
          <div class="stat-value" id="kpi-meetings-booked">0</div>
          <div class="stat-desc">High-Intent Conversions</div>
        </div>
        <div class="stat-card alert-stat" onclick="switchTab('review')">
          <div class="stat-label">Review Queue</div>
          <div class="stat-value text-amber" id="kpi-pending-review">0</div>
          <div class="stat-desc">Pending Rep Sign-Off →</div>
        </div>
        <div class="stat-card safeguard-stat" onclick="switchTab('safeguards')">
          <div class="stat-label">Opt-Out Safeguards</div>
          <div class="stat-value text-teal" id="kpi-opt-outs">0</div>
          <div class="stat-desc">Consent Compliance →</div>
        </div>
      </div>
    </header>

    <!-- MAIN VIEW AREA -->
    <main class="content-body">

      <!-- 0. CLIENT REQUIREMENTS HUB (NEW VIEW) -->
      <section id="view-requirements" class="view-panel">
        <div class="view-header">
          <div>
            <h2>Client Requirements & Sales Executive Strategy Hub</h2>
            <p class="subtitle">The client defines company offerings, target criteria, and directs their dedicated sales executive to run the pipeline.</p>
          </div>
          <div>
            <button class="btn btn-secondary btn-sm" onclick="openLoginModal()">⇄ Switch Role / User</button>
          </div>
        </div>

        <div class="requirements-layout">
          <!-- Left: Client Requirements Form -->
          <div class="settings-card req-form-card">
            <div style="display:flex; justify-content:space-between; align-items:center; margin-bottom:12px;">
              <div>
                <h3>Client Business & Product Requirements</h3>
                <p class="settings-desc">Specify your product, target customer profile, and pain points for AI autopilot matching.</p>
              </div>
              <span class="badge badge-teal">Client Managed</span>
            </div>

            <form id="client-requirements-form" onsubmit="saveClientRequirements(event)">
              <div class="form-row">
                <div class="form-group">
                  <label>Client Company Name *</label>
                  <input type="text" id="req-company-name" class="form-control" required>
                </div>
                <div class="form-group">
                  <label>Product / Solution Name *</label>
                  <input type="text" id="req-product-name" class="form-control" required>
                </div>
              </div>

              <div class="form-group">
                <label>Core Value Proposition & Pitch *</label>
                <textarea id="req-value-prop" class="form-control" rows="3" required placeholder="What makes your solution superior?"></textarea>
              </div>

              <div class="form-row">
                <div class="form-group">
                  <label>Target Industries *</label>
                  <input type="text" id="req-target-industries" class="form-control" required placeholder="e.g. Software, Cloud SaaS, Biotech">
                </div>
                <div class="form-group">
                  <label>Ideal Company Size Range *</label>
                  <input type="text" id="req-company-size" class="form-control" required placeholder="e.g. 50-500 employees">
                </div>
              </div>

              <div class="form-row">
                <div class="form-group">
                  <label>Target Geography / Regions</label>
                  <input type="text" id="req-target-geography" class="form-control" placeholder="e.g. North America & Remote EU">
                </div>
                <div class="form-group">
                  <label>Pricing Model / Tier</label>
                  <input type="text" id="req-pricing-tier" class="form-control" placeholder="e.g. $120 - $180/device/month">
                </div>
              </div>

              <div class="form-group">
                <label>Key Buying Signals to Prioritize *</label>
                <textarea id="req-buying-signals" class="form-control" rows="2" placeholder="e.g. Hiring surges, Series A/B funding, hardware RFP announcements"></textarea>
              </div>

              <div class="form-group">
                <label>Pain Points Your Product Solves *</label>
                <textarea id="req-pain-points" class="form-control" rows="2" placeholder="e.g. Onboarding hardware latency, tied-up CapEx, IT device maintenance"></textarea>
              </div>

              <div class="form-group">
                <label>Directives & Notes for Assigned Sales Executive</label>
                <textarea id="req-exec-notes" class="form-control" rows="2" placeholder="Specific notes or guidelines for Alex Morgan when pitching prospects"></textarea>
              </div>

              <button type="submit" class="btn btn-primary btn-block btn-lg">
                <span>💾 Save & Synchronize Autopilot Engine</span>
              </button>
            </form>
          </div>

          <!-- Right: Assigned Sales Executive Profile & Relationship Card -->
          <div class="settings-card req-exec-card">
            <div style="display:flex; justify-content:space-between; align-items:center; margin-bottom:12px;">
              <h3>Assigned Sales Executive</h3>
              <span class="badge badge-green">Dedicated Rep</span>
            </div>
            <p class="settings-desc">Your designated sales professional who reviews outreach, manages prospect communication, and handles deal progression.</p>

            <div class="exec-profile-box">
              <div class="exec-avatar-lg" id="req-exec-avatar">👨‍💼</div>
              <div class="exec-details">
                <h4 id="req-exec-name">Alex Morgan</h4>
                <div class="exec-title" id="req-exec-title">Senior Enterprise Account Executive</div>
                <div class="exec-email" id="req-exec-email">alex.morgan@salesautopilot.io</div>
              </div>
            </div>

            <div class="exec-stats-grid">
              <div class="mini-stat">
                <div class="mini-stat-label">Pipeline Active</div>
                <div class="mini-stat-val" id="exec-pipeline-val">$445k</div>
              </div>
              <div class="mini-stat">
                <div class="mini-stat-label">Response Rate</div>
                <div class="mini-stat-val text-green" id="exec-response-rate">60%</div>
              </div>
              <div class="mini-stat">
                <div class="mini-stat-label">Demos Booked</div>
                <div class="mini-stat-val text-blue" id="exec-demos-booked">1</div>
              </div>
            </div>

            <div class="res-box-highlight" style="margin-top:16px;">
              <div class="res-title">🤝 Client & Sales Executive Workflow</div>
              <ul style="font-size:0.8rem; color:#cbd5e1; padding-left:18px; line-height:1.6; margin-top:6px;">
                <li><strong>Client defines:</strong> Product value proposition, ICP rules, and buying signals on this dashboard.</li>
                <li><strong>AI Autopilot discovers:</strong> Finds qualified accounts matching the client's specifications.</li>
                <li><strong>Sales Executive executes:</strong> Reviews personalized drafts in the Human Review Queue and oversees deal meetings.</li>
              </ul>
            </div>

            <div style="margin-top:20px; display:flex; gap:10px;">
              <button class="btn btn-secondary btn-block" onclick="switchTab('review')">
                <span>View Alex's Review Queue →</span>
              </button>
              <button class="btn btn-primary btn-block" onclick="switchUserRole('sales_executive')">
                <span>Switch to Sales Exec View ⇄</span>
              </button>
            </div>
          </div>
        </div>
      </section>

      <!-- 1. KANBAN PIPELINE VIEW -->
      <section id="view-kanban" class="view-panel active">
        <div class="view-header">
          <div>
            <h2>Sales Pipeline Board</h2>
            <p class="subtitle">Autonomous progression from discovery to meeting booking. Deals adapt in real-time as prospect signals evolve.</p>
          </div>
          <div class="quick-filters">
            <button class="btn btn-secondary btn-sm" onclick="triggerAutopilotCycle()">⚡ Run Cycle</button>
            <button class="btn btn-primary btn-sm" onclick="discoverBatchLeads()">🔍 Discover +3 Leads</button>
          </div>
        </div>
        <div class="kanban-board" id="kanban-board">
          <!-- Rendered by JS -->
        </div>
      </section>

      <!-- 2. LEAD ROSTER & SCORING VIEW -->
      <section id="view-list" class="view-panel">
        <div class="view-header">
          <div>
            <h2>Enriched Prospect Roster & Scoring</h2>
            <p class="subtitle">Multi-dimensional scoring against target customer profile, buying signals, and authority level.</p>
          </div>
          <div class="table-filters-bar">
            <input type="text" id="lead-search" placeholder="Search company, contact, stack..." oninput="handleSearch(this.value)">
            <select id="stage-filter" onchange="handleStageFilter(this.value)">
              <option value="all">All Deal Stages</option>
              <option value="discovered">Discovered</option>
              <option value="scored">Scored</option>
              <option value="outreach_drafted">Outreach Drafted</option>
              <option value="in_sequence">In Sequence</option>
              <option value="replied">Replied</option>
              <option value="meeting_booked">Meeting Booked</option>
              <option value="follow_up_scheduled">Follow-up Scheduled</option>
              <option value="opted_out">Opted Out</option>
              <option value="disqualified">Disqualified</option>
            </select>
            <select id="priority-filter" onchange="handlePriorityFilter(this.value)">
              <option value="all">All Priorities</option>
              <option value="high">High Priority (80+)</option>
              <option value="medium">Medium Priority (55-79)</option>
              <option value="low">Low Priority (&lt;55)</option>
            </select>
          </div>
        </div>
        <div class="table-container">
          <table class="leads-table">
            <thead>
              <tr>
                <th>Company & Contact</th>
                <th>Industry & Headcount</th>
                <th>Buying Signals & RFP Triggers</th>
                <th>AI Score Breakdown</th>
                <th>CRM Stage</th>
                <th>Next Action / Follow-up</th>
                <th>Actions</th>
              </tr>
            </thead>
            <tbody id="leads-table-body">
              <!-- Rendered by JS -->
            </tbody>
          </table>
        </div>
      </section>

      <!-- 3. HUMAN-IN-THE-LOOP REVIEW QUEUE -->
      <section id="view-review" class="view-panel">
        <div class="view-header">
          <div>
            <h2>Human-in-the-Loop Review Queue</h2>
            <p class="subtitle">"Automate the Work, Not the Relationship." Verify and personalize AI-crafted outreach before it leaves your outbox.</p>
          </div>
          <div style="font-size:0.8rem; color:#9ca3af;">
            Reviewing as: <strong id="review-exec-name" style="color:#60a5fa;">Alex Morgan (Sales Exec)</strong>
          </div>
        </div>
        <div class="review-container" id="review-list">
          <!-- Rendered by JS -->
        </div>
      </section>

      <!-- 4. RESPONSE INTELLIGENCE SIMULATOR (SHOWCASE SANDBOX) -->
      <section id="view-simulator" class="view-panel">
        <div class="view-header">
          <div>
            <h2>Response Intelligence Simulator</h2>
            <p class="subtitle">Experience how the autopilot interprets messy prospect replies, extracts temporal intent, and updates CRM state automatically.</p>
          </div>
        </div>

        <div class="simulator-layout">
          <!-- Left: Input & Sandbox -->
          <div class="sim-card sim-input-panel">
            <div class="sim-header">
              <span class="badge badge-purple">Step 1: Test Inbound Reply</span>
              <h3>Choose Prospect & Reply Scenario</h3>
              <p>Pick a prospect and either select a realistic test reply or write your own custom email reply.</p>
            </div>

            <div class="form-group">
              <label>1. Select Prospect From Active Pipeline:</label>
              <select id="sim-lead-select" class="form-control" onchange="updateSimLeadContext()">
                <!-- Populated by JS -->
              </select>
            </div>

            <div class="lead-context-card" id="sim-lead-context">
              <!-- Lead Snapshot -->
            </div>

            <div class="form-group">
              <label>2. Choose Preset Test Scenario:</label>
              <div class="preset-chips">
                <button class="chip" onclick="setPresetReply('delay')">⏰ "Contact me next month"</button>
                <button class="chip" onclick="setPresetReply('meeting')">📅 "Let's do a demo Thursday"</button>
                <button class="chip" onclick="setPresetReply('referral')">👤 "Talk to Sarah Chen, VP of IT"</button>
                <button class="chip" onclick="setPresetReply('objection')">💰 "Budget frozen until Q4"</button>
                <button class="chip" onclick="setPresetReply('info')">📄 "Send over security specs & pricing"</button>
                <button class="chip" onclick="setPresetReply('optout')">🚫 "Unsubscribe and remove us"</button>
              </div>
            </div>

            <div class="form-group">
              <label>Prospect's Incoming Message:</label>
              <textarea id="sim-reply-text" class="form-control" rows="4" placeholder="Enter or paste an email reply from the prospect..."></textarea>
            </div>

            <button class="btn btn-primary btn-block btn-lg" onclick="executeReplySimulation()">
              <span>⚡ Run AI Response Intelligence</span>
            </button>
          </div>

          <!-- Right: Autopilot Output -->
          <div class="sim-card sim-output-panel">
            <div class="sim-header">
              <span class="badge badge-blue">Step 2: Autopilot Decision Engine</span>
              <h3>Autonomous Classification & CRM Synchronization</h3>
              <p>Live output showing intent analysis, calendar calculation, and stage transitions.</p>
            </div>

            <div id="sim-result-container" class="sim-placeholder">
              <div class="empty-state-icon">🤖</div>
              <h4>Waiting for reply input...</h4>
              <p>Select a scenario on the left and click "Run AI Response Intelligence" to watch the autonomous engine adapt what happens next.</p>
            </div>
          </div>
        </div>
      </section>

      <!-- 5. MULTI-TOUCH SEQUENCE CADENCE -->
      <section id="view-cadence" class="view-panel">
        <div class="view-header">
          <div>
            <h2>Multi-Touch Sequence Cadence</h2>
            <p class="subtitle">Autonomous follow-up sequences. Outreach adapts to the prospect's tech stack and buying signals at every touchpoint.</p>
          </div>
        </div>

        <div class="cadence-container" id="cadence-steps-container">
          <!-- Rendered by JS -->
        </div>
      </section>

      <!-- 6. ICP RULES & SCORING WEIGHTS -->
      <section id="view-icp" class="view-panel">
        <div class="view-header">
          <div>
            <h2>Ideal Customer Profile (ICP) & Scoring Rules</h2>
            <p class="subtitle">Configure target criteria, buying signal triggers, and algorithm weights used by the autonomous scorer.</p>
          </div>
        </div>

        <div class="icp-settings-grid">
          <div class="settings-card">
            <h3>Algorithm Scoring Weights</h3>
            <p class="settings-desc">Adjust the weighting balance of the 0–100 suitability rating.</p>

            <div class="slider-group">
              <div class="slider-label">
                <span>ICP Fit (Firmographics & Size)</span>
                <span id="label-icp-weight">40%</span>
              </div>
              <input type="range" id="input-icp-weight" min="10" max="70" value="40" oninput="updateWeightLabels()">
            </div>

            <div class="slider-group">
              <div class="slider-label">
                <span>Buying Signals & Urgency</span>
                <span id="label-urgency-weight">35%</span>
              </div>
              <input type="range" id="input-urgency-weight" min="10" max="70" value="35" oninput="updateWeightLabels()">
            </div>

            <div class="slider-group">
              <div class="slider-label">
                <span>Decision Maker Authority</span>
                <span id="label-authority-weight">25%</span>
              </div>
              <input type="range" id="input-authority-weight" min="10" max="70" value="25" oninput="updateWeightLabels()">
            </div>

            <div class="form-group" style="margin-top:16px;">
              <label>Default Outreach Tone</label>
              <select id="input-default-tone" class="form-control">
                <option value="consultative">Consultative & Engineering Problem-Solving</option>
                <option value="executive">Executive Direct (CapEx & SLA Focus)</option>
                <option value="challenger">Challenger (Questioning Status Quo & Latency)</option>
                <option value="casual">Casual Founder-to-Founder</option>
              </select>
            </div>

            <button class="btn btn-primary" onclick="saveIcpSettings()">Save Scoring Configuration</button>
          </div>

          <div class="settings-card">
            <h3>Target Customer Profile Criteria</h3>
            <p class="settings-desc">Define what constitutes an ideal customer for automated market discovery.</p>

            <div class="form-group">
              <label>Target Industries</label>
              <input type="text" id="input-target-industries" class="form-control" value="Software, AI Infrastructure, Cloud SaaS, Biotech, Cybersecurity, Fintech">
            </div>

            <div class="form-group">
              <label>Ideal Company Size Range</label>
              <input type="text" id="input-target-size" class="form-control" value="50-500 employees (Sweet Spot)">
            </div>

            <div class="form-group">
              <label>Value Proposition Pitch</label>
              <textarea id="input-target-pitch" class="form-control" rows="3">Next-Gen Enterprise High-Performance Laptops & Fleet Lifecycle Management with Zero-Touch Provisioning.</textarea>
            </div>

            <button class="btn btn-secondary" onclick="saveIcpSettings()">Update ICP Profile</button>
          </div>
        </div>
      </section>

      <!-- 7. COMPLIANCE & DNC LIST -->
      <section id="view-safeguards" class="view-panel">
        <div class="view-header">
          <div>
            <h2>Compliance, Consent & Do-Not-Contact (DNC) List</h2>
            <p class="subtitle">Guaranteed compliance safeguards: Automatic opt-out handling, duplicate prevention, and consent suppression.</p>
          </div>
        </div>

        <div class="safeguards-layout">
          <div class="settings-card">
            <h3>Add Email to Suppression List</h3>
            <p class="settings-desc">Manually suppress any contact or domain to permanently block outreach.</p>
            <form onsubmit="submitManualSuppression(event)">
              <div class="form-group">
                <label>Email Address *</label>
                <input type="email" id="suppress-email" class="form-control" placeholder="prospect@company.com" required>
              </div>
              <div class="form-group">
                <label>Opt-Out Reason</label>
                <input type="text" id="suppress-reason" class="form-control" placeholder="e.g. Expressed Do-Not-Contact via telephone or GDPR request">
              </div>
              <button type="submit" class="btn btn-danger">🚫 Add to Do-Not-Contact List</button>
            </form>
          </div>

          <div class="settings-card">
            <h3>Active Do-Not-Contact Registry</h3>
            <p class="settings-desc">Contacts currently protected by suppression safeguards.</p>
            <div class="table-container" style="max-height:300px; overflow-y:auto;">
              <table class="leads-table">
                <thead>
                  <tr>
                    <th>Company & Email</th>
                    <th>Reason</th>
                    <th>Date</th>
                    <th>Status</th>
                  </tr>
                </thead>
                <tbody id="suppression-table-body">
                  <!-- Rendered by JS -->
                </tbody>
              </table>
            </div>
          </div>
        </div>
      </section>

      <!-- 8. AUTOPILOT AUDIT LOGS -->
      <section id="view-activity" class="view-panel">
        <div class="view-header">
          <div>
            <h2>Autopilot Audit Trail & Event Stream</h2>
            <p class="subtitle">Complete chronological record of autonomous lead enrichment, scoring decisions, drafts, and responses.</p>
          </div>
        </div>
        <div class="timeline-container" id="timeline-stream">
          <!-- Rendered by JS -->
        </div>
      </section>

    </main>
  </div>

  <!-- USER LOGIN / SWITCH ROLE MODAL -->
  <div class="modal" id="login-modal">
    <div class="modal-backdrop" onclick="closeLoginModal()"></div>
    <div class="modal-dialog">
      <div class="modal-header">
        <div>
          <h2>User Login & Role Switcher</h2>
          <p class="subtitle">Select your active role to manage requirements or execute sales pipeline.</p>
        </div>
        <button class="modal-close" onclick="closeLoginModal()">✕</button>
      </div>
      <div class="modal-body">
        <div style="margin-bottom:18px;">
          <label style="font-size:0.75rem; text-transform:uppercase; color:#9ca3af; font-weight:700;">1-Click Role Switch (Demo Mode):</label>
          <div class="role-selector-cards" style="margin-top:8px;">
            <div class="role-card" onclick="switchUserRole('client')">
              <div class="role-card-icon">👩‍💼</div>
              <div class="role-card-body">
                <div class="role-card-title">Client / Business Owner</div>
                <div class="role-card-name">Sarah Jenkins (Apex Enterprise)</div>
                <div class="role-card-desc">Defines product, requirements, target ICP & directs sales executive.</div>
              </div>
            </div>

            <div class="role-card" onclick="switchUserRole('sales_executive')">
              <div class="role-card-icon">👨‍💼</div>
              <div class="role-card-body">
                <div class="role-card-title">Dedicated Sales Executive</div>
                <div class="role-card-name">Alex Morgan (Senior AE)</div>
                <div class="role-card-desc">Executes outreach, approves drafts & closes deals for client.</div>
              </div>
            </div>
          </div>
        </div>

        <div style="border-top:1px solid var(--border-subtle); padding-top:16px;">
          <label style="font-size:0.75rem; text-transform:uppercase; color:#9ca3af; font-weight:700;">Or Login with Credentials:</label>
          <form onsubmit="handleManualLogin(event)" style="margin-top:8px;">
            <div class="form-group">
              <label>Email Address</label>
              <input type="email" id="login-email" class="form-control" placeholder="client@apextech.io" value="client@apextech.io" required>
            </div>
            <div class="form-group">
              <label>Password</label>
              <input type="password" id="login-password" class="form-control" value="password123" required>
            </div>
            <button type="submit" class="btn btn-primary btn-block">Sign In</button>
          </form>
        </div>
      </div>
    </div>
  </div>

  <!-- LEAD DOSSIER MODAL -->
  <div class="modal" id="lead-modal">
    <div class="modal-backdrop" onclick="closeLeadModal()"></div>
    <div class="modal-dialog modal-lg">
      <div class="modal-header">
        <div>
          <span class="badge" id="modal-stage-badge">Stage</span>
          <h2 id="modal-company-name">Company Name</h2>
          <p id="modal-contact-details" class="subtitle">Contact Name • Title</p>
        </div>
        <button class="modal-close" onclick="closeLeadModal()">✕</button>
      </div>
      <div class="modal-body" id="modal-lead-body">
        <!-- Rendered by JS -->
      </div>
      <div class="modal-footer" id="modal-actions-footer">
        <!-- Action Buttons -->
      </div>
    </div>
  </div>

  <!-- ADD PROSPECT MODAL -->
  <div class="modal" id="add-lead-modal">
    <div class="modal-backdrop" onclick="closeAddLeadModal()"></div>
    <div class="modal-dialog">
      <div class="modal-header">
        <div>
          <h2>Add & Enrich Prospect</h2>
          <p class="subtitle">Enter prospect profile for immediate AI ICP scoring and personalized outreach drafting.</p>
        </div>
        <button class="modal-close" onclick="closeAddLeadModal()">✕</button>
      </div>
      <div class="modal-body">
        <form id="add-lead-form" onsubmit="submitNewLead(event)">
          <div class="form-row">
            <div class="form-group">
              <label>Company Name *</label>
              <input type="text" id="add-company" class="form-control" placeholder="e.g. Acme Cloud Corp" required>
            </div>
            <div class="form-group">
              <label>Industry *</label>
              <input type="text" id="add-industry" class="form-control" placeholder="e.g. Enterprise SaaS / AI" required>
            </div>
          </div>
          <div class="form-row">
            <div class="form-group">
              <label>Contact Full Name *</label>
              <input type="text" id="add-contact" class="form-control" placeholder="e.g. Jordan Smith" required>
            </div>
            <div class="form-group">
              <label>Job Title *</label>
              <input type="text" id="add-title" class="form-control" placeholder="e.g. VP of Engineering" required>
            </div>
          </div>
          <div class="form-row">
            <div class="form-group">
              <label>Work Email *</label>
              <input type="email" id="add-email" class="form-control" placeholder="jordan@acme.io" required>
            </div>
            <div class="form-group">
              <label>Company Size *</label>
              <select id="add-size" class="form-control" required>
                <option value="150-200 employees">150-200 employees (Ideal Tier 1)</option>
                <option value="80-120 employees">80-120 employees (Strong Tier 2)</option>
                <option value="300-500 employees">300-500 employees (Mid-Enterprise)</option>
                <option value="1,000+ employees">1,000+ employees (Enterprise Fleet)</option>
                <option value="10-15 employees">10-15 employees (SMB / Low ICP)</option>
              </select>
            </div>
          </div>
          <div class="form-group">
            <label>Tech Stack & Environment</label>
            <input type="text" id="add-tech" class="form-control" placeholder="e.g. macOS M3, AWS, Linux, Slack, Jamf">
          </div>
          <div class="form-group">
            <label>Buying Signals & Triggers</label>
            <textarea id="add-signals" class="form-control" rows="2" placeholder="e.g. Hiring 30 engineers this quarter; posted RFP for laptop refresh."></textarea>
          </div>
          <div class="modal-footer" style="padding-left:0; padding-right:0; margin-bottom:0;">
            <button type="button" class="btn btn-ghost" onclick="closeAddLeadModal()">Cancel</button>
            <button type="submit" class="btn btn-primary">Score & Add to Pipeline</button>
          </div>
        </form>
      </div>
    </div>
  </div>

  <div id="toast-container" class="toast-container"></div>
  <script src="/static/js/app.js"></script>
</body>
</html>
```

---

## 📄 `static/css/style.css`
**Description**: Custom Modern SaaS Design System & CSS Styles  
**File Path**: `/Users/kruthikaran/.gemini/antigravity/scratch/sales-pipeline-autopilot/static/css/style.css`

```css
:root {
  --bg-primary: #0a0e17;
  --bg-sidebar: #0f1523;
  --bg-surface: #131b2e;
  --bg-surface-elevated: #1a243d;
  --bg-surface-hover: #222f4f;
  --border-subtle: #212d45;
  --border-active: #3b82f6;

  --text-main: #f8fafc;
  --text-muted: #94a3b8;
  --text-faint: #64748b;

  --accent-blue: #3b82f6;
  --accent-cyan: #06b6d4;
  --accent-emerald: #10b981;
  --accent-amber: #f59e0b;
  --accent-purple: #8b5cf6;
  --accent-rose: #ef4444;

  --font-sans: 'Plus Jakarta Sans', -apple-system, BlinkMacSystemFont, sans-serif;
  --font-mono: 'JetBrains Mono', monospace;
  --radius-sm: 6px;
  --radius-md: 10px;
  --radius-lg: 14px;
  --shadow-sm: 0 2px 4px rgba(0,0,0,0.25);
  --shadow-md: 0 4px 12px rgba(0,0,0,0.35);
  --shadow-lg: 0 10px 25px rgba(0,0,0,0.5);
}

* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

body.app-layout {
  font-family: var(--font-sans);
  background-color: var(--bg-primary);
  color: var(--text-main);
  line-height: 1.5;
  -webkit-font-smoothing: antialiased;
  display: flex;
  height: 100vh;
  width: 100vw;
  overflow: hidden;
}

/* -------------------------------------------------------------
   LEFT SIDEBAR (OPTIONS VERTICALLY DOWNWARDS)
------------------------------------------------------------- */
.sidebar {
  width: 290px;
  background-color: var(--bg-sidebar);
  border-right: 1px solid var(--border-subtle);
  display: flex;
  flex-direction: column;
  flex-shrink: 0;
  height: 100vh;
  overflow-y: auto;
  z-index: 40;
}

.sidebar-brand {
  padding: 20px 20px 16px 20px;
  display: flex;
  align-items: center;
  gap: 12px;
  border-bottom: 1px solid var(--border-subtle);
}

.logo-icon {
  width: 36px;
  height: 36px;
  border-radius: 9px;
  background: linear-gradient(135deg, #3b82f6, #8b5cf6);
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 18px;
  box-shadow: 0 0 12px rgba(59, 130, 246, 0.4);
}

.brand-text {
  display: flex;
  flex-direction: column;
}

.brand-title {
  font-size: 1.05rem;
  font-weight: 800;
  letter-spacing: -0.02em;
}

.gradient-text {
  background: linear-gradient(90deg, #60a5fa, #c084fc);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  font-weight: 700;
  font-size: 0.78rem;
}

.sidebar-status-box {
  margin: 14px 16px;
  padding: 10px 12px;
  background: rgba(16, 185, 129, 0.08);
  border: 1px solid rgba(16, 185, 129, 0.25);
  border-radius: var(--radius-sm);
}

.status-top {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 0.78rem;
  font-weight: 700;
  color: #34d399;
}

.pulse-dot {
  width: 7px;
  height: 7px;
  border-radius: 50%;
  background-color: #10b981;
  box-shadow: 0 0 8px #10b981;
  animation: pulse 2s infinite;
}

@keyframes pulse {
  0% { transform: scale(0.95); box-shadow: 0 0 0 0 rgba(16, 185, 129, 0.7); }
  70% { transform: scale(1); box-shadow: 0 0 0 6px rgba(16, 185, 129, 0); }
  100% { transform: scale(0.95); box-shadow: 0 0 0 0 rgba(16, 185, 129, 0); }
}

.status-sub {
  font-size: 0.68rem;
  color: var(--text-muted);
  margin-top: 3px;
}

/* SIDEBAR NAVIGATION ITEMS (DOWNWARDS) */
.sidebar-nav {
  display: flex;
  flex-direction: column;
  gap: 3px;
  padding: 0 12px;
  flex: 1;
}

.nav-section-label {
  font-size: 0.68rem;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.06em;
  color: var(--text-faint);
  padding: 12px 10px 4px 10px;
}

.nav-item {
  display: flex;
  align-items: center;
  gap: 10px;
  width: 100%;
  padding: 9px 12px;
  border-radius: var(--radius-sm);
  background: transparent;
  border: none;
  color: var(--text-muted);
  font-family: var(--font-sans);
  font-size: 0.83rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.15s ease;
  text-align: left;
}

.nav-item:hover {
  background: var(--bg-surface-elevated);
  color: var(--text-main);
}

.nav-item.active {
  background: rgba(59, 130, 246, 0.15);
  color: #60a5fa;
  border-left: 3px solid #3b82f6;
  font-weight: 700;
}

.nav-item.highlight-item.active {
  background: rgba(139, 92, 246, 0.15);
  color: #c084fc;
  border-left-color: #a855f7;
}

.nav-icon {
  font-size: 1.05rem;
  width: 20px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.nav-label {
  flex: 1;
}

.nav-badge {
  font-size: 0.68rem;
  padding: 1px 6px;
  border-radius: 9999px;
  font-weight: 700;
}

/* SIDEBAR BOTTOM ACTIONS */
.sidebar-bottom-actions {
  padding: 12px;
  border-top: 1px solid var(--border-subtle);
  display: flex;
  flex-direction: column;
  gap: 6px;
  background: rgba(10, 14, 23, 0.4);
}

.sidebar-action-btn {
  display: flex;
  align-items: center;
  gap: 10px;
  width: 100%;
  padding: 8px 10px;
  background: var(--bg-surface-elevated);
  border: 1px solid var(--border-subtle);
  border-radius: var(--radius-sm);
  color: var(--text-main);
  cursor: pointer;
  transition: all 0.15s ease;
  text-align: left;
}

.sidebar-action-btn:hover {
  background: var(--bg-surface-hover);
  border-color: #475569;
}

.btn-autopilot-run {
  background: linear-gradient(135deg, rgba(37, 99, 235, 0.25), rgba(139, 92, 246, 0.25));
  border-color: rgba(59, 130, 246, 0.4);
}

.btn-autopilot-run:hover {
  background: linear-gradient(135deg, rgba(37, 99, 235, 0.4), rgba(139, 92, 246, 0.4));
  border-color: #60a5fa;
}

.action-icon {
  font-size: 1.1rem;
}

.action-text {
  display: flex;
  flex-direction: column;
}

.action-title {
  font-size: 0.78rem;
  font-weight: 700;
}

.action-desc {
  font-size: 0.68rem;
  color: var(--text-muted);
}

.btn-reset {
  opacity: 0.7;
}
.btn-reset:hover { opacity: 1; }

/* -------------------------------------------------------------
   MAIN APP AREA (RIGHT OF SIDEBAR)
------------------------------------------------------------- */
.app-main {
  flex: 1;
  display: flex;
  flex-direction: column;
  height: 100vh;
  overflow: hidden;
  min-width: 0;
}

.top-header {
  background: var(--bg-surface);
  border-bottom: 1px solid var(--border-subtle);
  padding: 12px 24px;
  flex-shrink: 0;
}

.header-banner {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(130px, 1fr));
  gap: 10px;
}

.stat-card {
  background: var(--bg-sidebar);
  border: 1px solid var(--border-subtle);
  border-radius: var(--radius-sm);
  padding: 10px 12px;
  display: flex;
  flex-direction: column;
}

.stat-card:hover {
  border-color: #3b82f6;
}

.stat-card.highlight {
  background: linear-gradient(145deg, #131b2e, #1c2642);
  border-color: rgba(99, 102, 241, 0.4);
}

.stat-card.alert-stat {
  cursor: pointer;
  border-color: rgba(245, 158, 11, 0.3);
}
.stat-card.alert-stat:hover { border-color: var(--accent-amber); }

.stat-card.safeguard-stat {
  cursor: pointer;
  border-color: rgba(6, 182, 212, 0.3);
}
.stat-card.safeguard-stat:hover { border-color: var(--accent-cyan); }

.stat-label {
  font-size: 0.68rem;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.04em;
  color: var(--text-muted);
}

.stat-value {
  font-size: 1.35rem;
  font-weight: 800;
  letter-spacing: -0.02em;
  color: var(--text-main);
  margin-top: 2px;
}

.stat-trend {
  font-size: 0.68rem;
  font-weight: 600;
  margin-top: 2px;
}
.stat-trend.positive { color: #34d399; }

.stat-desc {
  font-size: 0.68rem;
  color: var(--text-faint);
  margin-top: 2px;
}

.text-amber { color: var(--accent-amber) !important; }
.text-teal { color: var(--accent-cyan) !important; }

/* -------------------------------------------------------------
   CONTENT BODY & PANELS
------------------------------------------------------------- */
.content-body {
  flex: 1;
  padding: 20px 24px;
  overflow-y: auto;
}

.view-panel {
  display: none;
}
.view-panel.active {
  display: block;
}

.view-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: 18px;
}

.view-header h2 {
  font-size: 1.3rem;
  font-weight: 800;
  letter-spacing: -0.01em;
}

.subtitle {
  font-size: 0.82rem;
  color: var(--text-muted);
  margin-top: 2px;
}

.quick-filters {
  display: flex;
  gap: 8px;
}

.table-filters-bar {
  display: flex;
  gap: 8px;
  align-items: center;
}

.table-filters-bar input,
.table-filters-bar select {
  background: var(--bg-surface-elevated);
  border: 1px solid var(--border-subtle);
  color: var(--text-main);
  padding: 6px 12px;
  border-radius: var(--radius-sm);
  font-size: 0.8rem;
  font-family: var(--font-sans);
}

/* -------------------------------------------------------------
   KANBAN BOARD
------------------------------------------------------------- */
.kanban-board {
  display: flex;
  gap: 12px;
  overflow-x: auto;
  padding-bottom: 14px;
  align-items: flex-start;
  min-height: calc(100vh - 210px);
}

.kanban-col {
  flex: 0 0 280px;
  background: var(--bg-surface);
  border: 1px solid var(--border-subtle);
  border-radius: var(--radius-md);
  display: flex;
  flex-direction: column;
  max-height: calc(100vh - 220px);
}

.kanban-col-header {
  padding: 10px 12px;
  border-bottom: 1px solid var(--border-subtle);
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.kanban-col-title {
  font-size: 0.78rem;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.04em;
  display: flex;
  align-items: center;
  gap: 6px;
}

.kanban-col-count {
  font-size: 0.7rem;
  padding: 2px 6px;
  border-radius: 9999px;
  background: var(--bg-surface-elevated);
  color: var(--text-muted);
  font-weight: 700;
}

.kanban-cards-list {
  padding: 8px;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
  gap: 8px;
  flex: 1;
}

.kanban-card {
  background: var(--bg-surface-elevated);
  border: 1px solid var(--border-subtle);
  border-radius: var(--radius-sm);
  padding: 10px 12px;
  cursor: pointer;
  transition: all 0.15s ease;
}

.kanban-card:hover {
  border-color: #3b82f6;
  transform: translateY(-2px);
  box-shadow: var(--shadow-md);
}

.card-top {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: 4px;
}

.card-company {
  font-size: 0.85rem;
  font-weight: 700;
  color: var(--text-main);
}

.card-contact {
  font-size: 0.72rem;
  color: var(--text-muted);
  margin-bottom: 6px;
}

.score-chip {
  font-family: var(--font-mono);
  font-size: 0.7rem;
  font-weight: 700;
  padding: 2px 6px;
  border-radius: 4px;
  display: inline-flex;
  align-items: center;
}

.score-high { background: rgba(16, 185, 129, 0.2); color: #34d399; }
.score-med { background: rgba(245, 158, 11, 0.2); color: #fbbf24; }
.score-low { background: rgba(239, 68, 68, 0.2); color: #f87171; }

.card-signal {
  font-size: 0.72rem;
  color: #94a3b8;
  background: rgba(10, 14, 23, 0.6);
  padding: 5px 8px;
  border-radius: 4px;
  margin-bottom: 6px;
  line-height: 1.35;
}

.card-footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-top: 6px;
  padding-top: 6px;
  border-top: 1px solid rgba(255, 255, 255, 0.05);
  font-size: 0.68rem;
  color: var(--text-faint);
}

.card-actions {
  display: flex;
  gap: 4px;
}

/* -------------------------------------------------------------
   TABLE VIEW
------------------------------------------------------------- */
.table-container {
  background: var(--bg-surface);
  border: 1px solid var(--border-subtle);
  border-radius: var(--radius-md);
  overflow-x: auto;
}

.leads-table {
  width: 100%;
  border-collapse: collapse;
  font-size: 0.82rem;
  text-align: left;
}

.leads-table th {
  padding: 10px 14px;
  background: var(--bg-surface-elevated);
  color: var(--text-muted);
  font-weight: 600;
  border-bottom: 1px solid var(--border-subtle);
}

.leads-table td {
  padding: 12px 14px;
  border-bottom: 1px solid var(--border-subtle);
  vertical-align: middle;
}

.leads-table tr:hover td {
  background: var(--bg-surface-hover);
}

/* -------------------------------------------------------------
   HUMAN REVIEW QUEUE
------------------------------------------------------------- */
.review-container {
  display: flex;
  flex-direction: column;
  gap: 14px;
}

.review-item-card {
  background: var(--bg-surface);
  border: 1px solid var(--border-subtle);
  border-radius: var(--radius-md);
  padding: 18px;
}

.review-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: 12px;
  padding-bottom: 10px;
  border-bottom: 1px solid var(--border-subtle);
}

.review-meta {
  display: flex;
  flex-direction: column;
  gap: 3px;
}

.review-company {
  font-size: 1rem;
  font-weight: 700;
}

.hook-tag {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  font-size: 0.73rem;
  color: #38bdf8;
  background: rgba(56, 189, 248, 0.1);
  padding: 2px 7px;
  border-radius: 4px;
  width: fit-content;
}

.review-subject {
  font-size: 0.88rem;
  font-weight: 700;
  margin-bottom: 8px;
  color: #f3f4f6;
}

.review-body {
  background: var(--bg-surface-elevated);
  border: 1px solid var(--border-subtle);
  border-radius: var(--radius-sm);
  padding: 12px;
  font-family: var(--font-sans);
  font-size: 0.83rem;
  white-space: pre-wrap;
  color: #d1d5db;
  line-height: 1.6;
}

.review-actions {
  display: flex;
  justify-content: flex-end;
  gap: 8px;
  margin-top: 12px;
}

/* -------------------------------------------------------------
   SIMULATOR LAYOUT
------------------------------------------------------------- */
.simulator-layout {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 18px;
}

@media (max-width: 980px) {
  .simulator-layout { grid-template-columns: 1fr; }
}

.sim-card {
  background: var(--bg-surface);
  border: 1px solid var(--border-subtle);
  border-radius: var(--radius-lg);
  padding: 20px;
}

.sim-header {
  margin-bottom: 14px;
}

.sim-header h3 {
  font-size: 1.15rem;
  font-weight: 700;
  margin-top: 4px;
  margin-bottom: 3px;
}

.sim-header p {
  font-size: 0.8rem;
  color: var(--text-muted);
}

.form-group {
  margin-bottom: 14px;
}

.form-group label {
  display: block;
  font-size: 0.74rem;
  font-weight: 700;
  color: var(--text-muted);
  margin-bottom: 5px;
  text-transform: uppercase;
  letter-spacing: 0.03em;
}

.form-control {
  width: 100%;
  background: var(--bg-surface-elevated);
  border: 1px solid var(--border-subtle);
  color: var(--text-main);
  padding: 8px 12px;
  border-radius: var(--radius-sm);
  font-family: var(--font-sans);
  font-size: 0.85rem;
}

.form-control:focus {
  outline: none;
  border-color: #3b82f6;
  box-shadow: 0 0 0 2px rgba(59, 130, 246, 0.2);
}

.preset-chips {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
}

.chip {
  background: var(--bg-surface-elevated);
  border: 1px solid var(--border-subtle);
  color: var(--text-muted);
  font-family: var(--font-sans);
  font-size: 0.72rem;
  font-weight: 600;
  padding: 4px 9px;
  border-radius: 9999px;
  cursor: pointer;
  transition: all 0.15s ease;
}

.chip:hover {
  background: var(--bg-surface-hover);
  border-color: #60a5fa;
  color: var(--text-main);
}

.sim-placeholder {
  text-align: center;
  padding: 60px 20px;
  color: var(--text-muted);
}

.empty-state-icon {
  font-size: 38px;
  margin-bottom: 10px;
}

.sim-result-box {
  display: flex;
  flex-direction: column;
  gap: 14px;
  animation: fadeIn 0.3s ease;
}

@keyframes fadeIn {
  from { opacity: 0; transform: translateY(6px); }
  to { opacity: 1; transform: translateY(0); }
}

.result-metric-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 10px;
}

.res-card {
  background: var(--bg-surface-elevated);
  border: 1px solid var(--border-subtle);
  border-radius: var(--radius-sm);
  padding: 10px;
}

.res-label {
  font-size: 0.68rem;
  text-transform: uppercase;
  color: var(--text-muted);
  font-weight: 700;
}

.res-val {
  font-size: 1.05rem;
  font-weight: 700;
  margin-top: 2px;
}

.res-box-highlight {
  background: rgba(59, 130, 246, 0.08);
  border: 1px solid rgba(59, 130, 246, 0.3);
  border-radius: var(--radius-sm);
  padding: 12px;
}

.res-box-highlight.safeguard-alert {
  background: rgba(239, 68, 68, 0.1);
  border-color: rgba(239, 68, 68, 0.4);
}

.res-title {
  font-size: 0.75rem;
  font-weight: 700;
  text-transform: uppercase;
  margin-bottom: 4px;
  letter-spacing: 0.03em;
}

.stage-transition-pill {
  display: flex;
  align-items: center;
  gap: 6px;
  margin-top: 6px;
  font-size: 0.82rem;
  font-weight: 600;
}

/* -------------------------------------------------------------
   CADENCE BUILDER STYLES (NEW OPTION)
------------------------------------------------------------- */
.cadence-container {
  display: flex;
  flex-direction: column;
  gap: 14px;
}

.cadence-step-card {
  background: var(--bg-surface);
  border: 1px solid var(--border-subtle);
  border-radius: var(--radius-md);
  padding: 16px 20px;
  display: flex;
  align-items: center;
  gap: 18px;
  position: relative;
}

.cadence-step-number {
  width: 44px;
  height: 44px;
  border-radius: 12px;
  background: linear-gradient(135deg, #2563eb, #7c3aed);
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: 800;
  font-size: 1.1rem;
  flex-shrink: 0;
  box-shadow: 0 0 10px rgba(37, 99, 235, 0.3);
}

.cadence-step-body {
  flex: 1;
}

.cadence-step-title {
  font-size: 0.95rem;
  font-weight: 700;
  margin-bottom: 2px;
}

.cadence-step-purpose {
  font-size: 0.8rem;
  color: var(--text-muted);
}

.cadence-delay-badge {
  font-family: var(--font-mono);
  font-size: 0.74rem;
  padding: 4px 10px;
  border-radius: 9999px;
  background: var(--bg-surface-elevated);
  border: 1px solid var(--border-subtle);
  color: #60a5fa;
  font-weight: 600;
}

/* -------------------------------------------------------------
   ICP SETTINGS & SAFEGUARDS GRIDS (NEW OPTIONS)
------------------------------------------------------------- */
.icp-settings-grid,
.safeguards-layout {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 18px;
}

@media (max-width: 900px) {
  .icp-settings-grid,
  .safeguards-layout { grid-template-columns: 1fr; }
}

.settings-card {
  background: var(--bg-surface);
  border: 1px solid var(--border-subtle);
  border-radius: var(--radius-lg);
  padding: 20px;
}

.settings-card h3 {
  font-size: 1.05rem;
  font-weight: 700;
  margin-bottom: 2px;
}

.settings-desc {
  font-size: 0.78rem;
  color: var(--text-muted);
  margin-bottom: 16px;
}

.slider-group {
  margin-bottom: 14px;
}

.slider-label {
  display: flex;
  justify-content: space-between;
  font-size: 0.78rem;
  font-weight: 600;
  color: var(--text-main);
  margin-bottom: 4px;
}

input[type="range"] {
  width: 100%;
  accent-color: #3b82f6;
  cursor: pointer;
}

/* -------------------------------------------------------------
   BUTTONS & BADGES
------------------------------------------------------------- */
.btn {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  font-family: var(--font-sans);
  font-size: 0.82rem;
  font-weight: 600;
  padding: 7px 12px;
  border-radius: var(--radius-sm);
  border: 1px solid transparent;
  cursor: pointer;
  transition: all 0.15s ease;
  text-decoration: none;
}

.btn-primary {
  background: linear-gradient(135deg, #2563eb, #1d4ed8);
  color: white;
  border-color: #3b82f6;
  box-shadow: 0 2px 6px rgba(37, 99, 235, 0.3);
}
.btn-primary:hover {
  background: linear-gradient(135deg, #3b82f6, #2563eb);
}

.btn-secondary {
  background: var(--bg-surface-elevated);
  color: var(--text-main);
  border-color: var(--border-subtle);
}
.btn-secondary:hover {
  background: var(--bg-surface-hover);
  border-color: #4b5563;
}

.btn-ghost {
  background: transparent;
  color: var(--text-muted);
}
.btn-ghost:hover {
  color: var(--text-main);
  background: rgba(255, 255, 255, 0.05);
}

.btn-danger {
  background: rgba(239, 68, 68, 0.15);
  color: #f87171;
  border-color: rgba(239, 68, 68, 0.3);
}
.btn-danger:hover { background: rgba(239, 68, 68, 0.25); }

.btn-success { background: #059669; color: white; }
.btn-success:hover { background: #10b981; }

.btn-block { width: 100%; justify-content: center; }
.btn-lg { padding: 10px 18px; font-size: 0.9rem; }
.btn-sm { padding: 4px 8px; font-size: 0.72rem; }

.badge {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  padding: 2px 7px;
  border-radius: 9999px;
  font-size: 0.68rem;
  font-weight: 700;
  letter-spacing: 0.02em;
}

.badge-blue { background: rgba(59, 130, 246, 0.15); color: #93c5fd; border: 1px solid rgba(59, 130, 246, 0.3); }
.badge-green { background: rgba(16, 185, 129, 0.15); color: #6ee7b7; border: 1px solid rgba(16, 185, 129, 0.3); }
.badge-amber { background: rgba(245, 158, 11, 0.15); color: #fcd34d; border: 1px solid rgba(245, 158, 11, 0.3); }
.badge-purple { background: rgba(139, 92, 246, 0.15); color: #c4b5fd; border: 1px solid rgba(139, 92, 246, 0.3); }
.badge-rose { background: rgba(239, 68, 68, 0.15); color: #fca5a5; border: 1px solid rgba(239, 68, 68, 0.3); }
.badge-teal { background: rgba(6, 182, 212, 0.15); color: #67e8f9; border: 1px solid rgba(6, 182, 212, 0.3); }
.badge-gray { background: rgba(156, 163, 175, 0.15); color: #d1d5db; border: 1px solid rgba(156, 163, 175, 0.3); }

/* -------------------------------------------------------------
   TIMELINE AUDIT
------------------------------------------------------------- */
.timeline-container {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.timeline-item {
  display: flex;
  align-items: flex-start;
  gap: 12px;
  padding: 12px 16px;
  background: var(--bg-surface);
  border: 1px solid var(--border-subtle);
  border-radius: var(--radius-md);
}

.timeline-icon-wrap {
  width: 30px;
  height: 30px;
  border-radius: 8px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 15px;
  flex-shrink: 0;
  background: var(--bg-surface-elevated);
}

.timeline-content { flex: 1; }

.timeline-title-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 3px;
}

.timeline-title { font-size: 0.85rem; font-weight: 700; }
.timeline-time { font-size: 0.7rem; font-family: var(--font-mono); color: var(--text-faint); }
.timeline-desc { font-size: 0.78rem; color: var(--text-muted); }

/* -------------------------------------------------------------
   MODALS
------------------------------------------------------------- */
.modal {
  position: fixed;
  inset: 0;
  z-index: 100;
  display: none;
  align-items: center;
  justify-content: center;
  padding: 20px;
}
.modal.active { display: flex; }

.modal-backdrop {
  position: absolute;
  inset: 0;
  background: rgba(0, 0, 0, 0.78);
  backdrop-filter: blur(4px);
}

.modal-dialog {
  position: relative;
  background: var(--bg-surface);
  border: 1px solid var(--border-subtle);
  border-radius: var(--radius-lg);
  width: 100%;
  max-width: 580px;
  box-shadow: var(--shadow-lg);
  overflow: hidden;
  max-height: 90vh;
  display: flex;
  flex-direction: column;
}

.modal-dialog.modal-lg { max-width: 820px; }

.modal-header {
  padding: 16px 20px;
  border-bottom: 1px solid var(--border-subtle);
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
}

.modal-close {
  background: transparent;
  border: none;
  color: var(--text-muted);
  font-size: 1.2rem;
  cursor: pointer;
}
.modal-close:hover { color: var(--text-main); }

.modal-body {
  padding: 18px 20px;
  overflow-y: auto;
  flex: 1;
}

.modal-footer {
  padding: 12px 20px;
  border-top: 1px solid var(--border-subtle);
  display: flex;
  justify-content: flex-end;
  gap: 10px;
  background: var(--bg-surface-elevated);
}

/* TOASTS */
.toast-container {
  position: fixed;
  bottom: 24px;
  right: 24px;
  z-index: 200;
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.toast {
  background: #1e293b;
  color: #f8fafc;
  padding: 10px 16px;
  border-radius: var(--radius-sm);
  border-left: 4px solid #3b82f6;
  box-shadow: var(--shadow-lg);
  font-size: 0.83rem;
  font-weight: 500;
  display: flex;
  align-items: center;
  gap: 8px;
  animation: slideIn 0.25s ease;
}

.toast.toast-success { border-left-color: #10b981; }
.toast.toast-warning { border-left-color: #f59e0b; }
.toast.toast-danger { border-left-color: #ef4444; }

@keyframes slideIn {
  from { transform: translateX(100%); opacity: 0; }
  to { transform: translateX(0); opacity: 1; }
}

/* -------------------------------------------------------------
   USER PROFILE CARD IN SIDEBAR (LOGIN & ROLE)
------------------------------------------------------------- */
.sidebar-user-card {
  margin: 12px 14px 4px 14px;
  padding: 10px 12px;
  background: var(--bg-surface-elevated);
  border: 1px solid var(--border-subtle);
  border-radius: var(--radius-md);
  display: flex;
  align-items: center;
  gap: 10px;
  cursor: pointer;
  transition: all 0.15s ease;
}

.sidebar-user-card:hover {
  border-color: #3b82f6;
  background: var(--bg-surface-hover);
}

.user-avatar {
  width: 38px;
  height: 38px;
  border-radius: 50%;
  background: rgba(59, 130, 246, 0.15);
  border: 1px solid rgba(59, 130, 246, 0.3);
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 20px;
  flex-shrink: 0;
}

.user-info {
  flex: 1;
  min-width: 0;
}

.user-name-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 4px;
}

.user-name {
  font-size: 0.82rem;
  font-weight: 700;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.role-pill {
  font-size: 0.65rem;
  font-weight: 700;
  text-transform: uppercase;
  padding: 1px 6px;
  border-radius: 9999px;
  background: rgba(6, 182, 212, 0.15);
  color: #67e8f9;
  border: 1px solid rgba(6, 182, 212, 0.3);
}

.role-pill.role-sales-exec {
  background: rgba(139, 92, 246, 0.15);
  color: #c4b5fd;
  border-color: rgba(139, 92, 246, 0.3);
}

.user-sub {
  font-size: 0.7rem;
  color: var(--text-muted);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.user-counterpart {
  font-size: 0.66rem;
  color: #60a5fa;
  margin-top: 2px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.user-switch-icon {
  font-size: 1rem;
  color: var(--text-muted);
}
.sidebar-user-card:hover .user-switch-icon {
  color: #60a5fa;
}

/* -------------------------------------------------------------
   CLIENT REQUIREMENTS & SALES EXECUTIVE HUB
------------------------------------------------------------- */
.requirements-layout {
  display: grid;
  grid-template-columns: 1.25fr 1fr;
  gap: 20px;
}

@media (max-width: 980px) {
  .requirements-layout { grid-template-columns: 1fr; }
}

.req-form-card {
  padding: 22px;
}

.req-exec-card {
  padding: 22px;
  display: flex;
  flex-direction: column;
}

.exec-profile-box {
  display: flex;
  align-items: center;
  gap: 14px;
  padding: 16px;
  background: var(--bg-surface-elevated);
  border: 1px solid var(--border-subtle);
  border-radius: var(--radius-md);
  margin-bottom: 16px;
}

.exec-avatar-lg {
  width: 52px;
  height: 52px;
  border-radius: 50%;
  background: rgba(16, 185, 129, 0.15);
  border: 1px solid rgba(16, 185, 129, 0.35);
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 28px;
  flex-shrink: 0;
}

.exec-details h4 {
  font-size: 1.05rem;
  font-weight: 700;
  color: var(--text-main);
}

.exec-title {
  font-size: 0.78rem;
  color: #94a3b8;
}

.exec-email {
  font-size: 0.74rem;
  color: #60a5fa;
  margin-top: 2px;
}

.exec-stats-grid {
  display: grid;
  grid-template-columns: 1fr 1fr 1fr;
  gap: 10px;
}

.mini-stat {
  background: var(--bg-surface-elevated);
  border: 1px solid var(--border-subtle);
  border-radius: var(--radius-sm);
  padding: 10px;
  text-align: center;
}

.mini-stat-label {
  font-size: 0.65rem;
  font-weight: 700;
  text-transform: uppercase;
  color: var(--text-muted);
}

.mini-stat-val {
  font-size: 1.15rem;
  font-weight: 800;
  margin-top: 2px;
}

.text-green { color: #34d399 !important; }
.text-blue { color: #60a5fa !important; }

/* -------------------------------------------------------------
   ROLE SELECTOR CARDS (LOGIN MODAL)
------------------------------------------------------------- */
.role-selector-cards {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.role-card {
  display: flex;
  align-items: center;
  gap: 14px;
  padding: 14px 16px;
  background: var(--bg-surface-elevated);
  border: 1px solid var(--border-subtle);
  border-radius: var(--radius-md);
  cursor: pointer;
  transition: all 0.15s ease;
}

.role-card:hover {
  border-color: #3b82f6;
  background: var(--bg-surface-hover);
  transform: translateY(-1px);
}

.role-card-icon {
  font-size: 28px;
}

.role-card-body {
  flex: 1;
}

.role-card-title {
  font-size: 0.88rem;
  font-weight: 800;
  color: var(--text-main);
}

.role-card-name {
  font-size: 0.78rem;
  color: #60a5fa;
  font-weight: 600;
}

.role-card-desc {
  font-size: 0.72rem;
  color: var(--text-muted);
  margin-top: 2px;
}
```

---

## 📄 `static/js/app.js`
**Description**: Frontend Single-Page-App Client Logic  
**File Path**: `/Users/kruthikaran/.gemini/antigravity/scratch/sales-pipeline-autopilot/static/js/app.js`

```javascript
// Sales Pipeline Autopilot - Client App Engine v2.0 with Auth & Client Hub

let currentLeads = [];
let selectedSimLeadId = null;
let currentSearch = "";
let currentFilterStage = "all";
let currentFilterPriority = "all";
let currentUser = null;

document.addEventListener("DOMContentLoaded", () => {
  initApp();
});

async function initApp() {
  await Promise.all([
    fetchCurrentUser(),
    fetchStats(),
    fetchLeads(),
    fetchPendingReview(),
    fetchActivities(),
    loadCadenceSteps(),
    loadIcpSettings(),
    loadSuppressionList(),
    loadClientRequirements()
  ]);
}

// -------------------------------------------------------------
// USER AUTHENTICATION & ROLE MANAGEMENT
// -------------------------------------------------------------
async function fetchCurrentUser() {
  try {
    const res = await fetch("/api/auth/current-user");
    currentUser = await res.json();
    renderUserCard(currentUser);
  } catch (err) {
    console.error("Failed to load user:", err);
  }
}

function renderUserCard(user) {
  if (!user) return;

  const avatar = document.getElementById("sidebar-user-avatar");
  const name = document.getElementById("sidebar-user-name");
  const rolePill = document.getElementById("sidebar-user-role-pill");
  const company = document.getElementById("sidebar-user-company");
  const counterpartText = document.getElementById("sidebar-counterpart-text");

  if (avatar) avatar.innerText = user.avatar || (user.role === "client" ? "👩‍💼" : "👨‍💼");
  if (name) name.innerText = user.name;
  if (company) company.innerText = user.company_name || "Apex Enterprise Systems";

  if (rolePill) {
    if (user.role === "client") {
      rolePill.className = "role-pill";
      rolePill.innerText = "Client / Owner";
    } else {
      rolePill.className = "role-pill role-sales-exec";
      rolePill.innerText = "Sales Exec";
    }
  }

  if (counterpartText && user.counterpart) {
    if (user.role === "client") {
      counterpartText.innerHTML = `Sales Exec: <strong>${escapeHtml(user.counterpart.name)}</strong>`;
    } else {
      counterpartText.innerHTML = `Client: <strong>${escapeHtml(user.counterpart.name)}</strong>`;
    }
  }

  // Update Review Queue header context
  const reviewExec = document.getElementById("review-exec-name");
  if (reviewExec) {
    if (user.role === "sales_executive") {
      reviewExec.innerText = `${user.name} (Assigned Sales Exec)`;
    } else {
      reviewExec.innerText = `${user.counterpart ? user.counterpart.name : 'Alex Morgan'} (Assigned Sales Exec)`;
    }
  }
}

function openLoginModal() {
  document.getElementById("login-modal").classList.add("active");
}

function closeLoginModal() {
  document.getElementById("login-modal").classList.remove("active");
}

async function switchUserRole(role) {
  try {
    const res = await fetch("/api/auth/switch-role", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ role })
    });
    const data = await res.json();
    if (data.success) {
      currentUser = data.user;
      renderUserCard(currentUser);
      closeLoginModal();

      if (role === "client") {
        showToast("Switched to Client / Business Owner (Sarah Jenkins)", "success");
        switchTab("requirements");
      } else {
        showToast("Switched to Dedicated Sales Executive (Alex Morgan)", "info");
        switchTab("review");
      }
      await fetchCurrentUser();
    }
  } catch (err) {
    showToast("Failed to switch role", "danger");
  }
}

async function handleManualLogin(e) {
  e.preventDefault();
  const email = document.getElementById("login-email").value.trim();
  const password = document.getElementById("login-password").value.trim();

  try {
    const res = await fetch("/api/auth/login", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ email, password })
    });
    const data = await res.json();
    if (data.error) {
      showToast(data.error, "danger");
      return;
    }

    currentUser = data.user;
    renderUserCard(currentUser);
    closeLoginModal();
    showToast(`Signed in as ${currentUser.name} (${currentUser.role})`, "success");
    await initApp();
  } catch (err) {
    showToast("Login failed", "danger");
  }
}

// -------------------------------------------------------------
// CLIENT REQUIREMENTS & STRATEGY HUB
// -------------------------------------------------------------
async function loadClientRequirements() {
  try {
    const res = await fetch("/api/client/requirements");
    if (!res.ok) return;
    const req = await res.json();

    const companyField = document.getElementById("req-company-name");
    const productField = document.getElementById("req-product-name");
    const valuePropField = document.getElementById("req-value-prop");
    const industriesField = document.getElementById("req-target-industries");
    const sizeField = document.getElementById("req-company-size");
    const geoField = document.getElementById("req-target-geography");
    const pricingField = document.getElementById("req-pricing-tier");
    const signalsField = document.getElementById("req-buying-signals");
    const painField = document.getElementById("req-pain-points");
    const notesField = document.getElementById("req-exec-notes");

    if (companyField) companyField.value = req.company_name || "";
    if (productField) productField.value = req.product_name || "";
    if (valuePropField) valuePropField.value = req.value_proposition || "";
    if (industriesField) industriesField.value = req.target_industries || "";
    if (sizeField) sizeField.value = req.target_company_size || "";
    if (geoField) geoField.value = req.target_geography || "";
    if (pricingField) pricingField.value = req.pricing_tier || "";
    if (signalsField) signalsField.value = req.key_buying_signals || "";
    if (painField) painField.value = req.pain_points_solved || "";
    if (notesField) notesField.value = req.notes_for_sales_exec || "";

    // Populate Sales Exec Details
    const execName = document.getElementById("req-exec-name");
    const execTitle = document.getElementById("req-exec-title");
    const execEmail = document.getElementById("req-exec-email");

    if (execName) execName.innerText = req.assigned_sales_exec_name || "Alex Morgan";
    if (execTitle) execTitle.innerText = "Senior Enterprise Account Executive";
    if (execEmail) execEmail.innerText = req.assigned_sales_exec_email || "alex.morgan@salesautopilot.io";

  } catch (err) {
    console.error("Failed to load client requirements:", err);
  }
}

async function saveClientRequirements(e) {
  e.preventDefault();

  const payload = {
    company_name: document.getElementById("req-company-name").value.trim(),
    product_name: document.getElementById("req-product-name").value.trim(),
    value_proposition: document.getElementById("req-value-prop").value.trim(),
    target_industries: document.getElementById("req-target-industries").value.trim(),
    target_company_size: document.getElementById("req-company-size").value.trim(),
    target_geography: document.getElementById("req-target-geography").value.trim(),
    pricing_tier: document.getElementById("req-pricing-tier").value.trim(),
    key_buying_signals: document.getElementById("req-buying-signals").value.trim(),
    pain_points_solved: document.getElementById("req-pain-points").value.trim(),
    notes_for_sales_exec: document.getElementById("req-exec-notes").value.trim()
  };

  try {
    const res = await fetch("/api/client/requirements", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(payload)
    });
    const data = await res.json();
    if (data.success) {
      showToast("Client requirements saved! Autopilot algorithms synchronized.", "success");
      await fetchStats();
      await fetchActivities();
    }
  } catch (err) {
    showToast("Failed to save client requirements", "danger");
  }
}

// -------------------------------------------------------------
// NAVIGATION & SIDEBAR VIEW SWITCHING
// -------------------------------------------------------------
function switchTab(tabId) {
  document.querySelectorAll(".nav-item").forEach(item => {
    item.classList.toggle("active", item.dataset.tab === tabId);
  });

  document.querySelectorAll(".view-panel").forEach(panel => {
    panel.classList.toggle("active", panel.id === `view-${tabId}`);
  });

  if (tabId === "requirements") loadClientRequirements();
  if (tabId === "review") fetchPendingReview();
  if (tabId === "activity") fetchActivities();
  if (tabId === "simulator") populateSimulatorDropdown();
  if (tabId === "cadence") loadCadenceSteps();
  if (tabId === "safeguards") loadSuppressionList();
  if (tabId === "icp") loadIcpSettings();
}

// -------------------------------------------------------------
// DATA FETCHING & KPIS
// -------------------------------------------------------------
async function fetchStats() {
  try {
    const res = await fetch("/api/stats");
    const data = await res.json();

    document.getElementById("kpi-pipeline-val").innerText = data.pipeline_value_formatted;
    document.getElementById("kpi-total-leads").innerText = data.total_leads;
    document.getElementById("kpi-high-icp").innerText = data.high_icp_leads;
    document.getElementById("kpi-response-rate").innerText = `${data.response_rate}%`;
    document.getElementById("kpi-meetings-booked").innerText = data.meetings_booked;
    document.getElementById("kpi-pending-review").innerText = data.pending_human_review;
    document.getElementById("kpi-opt-outs").innerText = data.opted_out_safeguards;

    const badge = document.getElementById("review-badge-count");
    if (badge) {
      badge.innerText = data.pending_human_review;
      badge.style.display = data.pending_human_review > 0 ? "inline-flex" : "none";
    }

    const dncBadge = document.getElementById("nav-dnc-count");
    if (dncBadge) {
      dncBadge.innerText = data.opted_out_safeguards;
    }

    // Mini stats in exec card
    const execPipe = document.getElementById("exec-pipeline-val");
    if (execPipe) execPipe.innerText = data.pipeline_value_formatted;
    const execResp = document.getElementById("exec-response-rate");
    if (execResp) execResp.innerText = `${data.response_rate}%`;
    const execBooked = document.getElementById("exec-demos-booked");
    if (execBooked) execBooked.innerText = data.meetings_booked;

  } catch (err) {
    console.error("Failed to load stats:", err);
  }
}

async function fetchLeads() {
  try {
    let url = `/api/leads?sort=score_desc`;
    if (currentFilterStage !== "all") url += `&stage=${currentFilterStage}`;
    if (currentFilterPriority !== "all") url += `&priority=${currentFilterPriority}`;
    if (currentSearch) url += `&search=${encodeURIComponent(currentSearch)}`;

    const res = await fetch(url);
    currentLeads = await res.json();

    renderKanban(currentLeads);
    renderTable(currentLeads);
    populateSimulatorDropdown();
  } catch (err) {
    console.error("Failed to fetch leads:", err);
  }
}

async function fetchPendingReview() {
  try {
    const res = await fetch("/api/outreach/pending");
    const drafts = await res.json();
    renderReviewQueue(drafts);
  } catch (err) {
    console.error("Failed to fetch review queue:", err);
  }
}

async function fetchActivities() {
  try {
    const res = await fetch("/api/activities");
    const logs = await res.json();
    renderActivityStream(logs);
  } catch (err) {
    console.error("Failed to fetch activities:", err);
  }
}

// -------------------------------------------------------------
// KANBAN PIPELINE VIEW
// -------------------------------------------------------------
const STAGES_CONFIG = [
  { id: "discovered", title: "Discovered", icon: "🔍" },
  { id: "scored", title: "Scored", icon: "🎯" },
  { id: "outreach_drafted", title: "Drafted", icon: "✍️" },
  { id: "in_sequence", title: "In Sequence", icon: "🚀" },
  { id: "replied", title: "Replied", icon: "💬" },
  { id: "meeting_booked", title: "Demo Booked", icon: "📅" },
  { id: "follow_up_scheduled", title: "Follow-up", icon: "⏰" },
  { id: "opted_out", title: "Opted Out", icon: "🚫" },
  { id: "disqualified", title: "Disqualified", icon: "✖️" }
];

function renderKanban(leads) {
  const container = document.getElementById("kanban-board");
  if (!container) return;

  container.innerHTML = "";

  STAGES_CONFIG.forEach(col => {
    const colLeads = leads.filter(l => l.stage === col.id);

    const colEl = document.createElement("div");
    colEl.className = "kanban-col";
    colEl.innerHTML = `
      <div class="kanban-col-header">
        <span class="kanban-col-title">${col.icon} ${col.title}</span>
        <span class="kanban-col-count">${colLeads.length}</span>
      </div>
      <div class="kanban-cards-list" id="col-${col.id}">
        ${colLeads.length === 0 ? `<div style="text-align:center; padding:18px; font-size:0.75rem; color:#64748b;">No leads</div>` : ""}
      </div>
    `;

    const listEl = colEl.querySelector(".kanban-cards-list");
    colLeads.forEach(lead => {
      const card = createKanbanCard(lead);
      listEl.appendChild(card);
    });

    container.appendChild(colEl);
  });
}

function createKanbanCard(lead) {
  const card = document.createElement("div");
  card.className = "kanban-card";
  card.onclick = (e) => {
    if (e.target.closest("button")) return;
    openLeadModal(lead.id);
  };

  const scoreClass = lead.overall_score >= 80 ? "score-high" : (lead.overall_score >= 55 ? "score-med" : "score-low");

  let followUpHtml = "";
  if (lead.follow_up_date) {
    followUpHtml = `<span title="Scheduled re-engagement">⏰ ${lead.follow_up_date}</span>`;
  }

  card.innerHTML = `
    <div class="card-top">
      <span class="card-company">${escapeHtml(lead.company_name)}</span>
      <span class="score-chip ${scoreClass}" title="AI Suitability Score">${lead.overall_score}/100</span>
    </div>
    <div class="card-contact">${escapeHtml(lead.contact_name)} • ${escapeHtml(lead.contact_title)}</div>
    <div class="card-signal" title="${escapeHtml(lead.buying_signals || '')}">
      ⚡ ${escapeHtml(truncate(lead.buying_signals || "Enriching telemetry...", 70))}
    </div>
    <div class="card-footer">
      <div>${followUpHtml || `<span>${lead.company_size}</span>`}</div>
      <div class="card-actions">
        <button class="btn btn-ghost btn-sm" onclick="event.stopPropagation(); triggerDraftOutreach(${lead.id})" title="Generate Personalized Outreach">✉️</button>
        <button class="btn btn-ghost btn-sm" onclick="event.stopPropagation(); openInSimulator(${lead.id})" title="Simulate Response">🧪</button>
      </div>
    </div>
  `;
  return card;
}

// -------------------------------------------------------------
// TABLE VIEW
// -------------------------------------------------------------
function renderTable(leads) {
  const tbody = document.getElementById("leads-table-body");
  if (!tbody) return;

  if (leads.length === 0) {
    tbody.innerHTML = `<tr><td colspan="7" style="text-align:center; padding:30px; color:#94a3b8;">No leads found matching criteria.</td></tr>`;
    return;
  }

  tbody.innerHTML = leads.map(lead => {
    const scoreClass = lead.overall_score >= 80 ? "score-high" : (lead.overall_score >= 55 ? "score-med" : "score-low");
    const stageBadgeClass = getStageBadgeClass(lead.stage);

    return `
      <tr>
        <td>
          <div style="font-weight:700; color:#f8fafc;">${escapeHtml(lead.company_name)}</div>
          <div style="font-size:0.75rem; color:#94a3b8;">${escapeHtml(lead.contact_name)} (${escapeHtml(lead.contact_title)})</div>
          <div style="font-size:0.7rem; color:#60a5fa;">${escapeHtml(lead.email)}</div>
        </td>
        <td>
          <div>${escapeHtml(lead.industry)}</div>
          <div style="font-size:0.75rem; color:#94a3b8;">${escapeHtml(lead.company_size)}</div>
        </td>
        <td style="max-width:240px; font-size:0.76rem; color:#cbd5e1;">
          ${escapeHtml(lead.buying_signals || 'None detected')}
        </td>
        <td>
          <span class="score-chip ${scoreClass}">${lead.overall_score}/100</span>
          <div style="font-size:0.68rem; color:#94a3b8; margin-top:2px;">
            Fit: ${lead.icp_score} | Urg: ${lead.urgency_score}
          </div>
        </td>
        <td>
          <span class="badge ${stageBadgeClass}">${formatStageName(lead.stage)}</span>
        </td>
        <td style="font-size:0.75rem; color:#cbd5e1;">
          ${lead.follow_up_date ? `📅 ${lead.follow_up_date}` : '<span style="color:#64748b;">—</span>'}
        </td>
        <td>
          <div style="display:flex; gap:5px;">
            <button class="btn btn-secondary btn-sm" onclick="openLeadModal(${lead.id})">Dossier</button>
            <button class="btn btn-primary btn-sm" onclick="openInSimulator(${lead.id})">Simulate</button>
          </div>
        </td>
      </tr>
    `;
  }).join("");
}

function handleSearch(val) {
  currentSearch = val;
  fetchLeads();
}

function handleStageFilter(val) {
  currentFilterStage = val;
  fetchLeads();
}

function handlePriorityFilter(val) {
  currentFilterPriority = val;
  fetchLeads();
}

// -------------------------------------------------------------
// HUMAN-IN-THE-LOOP REVIEW QUEUE
// -------------------------------------------------------------
function renderReviewQueue(drafts) {
  const container = document.getElementById("review-list");
  if (!container) return;

  if (drafts.length === 0) {
    container.innerHTML = `
      <div style="background:var(--bg-surface); border:1px dashed var(--border-subtle); border-radius:var(--radius-md); padding:40px; text-align:center;">
        <div style="font-size:32px; margin-bottom:10px;">✨</div>
        <h3 style="font-size:1.1rem; margin-bottom:4px;">Approval Queue is Clear</h3>
        <p style="color:#94a3b8; font-size:0.85rem;">All generated outreach drafts have been reviewed or dispatched. Click "✉️" on any lead card to generate fresh drafts.</p>
      </div>
    `;
    return;
  }

  container.innerHTML = drafts.map(d => {
    let hookText = "Targeted Buying Trigger";
    if (d.personalization_hooks && d.personalization_hooks.signal_cited) {
      hookText = d.personalization_hooks.signal_cited;
    }

    return `
      <div class="review-item-card">
        <div class="review-header">
          <div class="review-meta">
            <span class="review-company">${escapeHtml(d.company_name)}</span>
            <span style="font-size:0.78rem; color:#94a3b8;">To: ${escapeHtml(d.contact_name)} • ${escapeHtml(d.contact_title)}</span>
            <div class="hook-tag">🎯 Personalization Hook: ${escapeHtml(hookText)}</div>
          </div>
          <div style="display:flex; align-items:center; gap:8px;">
            <span class="badge badge-purple">Touch ${d.touch_number || 1}</span>
            <span class="badge badge-amber">Awaiting Approval</span>
          </div>
        </div>
        <div class="review-subject">Subject: ${escapeHtml(d.subject)}</div>
        <div class="review-body">${escapeHtml(d.body)}</div>
        <div class="review-actions">
          <button class="btn btn-danger btn-sm" onclick="rejectDraft(${d.id})">Reject</button>
          <button class="btn btn-success" onclick="approveDraft(${d.id})">
            <span>✓ Approve & Send</span>
          </button>
        </div>
      </div>
    `;
  }).join("");
}

async function approveDraft(draftId) {
  try {
    const res = await fetch(`/api/outreach/${draftId}/approve`, { method: "POST" });
    const data = await res.json();
    if (data.success) {
      showToast("Outreach approved and dispatched! Lead moved to In Sequence.", "success");
      await initApp();
    }
  } catch (err) {
    showToast("Failed to approve draft", "danger");
  }
}

async function rejectDraft(draftId) {
  try {
    const res = await fetch(`/api/outreach/${draftId}/reject`, { method: "POST" });
    const data = await res.json();
    if (data.success) {
      showToast("Draft rejected.", "warning");
      await initApp();
    }
  } catch (err) {
    showToast("Failed to reject draft", "danger");
  }
}

async function triggerDraftOutreach(leadId, tone = "consultative") {
  try {
    showToast("AI generating customized 1:1 outreach...", "info");
    const res = await fetch(`/api/leads/${leadId}/generate-outreach`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ tone: tone })
    });
    const data = await res.json();
    if (data.error) {
      showToast(data.error, "danger");
      return;
    }
    showToast("Personalized draft created & placed in Human Review Queue!", "success");
    await initApp();
    switchTab("review");
  } catch (err) {
    showToast("Failed to generate outreach", "danger");
  }
}

// -------------------------------------------------------------
// RESPONSE INTELLIGENCE SIMULATOR (SHOWCASE SANDBOX)
// -------------------------------------------------------------
function populateSimulatorDropdown() {
  const select = document.getElementById("sim-lead-select");
  if (!select) return;

  const currentVal = select.value;
  select.innerHTML = currentLeads.map(l => `
    <option value="${l.id}">${escapeHtml(l.company_name)} - ${escapeHtml(l.contact_name)} (${formatStageName(l.stage)})</option>
  `).join("");

  if (selectedSimLeadId) {
    select.value = selectedSimLeadId;
  } else if (currentVal) {
    select.value = currentVal;
  }

  updateSimLeadContext();
}

function updateSimLeadContext() {
  const select = document.getElementById("sim-lead-select");
  const contextDiv = document.getElementById("sim-lead-context");
  if (!select || !contextDiv) return;

  const leadId = parseInt(select.value);
  selectedSimLeadId = leadId;
  const lead = currentLeads.find(l => l.id === leadId);

  if (!lead) {
    contextDiv.innerHTML = "No lead selected.";
    return;
  }

  contextDiv.innerHTML = `
    <div style="display:flex; justify-content:space-between; margin-bottom:4px;">
      <strong>${escapeHtml(lead.company_name)}</strong>
      <span class="badge ${getStageBadgeClass(lead.stage)}">${formatStageName(lead.stage)}</span>
    </div>
    <div style="color:#94a3b8; font-size:0.75rem;">
      Contact: ${escapeHtml(lead.contact_name)} (${escapeHtml(lead.contact_title)}) • ICP: ${lead.overall_score}/100
    </div>
    <div style="color:#cbd5e1; font-size:0.73rem; margin-top:4px;">
      Signals: ${escapeHtml(lead.buying_signals || "None")}
    </div>
  `;
}

function openInSimulator(leadId) {
  selectedSimLeadId = leadId;
  switchTab("simulator");
  populateSimulatorDropdown();
}

function setPresetReply(type) {
  const textarea = document.getElementById("sim-reply-text");
  if (!textarea) return;

  switch (type) {
    case "delay":
      textarea.value = "I'm interested, but contact me next month.";
      break;
    case "meeting":
      textarea.value = "Thanks for reaching out! Can we do a 15-min demo this Thursday at 2 PM EST to see fleet specs?";
      break;
    case "referral":
      textarea.value = "I am not the right person for laptop procurement. Please speak with Sarah Chen, VP of IT (schen@company.com).";
      break;
    case "objection":
      textarea.value = "Our hardware procurement budget is frozen until Q4 and we are locked into our existing Dell contract.";
      break;
    case "info":
      textarea.value = "Sounds promising. Can you send over compliance documentation, SOC2 specs, and your enterprise pricing sheet?";
      break;
    case "optout":
      textarea.value = "Please unsubscribe me and remove our company from your email lists immediately.";
      break;
  }
}

async function executeReplySimulation() {
  const select = document.getElementById("sim-lead-select");
  const textarea = document.getElementById("sim-reply-text");
  const outputContainer = document.getElementById("sim-result-container");

  const leadId = parseInt(select.value);
  const text = textarea.value.trim();

  if (!leadId) {
    showToast("Please select a lead first", "warning");
    return;
  }

  if (!text) {
    showToast("Please enter or select a reply message", "warning");
    return;
  }

  outputContainer.innerHTML = `
    <div style="text-align:center; padding:40px;">
      <div class="empty-state-icon">⚙️</div>
      <h4>Autopilot Analyzing Reply Intent...</h4>
      <p style="color:#94a3b8; font-size:0.85rem;">Processing sentiment, extracting temporal constraints, and synchronizing CRM state.</p>
    </div>
  `;

  try {
    const res = await fetch("/api/replies/simulate", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ lead_id: leadId, reply_text: text })
    });

    const data = await res.json();
    if (data.error) {
      showToast(data.error, "danger");
      return;
    }

    renderSimulationResult(data);
    showToast("CRM synchronized autonomously!", "success");
    await fetchStats();
    await fetchLeads();
    await fetchActivities();
  } catch (err) {
    showToast("Failed to simulate reply", "danger");
  }
}

function renderSimulationResult(result) {
  const container = document.getElementById("sim-result-container");
  const a = result.analysis;

  const sentimentBadgeClass = a.sentiment === "positive" ? "badge-green" : (a.sentiment === "opt_out" ? "badge-rose" : "badge-blue");

  let delayHtml = "";
  if (result.follow_up_date) {
    delayHtml = `
      <div class="res-card">
        <div class="res-label">Scheduled Re-Engagement</div>
        <div class="res-val" style="color:#60a5fa; font-size:0.95rem;">📅 ${result.follow_up_date} (+${a.delay_days}d)</div>
      </div>
    `;
  }

  let safeguardAlertHtml = "";
  if (result.is_opted_out) {
    safeguardAlertHtml = `
      <div class="res-box-highlight safeguard-alert">
        <div class="res-title" style="color:#f87171;">🛡️ Consent Safeguard Active</div>
        <p style="font-size:0.83rem; color:#fca5a5;">
          Recipient was immediately suppressed from automated messaging. Any queued outreach drafts have been cancelled automatically.
        </p>
      </div>
    `;
  }

  container.className = "sim-result-box";
  container.innerHTML = `
    <div class="result-metric-grid">
      <div class="res-card">
        <div class="res-label">Detected Intent</div>
        <div class="res-val" style="font-size:1.05rem;">
          <span class="badge ${sentimentBadgeClass}">${a.intent.replace('_', ' ').toUpperCase()}</span>
        </div>
      </div>
      <div class="res-card">
        <div class="res-label">AI Confidence</div>
        <div class="res-val" style="color:#34d399;">${Math.round(a.confidence * 100)}%</div>
      </div>
      ${delayHtml}
      <div class="res-card">
        <div class="res-label">CRM Deal Stage Transition</div>
        <div class="stage-transition-pill">
          ➔ <span class="badge ${getStageBadgeClass(result.updated_stage)}">${formatStageName(result.updated_stage)}</span>
        </div>
      </div>
    </div>

    ${safeguardAlertHtml}

    <div class="res-box-highlight">
      <div class="res-title">🧠 AI Decision & Rationale</div>
      <p style="font-size:0.83rem; color:#cbd5e1; margin-bottom:8px;">${escapeHtml(a.ai_reasoning)}</p>
      <div style="font-size:0.75rem; color:#60a5fa; font-weight:600;">
        ⚡ Recommended Next Action: ${escapeHtml(a.recommended_action)}
      </div>
    </div>
  `;
}

// -------------------------------------------------------------
// CADENCE BUILDER
// -------------------------------------------------------------
async function loadCadenceSteps() {
  try {
    const res = await fetch("/api/cadence");
    const steps = await res.json();
    const container = document.getElementById("cadence-steps-container");
    if (!container) return;

    container.innerHTML = steps.map(s => `
      <div class="cadence-step-card">
        <div class="cadence-step-number">T${s.step_number}</div>
        <div class="cadence-step-body">
          <div class="cadence-step-title">${escapeHtml(s.title)}</div>
          <div class="cadence-step-purpose">${escapeHtml(s.purpose)}</div>
          <div style="font-size:0.72rem; color:#60a5fa; margin-top:4px;">
            Focus: ${escapeHtml(s.template_focus)}
          </div>
        </div>
        <div class="cadence-delay-badge">
          ${s.delay_days === 0 ? "⚡ Day 1 Immediate" : `⏰ +${s.delay_days} Days Delay`}
        </div>
      </div>
    `).join("");
  } catch (err) {
    console.error("Failed to load cadence:", err);
  }
}

// -------------------------------------------------------------
// ICP SETTINGS & WEIGHTS
// -------------------------------------------------------------
async function loadIcpSettings() {
  try {
    const res = await fetch("/api/settings/icp");
    const settings = await res.json();

    if (settings.icp_weight) document.getElementById("input-icp-weight").value = settings.icp_weight;
    if (settings.urgency_weight) document.getElementById("input-urgency-weight").value = settings.urgency_weight;
    if (settings.authority_weight) document.getElementById("input-authority-weight").value = settings.authority_weight;
    if (settings.default_tone) document.getElementById("input-default-tone").value = settings.default_tone;
    if (settings.target_industries) document.getElementById("input-target-industries").value = settings.target_industries;
    if (settings.target_icp_min_size) document.getElementById("input-target-size").value = settings.target_icp_min_size;
    if (settings.company_pitch) document.getElementById("input-target-pitch").value = settings.company_pitch;

    updateWeightLabels();
  } catch (err) {
    console.error("Failed to load ICP settings:", err);
  }
}

function updateWeightLabels() {
  const icp = document.getElementById("input-icp-weight").value;
  const urg = document.getElementById("input-urgency-weight").value;
  const auth = document.getElementById("input-authority-weight").value;

  document.getElementById("label-icp-weight").innerText = `${icp}%`;
  document.getElementById("label-urgency-weight").innerText = `${urg}%`;
  document.getElementById("label-authority-weight").innerText = `${auth}%`;
}

async function saveIcpSettings() {
  const payload = {
    icp_weight: document.getElementById("input-icp-weight").value,
    urgency_weight: document.getElementById("input-urgency-weight").value,
    authority_weight: document.getElementById("input-authority-weight").value,
    default_tone: document.getElementById("input-default-tone").value,
    target_industries: document.getElementById("input-target-industries").value,
    target_icp_min_size: document.getElementById("input-target-size").value,
    company_pitch: document.getElementById("input-target-pitch").value
  };

  try {
    const res = await fetch("/api/settings/icp", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(payload)
    });
    const data = await res.json();
    if (data.success) {
      showToast("ICP & scoring weights updated successfully!", "success");
    }
  } catch (err) {
    showToast("Failed to save ICP settings", "danger");
  }
}

// -------------------------------------------------------------
// COMPLIANCE & DNC LIST
// -------------------------------------------------------------
async function loadSuppressionList() {
  try {
    const res = await fetch("/api/safeguards/suppression-list");
    const items = await res.json();
    const tbody = document.getElementById("suppression-table-body");
    if (!tbody) return;

    if (items.length === 0) {
      tbody.innerHTML = `<tr><td colspan="4" style="text-align:center; padding:15px; color:#94a3b8;">No contacts currently suppressed.</td></tr>`;
      return;
    }

    tbody.innerHTML = items.map(item => `
      <tr>
        <td>
          <div style="font-weight:700;">${escapeHtml(item.company_name)}</div>
          <div style="font-size:0.75rem; color:#60a5fa;">${escapeHtml(item.email)}</div>
        </td>
        <td style="font-size:0.78rem; color:#cbd5e1;">${escapeHtml(item.opt_out_reason || 'Unsubscribe requested')}</td>
        <td style="font-size:0.72rem; color:#94a3b8;">${escapeHtml(item.updated_at || 'Active')}</td>
        <td><span class="badge badge-rose">Suppressed</span></td>
      </tr>
    `).join("");
  } catch (err) {
    console.error("Failed to load suppression list:", err);
  }
}

async function submitManualSuppression(e) {
  e.preventDefault();
  const email = document.getElementById("suppress-email").value.trim();
  const reason = document.getElementById("suppress-reason").value.trim();

  try {
    const res = await fetch("/api/safeguards/suppression-list", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ email, reason })
    });
    const data = await res.json();
    if (data.success) {
      showToast("Email added to Do-Not-Contact registry.", "success");
      document.getElementById("suppress-email").value = "";
      document.getElementById("suppress-reason").value = "";
      await initApp();
    }
  } catch (err) {
    showToast("Failed to add suppression", "danger");
  }
}

// -------------------------------------------------------------
// ONE-CLICK AUTOPILOT CYCLE
// -------------------------------------------------------------
async function triggerAutopilotCycle() {
  try {
    showToast("Running autonomous cycle: Scoring leads & drafting outreach...", "info");
    const res = await fetch("/api/autopilot/run-cycle", { method: "POST" });
    const data = await res.json();
    if (data.success) {
      showToast(`Autopilot Cycle: Scored ${data.scored_count} leads, Generated ${data.drafts_generated} drafts!`, "success");
      await initApp();
    }
  } catch (err) {
    showToast("Error running autopilot cycle", "danger");
  }
}

// -------------------------------------------------------------
// TIMELINE STREAM
// -------------------------------------------------------------
function renderActivityStream(logs) {
  const container = document.getElementById("timeline-stream");
  if (!container) return;

  if (logs.length === 0) {
    container.innerHTML = `<div style="text-align:center; padding:30px; color:#94a3b8;">No activity recorded yet.</div>`;
    return;
  }

  container.innerHTML = logs.map(log => {
    let icon = "⚡";
    if (log.action_type === "lead_discovered") icon = "🔍";
    if (log.action_type === "outreach_generated") icon = "✍️";
    if (log.action_type === "human_approved") icon = "✓";
    if (log.action_type === "reply_received") icon = "💬";
    if (log.action_type === "intent_classified") icon = "🤖";
    if (log.action_type === "follow_up_scheduled") icon = "⏰";
    if (log.action_type === "safeguard_triggered") icon = "🛡️";

    return `
      <div class="timeline-item">
        <div class="timeline-icon-wrap">${icon}</div>
        <div class="timeline-content">
          <div class="timeline-title-row">
            <span class="timeline-title">${escapeHtml(log.title)}</span>
            <span class="timeline-time">${escapeHtml(log.created_at)}</span>
          </div>
          <div class="timeline-desc">${escapeHtml(log.description)}</div>
          ${log.company_name ? `<div style="font-size:0.72rem; color:#60a5fa; margin-top:2px;">Target: ${escapeHtml(log.company_name)}</div>` : ""}
        </div>
      </div>
    `;
  }).join("");
}

// -------------------------------------------------------------
// LEAD DOSSIER MODAL
// -------------------------------------------------------------
async function openLeadModal(leadId) {
  try {
    const res = await fetch(`/api/leads/${leadId}`);
    const data = await res.json();
    if (data.error) return;

    const lead = data.lead;
    const modal = document.getElementById("lead-modal");

    document.getElementById("modal-company-name").innerText = lead.company_name;
    document.getElementById("modal-contact-details").innerText = `${lead.contact_name} • ${lead.contact_title} (${lead.email})`;

    const stageBadge = document.getElementById("modal-stage-badge");
    stageBadge.className = `badge ${getStageBadgeClass(lead.stage)}`;
    stageBadge.innerText = formatStageName(lead.stage);

    const breakdown = lead.scoring_breakdown || {};

    const body = document.getElementById("modal-lead-body");
    body.innerHTML = `
      <div style="display:grid; grid-template-columns:1fr 1fr; gap:14px; margin-bottom:18px;">
        <div style="background:var(--bg-surface-elevated); padding:12px; border-radius:var(--radius-sm); border:1px solid var(--border-subtle);">
          <div style="font-size:0.72rem; text-transform:uppercase; color:#94a3b8; font-weight:700; margin-bottom:4px;">Firmographics</div>
          <div style="font-size:0.8rem;"><strong>Industry:</strong> ${escapeHtml(lead.industry)}</div>
          <div style="font-size:0.8rem;"><strong>Size:</strong> ${escapeHtml(lead.company_size)}</div>
          <div style="font-size:0.8rem;"><strong>Budget:</strong> ${escapeHtml(lead.estimated_budget || 'N/A')}</div>
          <div style="font-size:0.8rem; margin-top:2px;"><strong>Stack:</strong> ${escapeHtml(lead.tech_stack || 'N/A')}</div>
        </div>

        <div style="background:var(--bg-surface-elevated); padding:12px; border-radius:var(--radius-sm); border:1px solid var(--border-subtle);">
          <div style="font-size:0.72rem; text-transform:uppercase; color:#94a3b8; font-weight:700; margin-bottom:4px;">AI Suitability Score</div>
          <div style="display:flex; align-items:baseline; gap:8px;">
            <span style="font-size:1.6rem; font-weight:800; color:#34d399;">${lead.overall_score}</span>
            <span style="color:#94a3b8; font-size:0.8rem;">/ 100 Overall</span>
          </div>
          <div style="font-size:0.72rem; color:#cbd5e1; margin-top:2px;">
            Fit: <strong>${lead.icp_score}/100</strong> | Urgency: <strong>${lead.urgency_score}/100</strong>
          </div>
          <div style="font-size:0.7rem; color:#94a3b8; margin-top:3px; font-style:italic;">
            ${escapeHtml(breakdown.summary || '')}
          </div>
        </div>
      </div>

      <div style="margin-bottom:16px;">
        <h4 style="font-size:0.75rem; text-transform:uppercase; color:#94a3b8; margin-bottom:4px;">Active Buying Signals</h4>
        <div style="background:rgba(59, 130, 246, 0.08); border:1px solid rgba(59, 130, 246, 0.25); border-radius:var(--radius-sm); padding:8px 12px; font-size:0.8rem; color:#93c5fd;">
          ⚡ ${escapeHtml(lead.buying_signals || 'No signals registered')}
        </div>
      </div>

      <div>
        <h4 style="font-size:0.75rem; text-transform:uppercase; color:#94a3b8; margin-bottom:6px;">Outreach & Reply History</h4>
        ${renderDossierHistory(data.drafts, data.replies)}
      </div>
    `;

    const footer = document.getElementById("modal-actions-footer");
    footer.innerHTML = `
      <button class="btn btn-secondary" onclick="closeLeadModal()">Close</button>
      <button class="btn btn-secondary" onclick="closeLeadModal(); openInSimulator(${lead.id})">Simulate Reply</button>
      <button class="btn btn-primary" onclick="closeLeadModal(); triggerDraftOutreach(${lead.id})">Generate Outreach</button>
    `;

    modal.classList.add("active");
  } catch (err) {
    console.error("Failed to load dossier:", err);
  }
}

function renderDossierHistory(drafts, replies) {
  if (drafts.length === 0 && replies.length === 0) {
    return `<div style="font-size:0.78rem; color:#64748b; padding:8px 0;">No outreach communications dispatched yet.</div>`;
  }

  let html = `<div style="display:flex; flex-direction:column; gap:8px;">`;

  drafts.forEach(d => {
    html += `
      <div style="background:var(--bg-surface-elevated); border:1px solid var(--border-subtle); border-radius:var(--radius-sm); padding:8px 12px;">
        <div style="display:flex; justify-content:space-between; margin-bottom:2px;">
          <span style="font-size:0.72rem; color:#94a3b8;">✉️ Outreach (${d.status.toUpperCase()})</span>
          <span style="font-size:0.68rem; color:#64748b;">${d.created_at}</span>
        </div>
        <div style="font-weight:600; font-size:0.8rem; margin-bottom:2px;">${escapeHtml(d.subject)}</div>
        <div style="font-size:0.75rem; color:#cbd5e1; white-space:pre-wrap;">${escapeHtml(truncate(d.body, 140))}</div>
      </div>
    `;
  });

  replies.forEach(r => {
    html += `
      <div style="background:rgba(16, 185, 129, 0.08); border:1px solid rgba(16, 185, 129, 0.25); border-radius:var(--radius-sm); padding:8px 12px;">
        <div style="display:flex; justify-content:space-between; margin-bottom:2px;">
          <span style="font-size:0.72rem; color:#34d399; font-weight:700;">💬 Prospect Reply (${r.intent.toUpperCase()})</span>
          <span style="font-size:0.68rem; color:#64748b;">${r.created_at}</span>
        </div>
        <div style="font-size:0.78rem; color:#f8fafc; margin-bottom:2px;">"${escapeHtml(r.reply_text)}"</div>
        <div style="font-size:0.7rem; color:#94a3b8;">AI Action: ${escapeHtml(r.recommended_action)}</div>
      </div>
    `;
  });

  html += `</div>`;
  return html;
}

function closeLeadModal() {
  document.getElementById("lead-modal").classList.remove("active");
}

function openAddLeadModal() {
  document.getElementById("add-lead-modal").classList.add("active");
}

function closeAddLeadModal() {
  document.getElementById("add-lead-modal").classList.remove("active");
}

async function submitNewLead(e) {
  e.preventDefault();

  const payload = {
    company_name: document.getElementById("add-company").value.trim(),
    industry: document.getElementById("add-industry").value.trim(),
    contact_name: document.getElementById("add-contact").value.trim(),
    contact_title: document.getElementById("add-title").value.trim(),
    email: document.getElementById("add-email").value.trim(),
    company_size: document.getElementById("add-size").value,
    tech_stack: document.getElementById("add-tech").value.trim(),
    buying_signals: document.getElementById("add-signals").value.trim()
  };

  try {
    const res = await fetch("/api/leads", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(payload)
    });

    const data = await res.json();
    if (data.error) {
      showToast(data.error, "danger");
      return;
    }

    showToast(`Lead added! Suitability scored: ${data.score.overall_score}/100`, "success");
    closeAddLeadModal();
    document.getElementById("add-lead-form").reset();
    await initApp();
  } catch (err) {
    showToast("Failed to create lead", "danger");
  }
}

async function discoverBatchLeads() {
  try {
    showToast("Scanning market sources for qualified accounts...", "info");
    const res = await fetch("/api/leads/discover-batch", { method: "POST" });
    const data = await res.json();

    if (data.discovered_count > 0) {
      showToast(`Discovered & enriched ${data.discovered_count} qualified accounts!`, "success");
    } else {
      showToast("Market scan complete: No new accounts meeting criteria.", "info");
    }

    await initApp();
  } catch (err) {
    showToast("Error during lead discovery scan", "danger");
  }
}

async function resetDemoData() {
  if (!confirm("Reset database to initial prototype test accounts?")) return;

  try {
    await fetch("/api/reset-demo", { method: "POST" });
    showToast("Demo dataset restored to initial state.", "success");
    await initApp();
  } catch (err) {
    showToast("Failed to reset demo data", "danger");
  }
}

// -------------------------------------------------------------
// HELPERS
// -------------------------------------------------------------
function getStageBadgeClass(stage) {
  switch (stage) {
    case "discovered": return "badge-gray";
    case "scored": return "badge-blue";
    case "outreach_drafted": return "badge-amber";
    case "in_sequence": return "badge-purple";
    case "replied": return "badge-blue";
    case "meeting_booked": return "badge-green";
    case "follow_up_scheduled": return "badge-amber";
    case "opted_out": return "badge-rose";
    case "disqualified": return "badge-gray";
    default: return "badge-gray";
  }
}

function formatStageName(stage) {
  return (stage || "").split("_").map(w => w.charAt(0).toUpperCase() + w.slice(1)).join(" ");
}

function truncate(str, len) {
  if (!str) return "";
  return str.length > len ? str.substring(0, len) + "..." : str;
}

function escapeHtml(str) {
  if (!str) return "";
  return String(str)
    .replace(/&/g, "&amp;")
    .replace(/</g, "&lt;")
    .replace(/>/g, "&gt;")
    .replace(/"/g, "&quot;")
    .replace(/'/g, "&#039;");
}

function showToast(msg, type = "info") {
  const container = document.getElementById("toast-container");
  if (!container) return;

  const toast = document.createElement("div");
  toast.className = `toast toast-${type}`;
  toast.innerText = msg;

  container.appendChild(toast);
  setTimeout(() => {
    toast.style.opacity = "0";
    toast.style.transition = "opacity 0.3s ease";
    setTimeout(() => toast.remove(), 300);
  }, 3500);
}
```

---

