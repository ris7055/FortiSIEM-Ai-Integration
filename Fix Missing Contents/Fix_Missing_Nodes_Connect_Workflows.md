# N8N FortiSIEM Workflow: Missing Nodes & Connection Guide

## 📋 Current Status Analysis

### What You're Seeing:
```
✅ TOP WORKFLOW (Incident Detection & Investigation)
   Trigger → Get Incidents → Parse → Filter → Create Ticket → 
   Threat Intel (VirusTotal & FortiGuard) → Consolidate → 
   Slack Alert → FortiSIEM Update → Database → Email

✅ BOTTOM WORKFLOW (Incident Resolution)
   Trigger → Verify Resolution → Check if Resolved → 
   Close Ticket → Mark Resolved → Update Database → Slack Alert
```

### Why Two Workflows?
```
This is CORRECT by design:

WORKFLOW 1 (Immediate - Seconds to Minutes)
- Runs every 5 minutes
- Detects new incidents
- Creates tickets
- Sends notifications

WORKFLOW 2 (Monitoring - Periodic Check)
- Runs every 30 minutes
- Checks incident status
- Verifies threats resolved
- Closes tickets

They run INDEPENDENTLY (not connected)
```

---

## 🔴 Missing Nodes Issue

### Why Nodes Are Missing:

**Common Reasons:**
1. **Credentials not configured** - N8N is hiding HTTP nodes without proper creds
2. **Node types not installed** - Some nodes may need to be added
3. **Credentials mismatch** - Referenced credentials don't exist in your instance

### How to Fix Missing Nodes:

#### Step 1: Check N8N Version
```
Go to: Profile (bottom left) → About
Should be: N8N 1.x or later
If older, update N8N
```

#### Step 2: Identify Missing Credential Types
**Red Triangle (⚠️) = Missing Credentials**

From your screenshot, likely missing:
- [ ] FortiSIEM API Token
- [ ] Jira Credentials
- [ ] Slack Bot Token
- [ ] VirusTotal API Key
- [ ] PostgreSQL Credentials

#### Step 3: Add Credentials

**For FortiSIEM:**
```
1. Click: Credentials (left sidebar)
2. Click: + New
3. Type: Select "HTTP Bearer Token"
4. Name: fortisiem_api_token
5. Token: Your FortiSIEM API token
6. Save & Test
```

**For Jira:**
```
1. Credentials → + New
2. Type: Jira
3. Jira Instance URL: https://your-jira.atlassian.net
4. Email: your-email@company.com
5. API Token: Your Jira API token
6. Save & Test
```

**For Slack:**
```
1. Credentials → + New
2. Type: Slack
3. Bot Token: xoxb-your-bot-token
4. Save & Test
```

**For VirusTotal:**
```
1. Credentials → + New
2. Type: HTTP Bearer Token
3. Name: virustotal_api_key
4. Token: Your VirusTotal API key
5. Save & Test
```

**For PostgreSQL:**
```
1. Credentials → + New
2. Type: PostgreSQL
3. Host: Your DB host
4. Port: 5432
5. Database: Your DB name
6. User: Your DB user
7. Password: Your DB password
8. Save & Test
```

#### Step 4: Re-add Missing Nodes

If nodes still show as missing after credentials are added:

**For HTTP Requests:**
```
1. Click: + Add Node
2. Search: HTTP Request
3. Select: HTTP Request
4. Configure with correct credential
5. Set URL properly
```

**Common HTTP Nodes Needed:**
- ✅ Get New Incidents from FortiSIEM (HTTP)
- ✅ Check IP with VirusTotal (HTTP)
- ✅ Check FortiGuard Threat Intel (HTTP)
- ✅ Update Incident Status (HTTP)
- ✅ Verify Threat Resolution (HTTP)

---

## 🔗 Do You Need to Connect the Two Workflows?

### SHORT ANSWER: **NO - They Run Independently**

### DETAILED EXPLANATION:

```
WORKFLOW 1: Incident Detection (Runs Every 5 Minutes)
├─ Trigger: Scheduled Interval (Every 5 min)
├─ Action: Check FortiSIEM for NEW incidents
├─ Output: Creates tickets, sends alerts
└─ SLA Tracking: Started

                    ↓ (TIME PASSES - 2 HOURS)

WORKFLOW 2: Incident Closure (Runs Every 30 Minutes)
├─ Trigger: Scheduled Interval (Every 30 min)
├─ Reads: SLA Database (from Workflow 1)
├─ Action: Check if incidents are resolved
├─ Output: Closes tickets, updates status
└─ SLA Tracking: Completed
```

### They Are Connected By:
```
✅ DATABASE (Shared State)

Workflow 1 writes to: sla_tracking table
Workflow 2 reads from: sla_tracking table

This is the PROPER way to connect independent processes
```

### Why This Design?
```
✅ Scalability: Handle 1000s of incidents independently
✅ Resilience: One workflow failure doesn't block the other
✅ Performance: Non-blocking parallel execution
✅ Monitoring: Easy to debug each phase separately
✅ Flexibility: Can run at different intervals
```

---

## ✅ CORRECT WORKFLOW SETUP

### Workflow 1: Incident Triage & Investigation
**Name:** `FortiSIEM - Incident Detection & Investigation`
**Trigger:** Every 5 minutes
**Flow:**
```
Poll FortiSIEM 
  ↓
Parse Incidents
  ↓
Filter (remove empty)
  ↓
For Each Incident:
  ├─ Create Jira Ticket
  ├─ Run Threat Intel (parallel):
  │  ├─ VirusTotal lookup
  │  └─ FortiGuard lookup
  ├─ Consolidate results
  ├─ Notify Slack
  ├─ Update FortiSIEM status
  ├─ Log to Database
  └─ Send Email to Customer
```

### Workflow 2: Incident Closure & Verification
**Name:** `FortiSIEM - Incident Remediation & Closure`
**Trigger:** Every 30 minutes
**Flow:**
```
Check SLA Database for "IN_PROGRESS" incidents
  ↓
For Each Incident:
  ├─ Verify threat resolution
  ├─ Check if activity ceased
  ├─ If resolved:
  │  ├─ Close Jira ticket
  │  ├─ Update FortiSIEM status
  │  ├─ Update Database
  │  ├─ Create knowledge base entry
  │  └─ Notify team
  └─ If not resolved:
     └─ Keep in progress (check again in 30 min)
```

---

## 🔧 Step-by-Step: Add Missing Nodes

### For Workflow 1 (Top):

#### Missing Node: "Parse Incident Data" (If showing as ?)
```
1. Delete the ? node
2. Click: + Add Node
3. Search: Function Item
4. Name: Parse Incident Data
5. Code:
   return {
     incidentId: $json.incident_id,
     incidentName: $json.incident_name,
     severity: $json.severity,
     sourceIP: $json.source_ip,
     destIP: $json.dest_ip
   };
6. Connect from: Get New Incidents from FortiSIEM
```

#### Missing Node: "Filter Empty Incidents" (If missing)
```
1. Click: + Add Node
2. Search: If
3. Name: Filter Empty Incidents
4. Condition: severity is not empty
5. Connect from: Parse Incident Data
6. Output: true path continues
```

#### Missing Node: "Consolidate Threat Intel" (The <> symbol)
```
1. Click the <> node
2. Or add new: Function Item
3. Name: Consolidate Threat Intel
4. Code:
   const vt = $input.first().json;
   const fg = $input.first().json;
   
   let riskScore = 0;
   if (vt?.data?.last_analysis_stats?.malicious > 3) riskScore += 50;
   if (fg?.blacklisted) riskScore += 30;
   
   return {
     sourceIP: $json.source_ip,
     riskScore: Math.min(riskScore, 100),
     virustotal: vt,
     fortiGuard: fg
   };
```

---

## 🔴 Common Missing Node Issues & Fixes

### Issue 1: HTTP Request nodes showing as "?" or greyed out
```
CAUSE: Credentials not configured
FIX:
1. Go to Credentials
2. Create HTTP Bearer Token for each API
3. Return to node
4. Select correct credential from dropdown
5. Should turn blue now
```

### Issue 2: Slack node says "post: message" with warning
```
CAUSE: Slack credentials missing
FIX:
1. Create Slack credentials (see above)
2. Go to Slack node
3. Click: Change Credentials
4. Select your Slack credential
5. Channel selector should now work
```

### Issue 3: PostgreSQL node shows error triangle
```
CAUSE: Database connection not configured
FIX:
1. Create PostgreSQL credentials
2. Test connection first
3. Return to node
4. Verify query syntax
5. Click: Execute to test
```

### Issue 4: Jira node isn't recognizing project
```
CAUSE: Jira credentials incorrect or project doesn't exist
FIX:
1. Test Jira credentials first
2. Verify project key (SOC) exists in Jira
3. Verify you have permission to create issues
4. Try manually creating issue in Jira
5. Then retry in N8N
```

---

## 📋 Complete Checklist: Get Workflows Running

### Pre-Requisites
- [ ] FortiSIEM API key obtained
- [ ] Jira API token created
- [ ] Slack bot token obtained
- [ ] VirusTotal API key obtained
- [ ] PostgreSQL database created
- [ ] Email credentials ready

### Workflow 1 Setup
- [ ] Credentials added (all 6)
- [ ] Trigger node configured (5 min interval)
- [ ] Get Incidents node has correct URL
- [ ] Filter node logic correct
- [ ] All HTTP nodes have credentials selected
- [ ] Jira node connected to correct project
- [ ] Slack node pointing to correct channel
- [ ] PostgreSQL node pointing to correct database
- [ ] Email node has SMTP credentials
- [ ] All connections properly linked
- [ ] Workflow saves without errors

### Workflow 2 Setup
- [ ] Trigger node configured (30 min interval)
- [ ] Verify Resolution node has correct FortiSIEM URL
- [ ] Check if Resolved node logic correct
- [ ] Close Ticket node connected
- [ ] Update incident node configured
- [ ] Database update query correct
- [ ] Slack notification configured
- [ ] All connections properly linked
- [ ] Workflow saves without errors

### Testing
- [ ] Workflow 1: Execute manually → Check for errors
- [ ] Workflow 1: Verify Slack message sent
- [ ] Workflow 1: Check database entry created
- [ ] Workflow 2: Execute manually → Check for errors
- [ ] Workflow 2: Verify database updated
- [ ] Both: Activate and monitor for 1 hour

---

## 🚀 Quick Fix Command (If Credentials Missing)

### For N8N via Azure VM:

If you're using N8N locally on Azure VM without Docker, the executable might be at:

```bash
# Test N8N installation
sudo systemctl status n8n

# Check N8N logs
sudo journalctl -u n8n -f

# Restart N8N (if needed)
sudo systemctl restart n8n

# Check N8N version
ps aux | grep n8n
```

### Add Credentials via CLI (If UI doesn't work):

If the N8N UI credentials section has issues:

```bash
# Connect via SSH to Azure VM
ssh azureuser@your-vm-ip

# Go to N8N user data directory
cd ~/.n8n/

# Check credentials file
ls -la

# Restart and use UI (should work now)
sudo systemctl restart n8n
```

---

## 📊 Expected Node Count

### Workflow 1 Should Have:
```
1. Trigger - Poll FortiSIEM ✅
2. Get New Incidents ✅
3. Parse Data ✅
4. Filter Empty ✅
5. Create Jira ✅
6. VirusTotal Check ✅
7. FortiGuard Check ✅
8. Consolidate ✅
9. Slack Alert ✅
10. Update FortiSIEM ✅
11. Log to DB ✅
12. Send Email ✅

Total: 12 nodes
```

### Workflow 2 Should Have:
```
1. Trigger - Check Status ✅
2. Verify Resolution ✅
3. Check if Resolved ✅
4. Close Jira ✅
5. Mark Incident Resolved ✅
6. Update Database ✅
7. Slack Notification ✅

Total: 7 nodes
```

---

## ❓ FAQ: Workflows & Connections

### Q: Should the workflows be connected by webhook?
**A:** NO. Scheduled independent workflows are better.
- Workflow 1 runs every 5 minutes
- Workflow 2 runs every 30 minutes
- They communicate through database

### Q: What if Workflow 1 fails?
**A:** Workflow 2 still runs. Independent execution = resilient.

### Q: Can I combine them into one workflow?
**A:** Possible but not recommended:
- Single workflow = 19 nodes (too complex)
- Hard to debug individual phases
- Better to keep separate + monitor each

### Q: What if I want real-time closure (not every 30 min)?
**A:** Change Workflow 2 trigger:
- From: Every 30 minutes
- To: Webhook (triggered by Workflow 1)
- Trade-off: Less resilient but faster

### Q: How do I know if a workflow is working?
**A:** Check:
- N8N Dashboard: Workflow executions log
- Slack: Notifications appearing
- Database: New entries in sla_tracking
- Jira: New tickets created

---

## 🆘 Troubleshooting: Common Missing Node Messages

### Message: "Node type X is not installed"
```
FIX:
1. Settings → Community Nodes
2. Search for missing node
3. Click Install
4. Restart N8N
5. Retry workflow
```

### Message: "Cannot find credential of type Y"
```
FIX:
1. Credentials → Create new
2. Select credential type
3. Name it (must match exactly)
4. Return to node
5. Select from dropdown
```

### Message: "This workflow has some unsaved changes"
```
FIX:
1. Click: Save (Ctrl+S)
2. Wait for save confirmation
3. Should say "Saved Successfully"
```

---

## ✅ FINAL ANSWER TO YOUR QUESTION

### Do you need to connect the two workflows?

**NO** - Here's why:

```
❌ DON'T DO THIS:
Workflow 1 (Trigger) → Webhook Call → Workflow 2

✅ DO THIS (Current Design):
Workflow 1 (Independent) → DATABASE ← Workflow 2 (Independent)
Runs every 5 min                    Runs every 30 min
Writes to database                  Reads from database
```

### Why Database Connection is Better:
1. **Decoupled**: Failures don't cascade
2. **Scalable**: Handle 10,000+ incidents
3. **Efficient**: Run at different speeds
4. **Debuggable**: Monitor each phase separately
5. **Cost-effective**: Less API calls

### What You Need to Do Now:
1. ✅ Add all missing credentials (6 total)
2. ✅ Fix nodes showing as ? or error
3. ✅ Test Workflow 1 independently
4. ✅ Test Workflow 2 independently
5. ✅ Activate both workflows
6. ✅ Monitor for 1 hour

---

## 📞 Next Steps

**If still having issues:**

1. **Tell me which nodes show errors** (screenshot with red triangles)
2. **Share the exact error message** (click node → see error detail)
3. **Confirm credentials added** (screenshot of Credentials page)
4. **N8N version** (profile → About)

**I'll help you fix the specific missing nodes!**
