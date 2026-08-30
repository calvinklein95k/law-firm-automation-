{
  "name": "Law Firm AI Automation System – Complete",
  "nodes": [
    {
      "parameters": {},
      "id": "30b4c816-6f8b-4bdd-bf15-11cc414050b9",
      "name": "Error Trigger",
      "type": "n8n-nodes-base.errorTrigger",
      "typeVersion": 1,
      "position": [
        -1312,
        256
      ],
      "notesInFlow": true,
      "notes": "Global catch-all error trigger. Assign this workflow as Error Workflow in each sub-workflow's settings."
    },
    {
      "parameters": {
        "jsCode": "\nconst exec = $json.execution;\nconst wf = $json.workflow;\nconst err = exec.error || {};\nreturn [{\n  json: {\n    alert_type: \"WORKFLOW_ERROR\",\n    workflow_name: wf.name,\n    workflow_id: wf.id,\n    execution_id: exec.id,\n    execution_url: exec.url,\n    failed_node: exec.lastNodeExecuted,\n    error_message: err.message || 'Unknown error',\n    error_name: err.name || 'Error',\n    timestamp: new Date().toISOString(),\n    alert_text: `⚠️ Workflow Error\\nWorkflow: ${wf.name}\\nFailed Node: ${exec.lastNodeExecuted}\\nError: ${err.message}\\nExecution: ${exec.url}`\n  }\n}];\n"
      },
      "id": "5dd8d116-5494-47fb-b589-8321eb46f9f9",
      "name": "Format Error Alert",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        -1088,
        256
      ],
      "notesInFlow": true,
      "notes": "Formats error data into a structured alert payload."
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://api.telegram.org/bot{{$vars.TELEGRAM_BOT_TOKEN}}/sendMessage",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={{ JSON.stringify({ chat_id: $vars.TELEGRAM_ALERT_CHAT_ID, text: $json.alert_text, parse_mode: 'HTML' }) }}",
        "options": {}
      },
      "id": "6c037116-b090-471e-ad8b-391929dbede1",
      "name": "Send Error to Telegram",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4,
      "position": [
        -864,
        256
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "notes": "Sends error alert to Telegram. Configure TELEGRAM_BOT_TOKEN and TELEGRAM_ALERT_CHAT_ID in n8n env vars."
    },
    {
      "parameters": {
        "httpMethod": "POST",
        "path": "law-intake",
        "responseMode": "responseNode",
        "options": {}
      },
      "id": "f1655eef-3db5-47db-b71f-f116acef1476",
      "name": "M1 – Intake Webhook",
      "type": "n8n-nodes-base.webhook",
      "typeVersion": 2,
      "position": [
        -1312,
        816
      ],
      "notesInFlow": true,
      "webhookId": "09efae20-9273-4a27-8371-fd46388acf5e",
      "notes": "Receives intake submissions from website forms, WhatsApp API, or any HTTP source. POST body must include: name, phone, email, case_category, location, issue_summary, urgency, preferred_time, source."
    },
    {
      "parameters": {
        "jsCode": "\nconst b = $json.body || $json;\nconst required = ['name','phone','email','issue_summary'];\nconst missing = required.filter(f => !b[f] || String(b[f]).trim() === '');\nif (missing.length) {\n  throw new Error('VALIDATION_ERROR: Missing required fields: ' + missing.join(', '));\n}\nconst phone = String(b.phone).replace(/\\D/g,'');\nif (phone.length < 10) throw new Error('VALIDATION_ERROR: Invalid phone number');\nreturn [{json: {\n  name: String(b.name).trim(),\n  phone: phone,\n  email: String(b.email || '').trim().toLowerCase(),\n  case_category: String(b.case_category || 'General Enquiry').trim(),\n  location: String(b.location || '').trim(),\n  issue_summary: String(b.issue_summary).trim(),\n  urgency: String(b.urgency || 'Normal').trim(),\n  preferred_time: String(b.preferred_time || '').trim(),\n  source: String(b.source || 'website').trim(),\n  intake_id: 'INT-' + Date.now(),\n  received_at: new Date().toISOString()\n}}];\n"
      },
      "id": "fe84a75d-7360-4f48-aef9-8490a90865c1",
      "name": "M1 – Validate Intake Input",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        -1088,
        816
      ],
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "Validates required intake fields. Throws VALIDATION_ERROR for invalid inputs."
    },
    {
      "parameters": {
        "respondWith": "json",
        "responseBody": "={{ JSON.stringify({ error: 'VALIDATION_ERROR', message: $json.error.message.replace('VALIDATION_ERROR: ','') }) }}",
        "options": {}
      },
      "id": "11d3703b-b39d-49b4-9710-322903ad9e61",
      "name": "M1 – Intake Validation Error Response",
      "type": "n8n-nodes-base.respondToWebhook",
      "typeVersion": 1,
      "position": [
        -800,
        912
      ],
      "notesInFlow": true,
      "notes": "Returns 400 for invalid intake submissions."
    },
    {
      "parameters": {
        "model": "llama-3.3-70b-versatile",
        "options": {
          "temperature": 0.1
        }
      },
      "id": "4cdd7075-c79b-4481-af38-420dfad4bc0d",
      "name": "M1 – Groq LLM (Classify)",
      "type": "@n8n/n8n-nodes-langchain.lmChatOpenAi",
      "typeVersion": 1,
      "position": [
        -800,
        736
      ],
      "notesInFlow": true,
      "notes": "Groq LLM via OpenAI-compatible credential. Set base URL to api.groq.com/openai/v1 in credential."
    },
    {
      "parameters": {
        "prompt": "You are a legal intake classification assistant for a professional law firm. Analyze the following client enquiry and return a JSON object ONLY — no prose, no markdown fences.\n\nClient Information:\nName: {{ $json.name }}\nCase Category: {{ $json.case_category }}\nLocation: {{ $json.location }}\nIssue Summary: {{ $json.issue_summary }}\nUrgency: {{ $json.urgency }}\nSource: {{ $json.source }}\n\nReturn exactly this JSON:\n{\n  \"legal_matter_type\": \"<one of: Criminal, Civil, Family, Property, Corporate, Employment, Consumer, Intellectual Property, Immigration, Taxation, Other>\",\n  \"sub_category\": \"<specific sub-type e.g. Divorce, NDA, FIR, etc.>\",\n  \"qualification_score\": \"<Hot|Warm|Cold>\",\n  \"qualification_reason\": \"<1-2 sentence reason>\",\n  \"urgency_level\": \"<High|Medium|Low>\",\n  \"estimated_complexity\": \"<Simple|Moderate|Complex>\",\n  \"recommended_lawyer_type\": \"<e.g. Criminal Defense Advocate, Corporate Lawyer, etc.>\",\n  \"key_facts\": [\"<fact 1>\", \"<fact 2>\"],\n  \"missing_info\": [\"<missing item 1 if any>\"],\n  \"intake_risk_flags\": [\"<flag if any, e.g. urgent court date, criminal matter, etc.>\"],\n  \"ai_disclaimer\": \"This classification is for administrative intake purposes only and does not constitute legal advice.\"\n}\n\nIMPORTANT: Never provide legal advice. Never make legal judgments. This is an administrative classification only."
      },
      "id": "7d76ccdb-14f9-4f4a-aa64-8e82cb797986",
      "name": "M1 – Classify & Qualify Lead",
      "type": "@n8n/n8n-nodes-langchain.chainLlm",
      "typeVersion": 1,
      "position": [
        -864,
        512
      ],
      "notesInFlow": true,
      "notes": "Uses Groq LLaMA to classify the legal matter and qualify the lead. Output is in $json.text."
    },
    {
      "parameters": {
        "jsCode": "\nconst prev = $('M1 – Validate Intake Input').item.json;\nlet classification = {};\ntry {\n  const raw = $json.text || '{}';\n  const cleaned = raw.replace(/```json/g,'').replace(/```/g,'').trim();\n  classification = JSON.parse(cleaned);\n} catch(e) {\n  classification = {\n    legal_matter_type: 'Other',\n    sub_category: 'General',\n    qualification_score: 'Warm',\n    qualification_reason: 'Auto-classified due to parse error',\n    urgency_level: prev.urgency === 'High' ? 'High' : 'Medium',\n    estimated_complexity: 'Moderate',\n    recommended_lawyer_type: 'General Advocate',\n    key_facts: [],\n    missing_info: [],\n    intake_risk_flags: [],\n    ai_disclaimer: 'Classification failed — manual review required.'\n  };\n}\nreturn [{json: {\n  ...prev,\n  ...classification,\n  classification_timestamp: new Date().toISOString()\n}}];\n"
      },
      "id": "c228761c-93b1-45e9-94f2-1b4e83805516",
      "name": "M1 – Parse Classification JSON",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        -512,
        608
      ],
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "Parses LLM JSON output with fallback for malformed responses."
    },
    {
      "parameters": {
        "operation": "appendOrUpdate",
        "documentId": {
          "__rl": true,
          "mode": "id",
          "value": "REPLACE_GOOGLE_SHEET_ID_CRM"
        },
        "sheetName": {
          "__rl": true,
          "mode": "name",
          "value": "Leads"
        },
        "columns": {
          "mappingMode": "defineBelow",
          "value": {
            "Intake ID": "={{ $json.intake_id }}",
            "Received At": "={{ $json.received_at }}",
            "Source": "={{ $json.source }}",
            "Name": "={{ $json.name }}",
            "Phone": "={{ $json.phone }}",
            "Email": "={{ $json.email }}",
            "Case Category": "={{ $json.case_category }}",
            "Location": "={{ $json.location }}",
            "Issue Summary": "={{ $json.issue_summary }}",
            "Urgency": "={{ $json.urgency }}",
            "Preferred Time": "={{ $json.preferred_time }}",
            "Legal Matter Type": "={{ $json.legal_matter_type }}",
            "Sub Category": "={{ $json.sub_category }}",
            "Qualification Score": "={{ $json.qualification_score }}",
            "Qualification Reason": "={{ $json.qualification_reason }}",
            "Urgency Level": "={{ $json.urgency_level }}",
            "Complexity": "={{ $json.estimated_complexity }}",
            "Recommended Lawyer": "={{ $json.recommended_lawyer_type }}",
            "Status": "New",
            "Assigned To": "",
            "Case ID": ""
          },
          "schema": [
            {
              "id": "Intake ID",
              "displayName": "Intake ID",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": true
            },
            {
              "id": "Received At",
              "displayName": "Received At",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Source",
              "displayName": "Source",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Name",
              "displayName": "Name",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Phone",
              "displayName": "Phone",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Email",
              "displayName": "Email",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Case Category",
              "displayName": "Case Category",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Location",
              "displayName": "Location",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Issue Summary",
              "displayName": "Issue Summary",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Urgency",
              "displayName": "Urgency",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Preferred Time",
              "displayName": "Preferred Time",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Legal Matter Type",
              "displayName": "Legal Matter Type",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Sub Category",
              "displayName": "Sub Category",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Qualification Score",
              "displayName": "Qualification Score",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Qualification Reason",
              "displayName": "Qualification Reason",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Urgency Level",
              "displayName": "Urgency Level",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Complexity",
              "displayName": "Complexity",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Recommended Lawyer",
              "displayName": "Recommended Lawyer",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Status",
              "displayName": "Status",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Assigned To",
              "displayName": "Assigned To",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Case ID",
              "displayName": "Case ID",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            }
          ]
        },
        "options": {}
      },
      "id": "756fe934-2bc4-4270-917b-8a134ba5f36f",
      "name": "M1 – Store Lead in CRM Sheet",
      "type": "n8n-nodes-base.googleSheets",
      "typeVersion": 4,
      "position": [
        -288,
        608
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "Appends new lead to Google Sheets CRM tab 'Leads'. Configure REPLACE_GOOGLE_SHEET_ID_CRM."
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://api.telegram.org/bot{{$vars.TELEGRAM_BOT_TOKEN}}/sendMessage",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={{ JSON.stringify({\n  chat_id: $vars.LAWYER_TELEGRAM_CHAT_ID,\n  text: `🔔 NEW LEAD — ${$json.qualification_score.toUpperCase()}\\n\\n👤 ${$json.name}\\n📞 ${$json.phone}\\n📧 ${$json.email}\\n\\n⚖️ Matter: ${$json.legal_matter_type} › ${$json.sub_category}\\n🚨 Urgency: ${$json.urgency_level}\\n📍 Location: ${$json.location}\\n\\n📝 Issue:\\n${$json.issue_summary}\\n\\n🤖 Recommended Lawyer: ${$json.recommended_lawyer_type}\\n📋 Intake ID: ${$json.intake_id}\\n\\n⚠️ AI classification only — not legal advice.`,\n  parse_mode: 'HTML'\n}) }}",
        "options": {}
      },
      "id": "c00e2554-5c8a-44ad-b2ea-e75b68a2487c",
      "name": "M1 – Notify Lawyer via Telegram",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4,
      "position": [
        -64,
        480
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "Sends Telegram notification to lawyer/team. Set LAWYER_TELEGRAM_CHAT_ID in env vars."
    },
    {
      "parameters": {
        "subject": "=New {{ $json.qualification_score }} Lead: {{ $json.legal_matter_type }} – {{ $json.name }}",
        "message": "=<h2>New Client Intake – {{ $json.qualification_score }} Lead</h2>\n<table border=\"1\" cellpadding=\"6\">\n<tr><th>Intake ID</th><td>{{ $json.intake_id }}</td></tr>\n<tr><th>Name</th><td>{{ $json.name }}</td></tr>\n<tr><th>Phone</th><td>{{ $json.phone }}</td></tr>\n<tr><th>Email</th><td>{{ $json.email }}</td></tr>\n<tr><th>Legal Matter</th><td>{{ $json.legal_matter_type }} › {{ $json.sub_category }}</td></tr>\n<tr><th>Urgency</th><td>{{ $json.urgency_level }}</td></tr>\n<tr><th>Location</th><td>{{ $json.location }}</td></tr>\n<tr><th>Preferred Time</th><td>{{ $json.preferred_time }}</td></tr>\n<tr><th>Recommended Lawyer</th><td>{{ $json.recommended_lawyer_type }}</td></tr>\n</table>\n<h3>Issue Summary</h3><p>{{ $json.issue_summary }}</p>\n<h3>AI Classification Notes</h3><p>{{ $json.qualification_reason }}</p>\n<p><em>⚠️ AI-generated classification for administrative purposes only. Not legal advice.</em></p>",
        "options": {}
      },
      "id": "988c58b6-5fe2-4729-87fc-78766efc6ed4",
      "name": "M1 – Notify Lawyer via Email",
      "type": "n8n-nodes-base.gmail",
      "typeVersion": 2,
      "position": [
        -64,
        672
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "webhookId": "f25b706d-7bb2-4ddb-a34b-5831c5479996",
      "onError": "continueErrorOutput",
      "notes": "Sends email notification to lawyer. Configure LAWYER_EMAIL in n8n env vars."
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/{{$vars.WHATSAPP_PHONE_NUMBER_ID}}/messages",
        "sendHeaders": true,
        "headerParameters": {
          "parameters": [
            {
              "name": "Authorization",
              "value": "=Bearer {{$vars.WHATSAPP_ACCESS_TOKEN}}"
            }
          ]
        },
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={{ JSON.stringify({\n  messaging_product: \"whatsapp\",\n  to: $json.phone,\n  type: \"text\",\n  text: {\n    body: `Dear ${$json.name},\\n\\nThank you for reaching out to our law firm. We have received your enquiry (Ref: ${$json.intake_id}) regarding ${$json.legal_matter_type}.\\n\\nOur team will review your case and contact you within 24 hours.\\n\\nIf this is urgent, please call: ${$vars.FIRM_PHONE}\\n\\nRegards,\\n${$vars.FIRM_NAME}`\n  }\n}) }}",
        "options": {}
      },
      "id": "0f4af67a-e991-4820-aaba-4a2088e9cba6",
      "name": "M1 – Send WhatsApp Ack to Client",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4,
      "position": [
        -64,
        864
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "Sends WhatsApp acknowledgement to client. Configure WHATSAPP_PHONE_NUMBER_ID, WHATSAPP_ACCESS_TOKEN, FIRM_PHONE, FIRM_NAME in env vars."
    },
    {
      "parameters": {
        "respondWith": "json",
        "responseBody": "={{ JSON.stringify({ success: true, intake_id: $('M1 – Parse Classification JSON').item.json.intake_id, message: 'Intake received. Our team will contact you within 24 hours.', qualification: $('M1 – Parse Classification JSON').item.json.qualification_score }) }}",
        "options": {}
      },
      "id": "345c944a-70d9-470f-bb1f-3ea08c32a6c3",
      "name": "M1 – Intake Success Response",
      "type": "n8n-nodes-base.respondToWebhook",
      "typeVersion": 1,
      "position": [
        160,
        480
      ],
      "notesInFlow": true,
      "notes": "Returns 200 with intake ID to the submitter."
    },
    {
      "parameters": {
        "httpMethod": "POST",
        "path": "sarvam-voice-callback",
        "responseMode": "responseNode",
        "options": {}
      },
      "id": "72c1cf9d-a9f4-465b-a7aa-cb54f8b92174",
      "name": "M2 – Voice Webhook (Sarvam Callback)",
      "type": "n8n-nodes-base.webhook",
      "typeVersion": 2,
      "position": [
        -1312,
        1744
      ],
      "notesInFlow": true,
      "webhookId": "0fc530d6-255a-40eb-aa45-ac3bf433c3cf",
      "notes": "Receives callbacks from Sarvam AI voice calls. Sarvam posts conversation data here after each call session."
    },
    {
      "parameters": {
        "respondWith": "json",
        "responseBody": "{\"status\":\"received\"}",
        "options": {}
      },
      "id": "83baf019-3c5b-4b5d-9ad9-a8ba84278e64",
      "name": "M2 – Ack Sarvam 200",
      "type": "n8n-nodes-base.respondToWebhook",
      "typeVersion": 1,
      "position": [
        -1088,
        1648
      ],
      "notesInFlow": true,
      "notes": "Immediately ACKs Sarvam webhook. Processing continues asynchronously."
    },
    {
      "parameters": {
        "jsCode": "\nconst b = $json.body || $json;\n// Sarvam callback structure (adapt to actual Sarvam response schema)\nconst transcript = b.transcript || b.conversation_transcript || '';\nconst callData = {\n  call_id: b.call_id || b.session_id || 'CALL-' + Date.now(),\n  caller_phone: b.from || b.caller_number || '',\n  call_duration_sec: b.duration || 0,\n  language_detected: b.language || 'en-IN',\n  transcript: transcript,\n  call_status: b.status || 'completed',\n  call_timestamp: b.timestamp || new Date().toISOString(),\n  // Fields to be extracted by AI below\n  client_name: b.extracted?.name || '',\n  client_email: b.extracted?.email || '',\n  case_summary: b.extracted?.case_summary || transcript.substring(0, 500),\n  preferred_time: b.extracted?.preferred_time || '',\n  urgency: b.extracted?.urgency || 'Normal',\n  appointment_requested: b.extracted?.appointment_requested || false\n};\nreturn [{json: callData}];\n"
      },
      "id": "c60f9c5b-297a-4a89-9917-60bb59ed06c8",
      "name": "M2 – Parse Voice Call Data",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        -1088,
        1840
      ],
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "Parses Sarvam AI voice callback. Adapt field mappings to match your Sarvam plan's actual webhook schema."
    },
    {
      "parameters": {
        "model": "llama-3.3-70b-versatile",
        "options": {
          "temperature": 0.1
        }
      },
      "id": "c9471bca-6011-4f64-97d8-43d382b20264",
      "name": "M2 – Groq LLM (Voice Extract)",
      "type": "@n8n/n8n-nodes-langchain.lmChatOpenAi",
      "typeVersion": 1,
      "position": [
        -800,
        2064
      ],
      "notesInFlow": true,
      "notes": "Groq LLM for extracting structured data from voice transcript."
    },
    {
      "parameters": {
        "prompt": "You are a legal receptionist AI assistant. Extract structured client information from this voice call transcript. Return JSON ONLY.\n\nCall Transcript:\n{{ $json.transcript }}\n\nCaller Phone: {{ $json.caller_phone }}\nLanguage: {{ $json.language_detected }}\n\nReturn this exact JSON:\n{\n  \"client_name\": \"<extracted or 'Unknown'>\",\n  \"client_email\": \"<extracted or ''>\",\n  \"case_category\": \"<one of: Criminal, Civil, Family, Property, Corporate, Employment, Consumer, Other>\",\n  \"issue_summary\": \"<2-3 sentence summary of the legal issue discussed>\",\n  \"urgency\": \"<High|Normal|Low>\",\n  \"preferred_consultation_time\": \"<extracted time preference or ''>\",\n  \"appointment_requested\": <true|false>,\n  \"language_spoken\": \"<language identified>\",\n  \"key_points\": [\"<point 1>\", \"<point 2>\"],\n  \"follow_up_required\": <true|false>,\n  \"follow_up_notes\": \"<any specific follow-up needed>\",\n  \"ai_disclaimer\": \"Extracted from voice call for administrative purposes only. Not legal advice.\"\n}\n\nIMPORTANT: Do not provide legal advice. Do not make legal judgments. Administrative extraction only."
      },
      "id": "061ca1ef-9606-47e5-ab3a-6794715775e4",
      "name": "M2 – Extract Client Info from Transcript",
      "type": "@n8n/n8n-nodes-langchain.chainLlm",
      "typeVersion": 1,
      "position": [
        -864,
        1840
      ],
      "notesInFlow": true,
      "notes": "Extracts structured client data from voice transcript using Groq LLaMA."
    },
    {
      "parameters": {
        "jsCode": "\nconst voiceData = $('M2 – Parse Voice Call Data').item.json;\nlet extracted = {};\ntry {\n  const raw = $json.text || '{}';\n  extracted = JSON.parse(raw.replace(/```json/g,'').replace(/```/g,'').trim());\n} catch(e) {\n  extracted = { client_name: 'Unknown', issue_summary: voiceData.transcript?.substring(0,300) || '' };\n}\nreturn [{json: {\n  ...voiceData,\n  ...extracted,\n  // Merge: AI extraction overrides raw parse where available\n  client_name: extracted.client_name || voiceData.client_name || 'Unknown',\n  case_category: extracted.case_category || 'General Enquiry',\n  issue_summary: extracted.issue_summary || voiceData.case_summary,\n  urgency: extracted.urgency || voiceData.urgency,\n  preferred_time: extracted.preferred_consultation_time || voiceData.preferred_time,\n  appointment_requested: extracted.appointment_requested || voiceData.appointment_requested,\n  source: 'voice_call',\n  intake_id: 'CALL-' + voiceData.call_id\n}}];\n"
      },
      "id": "473b2988-8ece-4919-b646-fc6314d45360",
      "name": "M2 – Merge Voice + AI Extraction",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        -512,
        1840
      ],
      "notesInFlow": true,
      "notes": "Merges raw voice call data with AI-extracted structured fields."
    },
    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": false,
            "leftValue": "",
            "typeValidation": "strict"
          },
          "conditions": [
            {
              "id": "e23801f3-5a0c-4609-9849-c2fc65d3eb1c",
              "leftValue": "={{ String($json.appointment_requested) }}",
              "rightValue": "true",
              "operator": {
                "type": "string",
                "operation": "equals"
              }
            }
          ],
          "combinator": "and"
        },
        "options": {}
      },
      "id": "3c7294a7-9662-4df9-86e0-9ea5f86cef59",
      "name": "M2 – Appointment Requested?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2,
      "position": [
        -288,
        1840
      ],
      "notesInFlow": true,
      "notes": "Routes to appointment booking if client requested one during voice call."
    },
    {
      "parameters": {
        "operation": "appendOrUpdate",
        "documentId": {
          "__rl": true,
          "mode": "id",
          "value": "REPLACE_GOOGLE_SHEET_ID_CRM"
        },
        "sheetName": {
          "__rl": true,
          "mode": "name",
          "value": "Leads"
        },
        "columns": {
          "mappingMode": "defineBelow",
          "value": {
            "Intake ID": "={{ $json.intake_id }}",
            "Received At": "={{ $json.call_timestamp }}",
            "Source": "voice_call",
            "Name": "={{ $json.client_name }}",
            "Phone": "={{ $json.caller_phone }}",
            "Email": "={{ $json.client_email }}",
            "Case Category": "={{ $json.case_category }}",
            "Issue Summary": "={{ $json.issue_summary }}",
            "Urgency": "={{ $json.urgency }}",
            "Preferred Time": "={{ $json.preferred_time }}",
            "Status": "New – Voice",
            "Legal Matter Type": "",
            "Sub Category": "",
            "Qualification Score": "Warm"
          },
          "schema": [
            {
              "id": "Intake ID",
              "displayName": "Intake ID",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": true
            },
            {
              "id": "Received At",
              "displayName": "Received At",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Source",
              "displayName": "Source",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Name",
              "displayName": "Name",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Phone",
              "displayName": "Phone",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Email",
              "displayName": "Email",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Case Category",
              "displayName": "Case Category",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Issue Summary",
              "displayName": "Issue Summary",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Urgency",
              "displayName": "Urgency",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Preferred Time",
              "displayName": "Preferred Time",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Status",
              "displayName": "Status",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Legal Matter Type",
              "displayName": "Legal Matter Type",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Sub Category",
              "displayName": "Sub Category",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Qualification Score",
              "displayName": "Qualification Score",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            }
          ]
        },
        "options": {}
      },
      "id": "0355c115-8d90-476b-97ce-df34b805fd1d",
      "name": "M2 – Store Voice Lead in CRM",
      "type": "n8n-nodes-base.googleSheets",
      "typeVersion": 4,
      "position": [
        -64,
        1744
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "Stores voice call lead in CRM. Both appointment and non-appointment paths converge here."
    },
    {
      "parameters": {
        "operation": "getAll",
        "calendar": {
          "__rl": true,
          "mode": "id",
          "value": "REPLACE_GOOGLE_CALENDAR_ID"
        },
        "options": {}
      },
      "id": "6322965e-53a8-4d20-bb94-91141abcf177",
      "name": "M2 – Check Calendar Availability",
      "type": "n8n-nodes-base.googleCalendar",
      "typeVersion": 1,
      "position": [
        -64,
        1936
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "Fetches existing calendar events to check lawyer availability. Configure REPLACE_GOOGLE_CALENDAR_ID."
    },
    {
      "parameters": {
        "calendar": {
          "__rl": true,
          "mode": "id",
          "value": "REPLACE_GOOGLE_CALENDAR_ID"
        },
        "start": "={{ $now.plus({days:1}).set({hour:10,minute:0,second:0}).toISO() }}",
        "end": "={{ $now.plus({days:1}).set({hour:11,minute:0,second:0}).toISO() }}",
        "additionalFields": {
          "attendees": "={{ $('M2 – Merge Voice + AI Extraction').item.json.client_email ? [$('M2 – Merge Voice + AI Extraction').item.json.client_email] : [] }}",
          "description": "=Client: {{ $('M2 – Merge Voice + AI Extraction').item.json.client_name }}\nPhone: {{ $('M2 – Merge Voice + AI Extraction').item.json.caller_phone }}\nCase: {{ $('M2 – Merge Voice + AI Extraction').item.json.issue_summary }}\n\nIntake ID: {{ $('M2 – Merge Voice + AI Extraction').item.json.intake_id }}",
          "summary": "=Consultation: {{ $('M2 – Merge Voice + AI Extraction').item.json.client_name }} – {{ $('M2 – Merge Voice + AI Extraction').item.json.case_category }}"
        }
      },
      "id": "06f004bb-9401-41e2-a76c-cc2a5e830a67",
      "name": "M2 – Create Appointment in Calendar",
      "type": "n8n-nodes-base.googleCalendar",
      "typeVersion": 1,
      "position": [
        160,
        1936
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "Creates calendar appointment. In production, implement availability-check logic before booking."
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/{{$vars.WHATSAPP_PHONE_NUMBER_ID}}/messages",
        "sendHeaders": true,
        "headerParameters": {
          "parameters": [
            {
              "name": "Authorization",
              "value": "=Bearer {{$vars.WHATSAPP_ACCESS_TOKEN}}"
            }
          ]
        },
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={{ JSON.stringify({\n  messaging_product: \"whatsapp\",\n  to: $('M2 – Merge Voice + AI Extraction').item.json.caller_phone,\n  type: \"text\",\n  text: {\n    body: `Dear ${$('M2 – Merge Voice + AI Extraction').item.json.client_name},\\n\\nYour consultation has been scheduled!\\n\\n📅 Date: ${$json.start?.split('T')[0] || 'TBD'}\\n⏰ Time: 10:00 AM\\n📍 ${$vars.FIRM_ADDRESS}\\n\\nPlease bring any relevant documents.\\n\\nRef: ${$('M2 – Merge Voice + AI Extraction').item.json.intake_id}\\n\\n${$vars.FIRM_NAME}`\n  }\n}) }}",
        "options": {}
      },
      "id": "e139395a-bc7b-4969-b5a0-2704e478ca2f",
      "name": "M2 – Send Appointment Confirmation WA",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4,
      "position": [
        384,
        1936
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "Sends WhatsApp appointment confirmation to client."
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://api.sarvam.ai/v1/phone-calls",
        "sendHeaders": true,
        "headerParameters": {
          "parameters": [
            {
              "name": "api-subscription-key",
              "value": "={{ $vars.SARVAM_API_KEY }}"
            },
            {
              "name": "Content-Type",
              "value": "application/json"
            }
          ]
        },
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={{ JSON.stringify({\n  agent_id: $vars.SARVAM_AGENT_ID,\n  phone_number: $json.phone,\n  metadata: {\n    client_name: $json.name,\n    case_ref: $json.intake_id,\n    callback_url: $vars.N8N_WEBHOOK_BASE + '/webhook/sarvam-voice-callback'\n  }\n}) }}",
        "options": {}
      },
      "id": "10018e45-c881-4aa0-91e7-f72275b62382",
      "name": "M2 – Initiate Sarvam Outbound Call",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4,
      "position": [
        -1312,
        2592
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "Initiates outbound Sarvam AI call. Configure SARVAM_API_KEY, SARVAM_AGENT_ID in env vars. Sarvam agent must be pre-configured in Sarvam dashboard with multilingual support (Hindi, Tamil, Telugu, etc.). N8N_WEBHOOK_BASE = your ngrok/production URL."
    },
    {
      "parameters": {
        "httpMethod": "POST",
        "path": "law-document-upload",
        "responseMode": "responseNode",
        "options": {}
      },
      "id": "ef22017e-8f20-445d-9063-8af95dd15a2a",
      "name": "M3 – Document Upload Webhook",
      "type": "n8n-nodes-base.webhook",
      "typeVersion": 2,
      "position": [
        -1312,
        2912
      ],
      "notesInFlow": true,
      "webhookId": "da493c59-7216-4bb9-bfd3-84a007752fb5",
      "notes": "Receives document uploads. Accepts multipart/form-data with file + metadata (intake_id, doc_type, client_name, case_category). Supports PDF, DOCX, TXT."
    },
    {
      "parameters": {
        "respondWith": "json",
        "responseBody": "{\"status\":\"received\",\"message\":\"Document received. Processing will complete within 2-3 minutes.\"}",
        "options": {}
      },
      "id": "9e91b23f-6376-45ff-b1f8-4dab2ca15fca",
      "name": "M3 – Ack Document Upload",
      "type": "n8n-nodes-base.respondToWebhook",
      "typeVersion": 1,
      "position": [
        -1088,
        2816
      ],
      "notesInFlow": true,
      "notes": "Immediately ACKs document upload webhook."
    },
    {
      "parameters": {
        "jsCode": "\n// Extract metadata from form fields\nconst body = $json.body || $json;\nconst binaryData = $binary;\n\n// Get file info\nconst fileKey = Object.keys(binaryData || {})[0] || 'data';\nconst file = binaryData[fileKey] || {};\n\nreturn [{\n  json: {\n    doc_id: 'DOC-' + Date.now(),\n    intake_id: body.intake_id || '',\n    client_name: body.client_name || 'Unknown',\n    case_category: body.case_category || 'General',\n    doc_type: body.doc_type || 'Unknown',\n    original_filename: file.fileName || 'document',\n    mime_type: file.mimeType || 'application/octet-stream',\n    file_size: file.fileSize || 0,\n    upload_timestamp: new Date().toISOString(),\n    extraction_note: 'Text extraction via external OCR API or n8n Extract from File node required. Configure EXTRACT_FROM_FILE_NODE or OCR_API_URL.'\n  },\n  binary: binaryData\n}];\n"
      },
      "id": "1dfc42d7-16fc-45d0-b9f8-8117b3cb9148",
      "name": "M3 – Extract Document Text",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        -1088,
        3008
      ],
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "Prepares document metadata. Binary file is passed through for OCR/extraction. For PDF text extraction, use n8n Extract from File node or connect to OCR API (e.g., Google Document AI, Textract)."
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://documentai.googleapis.com/v1/projects/{{$vars.GOOGLE_PROJECT_ID}}/locations/us/processors/{{$vars.DOCUMENT_AI_PROCESSOR_ID}}:process",
        "sendHeaders": true,
        "headerParameters": {
          "parameters": [
            {
              "name": "Authorization",
              "value": "=Bearer {{$vars.GOOGLE_DOCUMENT_AI_TOKEN}}"
            }
          ]
        },
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={{ JSON.stringify({\n  rawDocument: {\n    content: $binary.data?.data || '',\n    mimeType: $json.mime_type || 'application/pdf'\n  }\n}) }}",
        "options": {
          "response": {
            "response": {}
          }
        }
      },
      "id": "05e4493b-fdc7-4b86-8471-965fdb78456c",
      "name": "M3 – OCR / Text Extraction API",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4,
      "position": [
        -864,
        3008
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "PLACEHOLDER: Google Document AI OCR. Replace with your OCR provider (AWS Textract, Mistral OCR API, or n8n Extract from File node for simple PDFs). Configure GOOGLE_PROJECT_ID, DOCUMENT_AI_PROCESSOR_ID, GOOGLE_DOCUMENT_AI_TOKEN. For simple text PDFs, use Extract from File node instead."
    },
    {
      "parameters": {
        "jsCode": "\nconst meta = $('M3 – Extract Document Text').item.json;\n// Try to extract text from Document AI response\nlet extractedText = '';\ntry {\n  const resp = $json;\n  extractedText = resp?.document?.text || resp?.text || resp?.content || '';\n  if (!extractedText && resp?.pages) {\n    extractedText = resp.pages.map(p => p.paragraphs?.map(para => \n      para.layout?.textAnchor?.textSegments?.map(seg => \n        resp.document?.text?.substring(seg.startIndex || 0, seg.endIndex || 0)\n      ).join(' ')\n    ).join('\\n')).join('\\n');\n  }\n} catch(e) {\n  extractedText = 'EXTRACTION_FAILED: ' + e.message;\n}\n\n// Truncate for LLM context window (keep ~8000 chars)\nconst truncated = extractedText.substring(0, 8000);\nconst wasTruncated = extractedText.length > 8000;\n\nreturn [{json: {\n  ...meta,\n  extracted_text: truncated,\n  was_truncated: wasTruncated,\n  text_length: extractedText.length,\n  extraction_success: !extractedText.startsWith('EXTRACTION_FAILED')\n}}];\n"
      },
      "id": "2a7e7a39-3ec0-47ca-95b2-9f21f2642775",
      "name": "M3 – Prepare Extracted Text",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        -640,
        3008
      ],
      "notesInFlow": true,
      "notes": "Extracts text from OCR API response. Truncates to 8000 chars for LLM context window."
    },
    {
      "parameters": {
        "model": "llama-3.3-70b-versatile",
        "options": {
          "maxTokens": 2048,
          "temperature": 0.1
        }
      },
      "id": "c4b741d7-eb08-4978-9bdc-b1b54d68b395",
      "name": "M3 – Groq LLM (Summarize)",
      "type": "@n8n/n8n-nodes-langchain.lmChatOpenAi",
      "typeVersion": 1,
      "position": [
        -352,
        3232
      ],
      "notesInFlow": true,
      "notes": "Groq LLaMA for legal document summarization."
    },
    {
      "parameters": {
        "prompt": "You are a legal document analysis assistant. Analyze the following legal document and return a structured JSON summary ONLY — no prose, no markdown fences.\n\nDocument Type: {{ $json.doc_type }}\nCase Category: {{ $json.case_category }}\nClient: {{ $json.client_name }}\n\nDocument Text:\n{{ $json.extracted_text }}\n\n{{ $json.was_truncated ? '⚠️ Note: Document was truncated due to length. Analysis is based on first 8000 characters.' : '' }}\n\nReturn this exact JSON:\n{\n  \"document_type\": \"<identified document type>\",\n  \"parties_involved\": [{\"role\": \"<Party Role>\", \"name\": \"<Party Name>\"}],\n  \"case_type\": \"<legal matter type>\",\n  \"key_facts\": [\"<fact 1>\", \"<fact 2>\", \"<fact 3>\"],\n  \"important_dates\": [{\"event\": \"<event description>\", \"date\": \"<date>\"}],\n  \"deadlines\": [{\"deadline\": \"<description>\", \"date\": \"<date>\", \"urgency\": \"<High|Medium|Low>\"}],\n  \"required_actions\": [\"<action 1>\", \"<action 2>\"],\n  \"missing_information\": [\"<missing item if any>\"],\n  \"missing_documents\": [\"<document needed if any>\"],\n  \"document_summary\": \"<2-3 paragraph plain-language summary of the document>\",\n  \"complexity_assessment\": \"<Simple|Moderate|Complex>\",\n  \"immediate_attention_required\": <true|false>,\n  \"attention_reason\": \"<reason if immediate attention required, else ''>\",\n  \"ai_disclaimer\": \"This summary is AI-generated for administrative review purposes only and does not constitute legal advice. All findings must be reviewed and verified by a qualified lawyer.\"\n}\n\nIMPORTANT: Do not provide legal advice. This is for administrative document processing only."
      },
      "id": "1216e5d8-777c-463f-80a7-3faefe748471",
      "name": "M3 – AI Document Summarization",
      "type": "@n8n/n8n-nodes-langchain.chainLlm",
      "typeVersion": 1,
      "position": [
        -416,
        3008
      ],
      "notesInFlow": true,
      "notes": "Generates structured legal document summary using Groq LLaMA."
    },
    {
      "parameters": {
        "jsCode": "\nconst meta = $('M3 – Prepare Extracted Text').item.json;\nlet summary = {};\ntry {\n  const raw = $json.text || '{}';\n  summary = JSON.parse(raw.replace(/```json/g,'').replace(/```/g,'').trim());\n} catch(e) {\n  summary = {\n    document_type: meta.doc_type,\n    document_summary: 'AI summary generation failed. Manual review required.',\n    ai_disclaimer: 'Parsing failed. Manual review required.',\n    missing_information: [],\n    missing_documents: [],\n    required_actions: ['Manual review required'],\n    immediate_attention_required: false\n  };\n}\nreturn [{json: {\n  ...meta,\n  ...summary,\n  summary_generated_at: new Date().toISOString()\n}}];\n"
      },
      "id": "9631164b-16dd-4984-8faf-3dcf3212dc72",
      "name": "M3 – Parse Document Summary",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        -64,
        3008
      ],
      "notesInFlow": true,
      "notes": "Parses LLM document summary with fallback."
    },
    {
      "parameters": {
        "name": "={{ $json.original_filename }}",
        "driveId": {
          "__rl": true,
          "mode": "myDrive",
          "value": "myDrive"
        },
        "folderId": {
          "__rl": true,
          "mode": "list",
          "value": "root",
          "cachedResultName": "/ (Root folder)"
        },
        "options": {}
      },
      "id": "9f74a413-c5ec-4797-be03-6d06232b391a",
      "name": "M3 – Upload Original Doc to Drive",
      "type": "n8n-nodes-base.googleDrive",
      "typeVersion": 3,
      "position": [
        160,
        2912
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "Uploads original document to Google Drive. Configure REPLACE_GDRIVE_DOCUMENTS_FOLDER_ID."
    },
    {
      "parameters": {
        "operation": "appendOrUpdate",
        "documentId": {
          "__rl": true,
          "mode": "id",
          "value": "REPLACE_GOOGLE_SHEET_ID_CRM"
        },
        "sheetName": {
          "__rl": true,
          "mode": "name",
          "value": "Documents"
        },
        "columns": {
          "mappingMode": "defineBelow",
          "value": {
            "Doc ID": "={{ $json.doc_id }}",
            "Intake ID": "={{ $json.intake_id }}",
            "Client Name": "={{ $json.client_name }}",
            "Original Filename": "={{ $json.original_filename }}",
            "Doc Type": "={{ $json.document_type }}",
            "Case Type": "={{ $json.case_type || $json.case_category }}",
            "Upload Timestamp": "={{ $json.upload_timestamp }}",
            "Summary Generated At": "={{ $json.summary_generated_at }}",
            "Key Summary": "={{ $json.document_summary }}",
            "Missing Docs": "={{ ($json.missing_documents || []).join(', ') }}",
            "Required Actions": "={{ ($json.required_actions || []).join(', ') }}",
            "Immediate Attention": "={{ $json.immediate_attention_required ? 'YES' : 'No' }}",
            "Complexity": "={{ $json.complexity_assessment }}",
            "AI Disclaimer": "={{ $json.ai_disclaimer }}"
          },
          "schema": [
            {
              "id": "Doc ID",
              "displayName": "Doc ID",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": true
            },
            {
              "id": "Intake ID",
              "displayName": "Intake ID",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Client Name",
              "displayName": "Client Name",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Original Filename",
              "displayName": "Original Filename",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Doc Type",
              "displayName": "Doc Type",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Case Type",
              "displayName": "Case Type",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Upload Timestamp",
              "displayName": "Upload Timestamp",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Summary Generated At",
              "displayName": "Summary Generated At",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Key Summary",
              "displayName": "Key Summary",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Missing Docs",
              "displayName": "Missing Docs",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Required Actions",
              "displayName": "Required Actions",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Immediate Attention",
              "displayName": "Immediate Attention",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Complexity",
              "displayName": "Complexity",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "AI Disclaimer",
              "displayName": "AI Disclaimer",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            }
          ]
        },
        "options": {}
      },
      "id": "85476765-6fce-47b4-a297-37f288f1532b",
      "name": "M3 – Store Summary in Sheets",
      "type": "n8n-nodes-base.googleSheets",
      "typeVersion": 4,
      "position": [
        160,
        3104
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "Stores document AI summary in the Documents tab of the CRM sheet."
    },
    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": false,
            "leftValue": "",
            "typeValidation": "strict"
          },
          "conditions": [
            {
              "id": "a01e816b-a89d-42de-b114-26066935b16b",
              "leftValue": "={{ ($json.missing_documents || []).length > 0 || ($json.missing_information || []).length > 0 }}",
              "rightValue": "true",
              "operator": {
                "type": "string",
                "operation": "equals"
              }
            }
          ],
          "combinator": "and"
        },
        "options": {}
      },
      "id": "db61ca9f-bb3d-4a78-913b-05e4fc7b11b8",
      "name": "M3 – Missing Docs?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2,
      "position": [
        384,
        3104
      ],
      "notesInFlow": true,
      "notes": "Checks if AI identified missing documents or information."
    },
    {
      "parameters": {
        "subject": "=⚠️ Missing Documents Alert – {{ $json.client_name }} ({{ $json.doc_id }})",
        "message": "=<h2>Missing Documents Detected</h2>\n<p><strong>Client:</strong> {{ $json.client_name }}<br>\n<strong>Doc ID:</strong> {{ $json.doc_id }}<br>\n<strong>Intake ID:</strong> {{ $json.intake_id }}</p>\n<h3>Missing Documents</h3><ul>{{ $json.missing_documents?.map(d => `<li>${d}</li>`).join('') || '<li>None</li>' }}</ul>\n<h3>Missing Information</h3><ul>{{ $json.missing_information?.map(i => `<li>${i}</li>`).join('') || '<li>None</li>' }}</ul>\n<h3>Required Actions</h3><ul>{{ $json.required_actions?.map(a => `<li>${a}</li>`).join('') || '<li>None</li>' }}</ul>\n<h3>Document Summary</h3><p>{{ $json.document_summary }}</p>\n<p><em>⚠️ AI-generated for review purposes. Not legal advice.</em></p>",
        "options": {}
      },
      "id": "4fb551ef-a0a8-43ae-b3a2-ec46fd0eeaf6",
      "name": "M3 – Notify Lawyer – Missing Docs",
      "type": "n8n-nodes-base.gmail",
      "typeVersion": 2,
      "position": [
        608,
        3104
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "webhookId": "351e89a9-d5a4-4d12-91ba-5b01e427a5fb",
      "onError": "continueErrorOutput",
      "notes": "Notifies lawyer when AI detects missing documents or information."
    },
    {
      "parameters": {
        "rule": {
          "interval": [
            {
              "field": "cronExpression",
              "expression": "0 9 * * 1-6"
            }
          ]
        }
      },
      "id": "a907cfc5-c556-4174-9fd3-4ce0fb806dc5",
      "name": "M4 – Daily Follow-up Scheduler",
      "type": "n8n-nodes-base.scheduleTrigger",
      "typeVersion": 1,
      "position": [
        -1312,
        3952
      ],
      "notesInFlow": true,
      "notes": "Triggers daily at 9 AM Monday-Saturday to process follow-up sequences and case status updates."
    },
    {
      "parameters": {
        "operation": "getAll",
        "documentId": {
          "__rl": true,
          "mode": "id",
          "value": "REPLACE_GOOGLE_SHEET_ID_CRM"
        }
      },
      "id": "f31e8483-e0c5-447b-b670-b66332e1e781",
      "name": "M4 – Fetch Active Cases from CRM",
      "type": "n8n-nodes-base.googleSheets",
      "typeVersion": 4,
      "position": [
        -1088,
        3952
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "Fetches all cases from the Cases tab. Cases sheet columns: Case ID, Client Name, Phone, Email, Lawyer, Status, Next Hearing, Deadline, Pending Docs, Last Contacted, Follow-up Count."
    },
    {
      "parameters": {
        "jsCode": "\nconst now = new Date();\nconst today = now.toISOString().split('T')[0];\n\nconst casesNeedingFollowup = $input.all()\n  .map(item => item.json)\n  .filter(c => {\n    const status = (c['Status'] || '').toLowerCase();\n    // Skip closed/completed cases\n    if (['closed','completed','cancelled'].includes(status)) return false;\n    \n    const deadline = c['Deadline'] ? new Date(c['Deadline']) : null;\n    const nextHearing = c['Next Hearing'] ? new Date(c['Next Hearing']) : null;\n    const lastContacted = c['Last Contacted'] ? new Date(c['Last Contacted']) : null;\n    const followupCount = parseInt(c['Follow-up Count'] || '0');\n    \n    // Max 3 follow-ups per case\n    if (followupCount >= 3) return false;\n    \n    // Urgent: deadline within 7 days\n    if (deadline) {\n      const daysToDeadline = Math.ceil((deadline - now) / (1000*60*60*24));\n      if (daysToDeadline <= 7 && daysToDeadline > 0) return true;\n    }\n    \n    // Hearing within 3 days\n    if (nextHearing) {\n      const daysToHearing = Math.ceil((nextHearing - now) / (1000*60*60*24));\n      if (daysToHearing <= 3 && daysToHearing > 0) return true;\n    }\n    \n    // Not contacted in 7+ days\n    if (!lastContacted) return true;\n    const daysSinceContact = Math.ceil((now - lastContacted) / (1000*60*60*24));\n    if (daysSinceContact >= 7) return true;\n    \n    // Has pending docs and not contacted recently\n    if (c['Pending Docs'] && c['Pending Docs'] !== '' && daysSinceContact >= 3) return true;\n    \n    return false;\n  })\n  .map(c => {\n    const deadline = c['Deadline'] ? new Date(c['Deadline']) : null;\n    const nextHearing = c['Next Hearing'] ? new Date(c['Next Hearing']) : null;\n    \n    let reason = 'routine';\n    let priority = 'normal';\n    \n    if (deadline && Math.ceil((deadline - now)/(1000*60*60*24)) <= 3) {\n      reason = 'urgent_deadline'; priority = 'high';\n    } else if (nextHearing && Math.ceil((nextHearing - now)/(1000*60*60*24)) <= 3) {\n      reason = 'hearing_reminder'; priority = 'high';\n    } else if (c['Pending Docs']) {\n      reason = 'pending_documents'; priority = 'medium';\n    }\n    \n    return { ...c, followup_reason: reason, followup_priority: priority };\n  });\n\nreturn casesNeedingFollowup.map(c => ({json: c}));\n"
      },
      "id": "dd198ed8-85a8-4a92-9486-216d1c06bc37",
      "name": "M4 – Filter Cases Needing Follow-up",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        -864,
        3952
      ],
      "notesInFlow": true,
      "notes": "Filters cases requiring follow-up based on deadlines, hearings, last contact date, and pending documents."
    },
    {
      "parameters": {
        "model": "llama-3.3-70b-versatile",
        "options": {
          "temperature": 0.5
        }
      },
      "id": "01685879-cacb-420c-83ba-3c263e741407",
      "name": "M4 – Groq LLM (Follow-up Msg)",
      "type": "@n8n/n8n-nodes-langchain.lmChatOpenAi",
      "typeVersion": 1,
      "position": [
        -576,
        4176
      ],
      "notesInFlow": true,
      "notes": "Groq LLaMA for generating personalized follow-up messages."
    },
    {
      "parameters": {
        "prompt": "Generate a personalized, professional client follow-up WhatsApp message for a law firm. Return JSON ONLY.\n\nClient: {{ $json['Client Name'] }}\nCase Status: {{ $json['Status'] }}\nFollow-up Reason: {{ $json.followup_reason }}\nPriority: {{ $json.followup_priority }}\nNext Hearing: {{ $json['Next Hearing'] || 'Not scheduled' }}\nDeadline: {{ $json['Deadline'] || 'Not set' }}\nPending Documents: {{ $json['Pending Docs'] || 'None' }}\nLawyer: {{ $json['Lawyer'] || 'Our team' }}\nFirm Name: {{ $vars.FIRM_NAME }}\nFirm Phone: {{ $vars.FIRM_PHONE }}\n\nReturn this JSON:\n{\n  \"whatsapp_message\": \"<personalized WhatsApp message in simple English, max 300 chars, professional and empathetic>\",\n  \"email_subject\": \"<email subject line>\",\n  \"email_body_html\": \"<short HTML email body, professional, with case status update>\",\n  \"escalate_to_lawyer\": <true if high priority and 3rd follow-up>,\n  \"escalation_note\": \"<note for lawyer if escalating>\"\n}\n\nRules:\n- Be warm, professional, and empathetic\n- Do NOT discuss legal strategy or provide legal advice\n- Refer to upcoming dates/deadlines if present\n- For pending documents, clearly list what is needed\n- Keep WhatsApp message under 300 characters"
      },
      "id": "55bbe004-f8eb-4638-ad60-ab168e994729",
      "name": "M4 – Generate Personalized Follow-up",
      "type": "@n8n/n8n-nodes-langchain.chainLlm",
      "typeVersion": 1,
      "position": [
        -640,
        3952
      ],
      "notesInFlow": true,
      "notes": "Generates personalized follow-up message using AI. Each case gets a unique, contextual message."
    },
    {
      "parameters": {
        "jsCode": "\nconst caseData = $('M4 – Filter Cases Needing Follow-up').item.json;\nlet msg = {};\ntry {\n  const raw = $json.text || '{}';\n  msg = JSON.parse(raw.replace(/```json/g,'').replace(/```/g,'').trim());\n} catch(e) {\n  const clientName = caseData['Client Name'] || 'Client';\n  msg = {\n    whatsapp_message: `Dear ${clientName}, this is a follow-up from ${$vars.FIRM_NAME} regarding your case. Please contact us at ${$vars.FIRM_PHONE}.`,\n    email_subject: `Follow-up – Case Update`,\n    email_body_html: `<p>Dear ${clientName},<br>Please contact us for an update on your case.</p>`,\n    escalate_to_lawyer: false,\n    escalation_note: ''\n  };\n}\nreturn [{json: {...caseData, ...msg, processed_at: new Date().toISOString()}}];\n"
      },
      "id": "c06af362-3954-404b-885d-b62149ea731f",
      "name": "M4 – Parse Follow-up Message",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        -288,
        3952
      ],
      "notesInFlow": true,
      "notes": "Parses AI-generated follow-up message with fallback template."
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/{{$vars.WHATSAPP_PHONE_NUMBER_ID}}/messages",
        "sendHeaders": true,
        "headerParameters": {
          "parameters": [
            {
              "name": "Authorization",
              "value": "=Bearer {{$vars.WHATSAPP_ACCESS_TOKEN}}"
            }
          ]
        },
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={{ JSON.stringify({\n  messaging_product: \"whatsapp\",\n  to: $json['Phone'],\n  type: \"text\",\n  text: { body: $json.whatsapp_message }\n}) }}",
        "options": {}
      },
      "id": "dcebe558-2119-454a-bd86-e43fde1556c3",
      "name": "M4 – Send Follow-up WhatsApp",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4,
      "position": [
        -64,
        3856
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "Sends personalized follow-up WhatsApp message to client."
    },
    {
      "parameters": {
        "subject": "={{ $json.email_subject }}",
        "message": "={{ $json.email_body_html }}",
        "options": {}
      },
      "id": "a81cacf0-43e5-44d8-b485-33074c2ee294",
      "name": "M4 – Send Follow-up Email",
      "type": "n8n-nodes-base.gmail",
      "typeVersion": 2,
      "position": [
        -64,
        4048
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "webhookId": "30e1bce2-c09c-44d3-a215-71acf219fd68",
      "onError": "continueErrorOutput",
      "notes": "Sends follow-up email to client."
    },
    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": false,
            "leftValue": "",
            "typeValidation": "strict"
          },
          "conditions": [
            {
              "id": "23b3c36b-fd72-4ae2-8a63-94afe7335ee1",
              "leftValue": "={{ String($json.escalate_to_lawyer) }}",
              "rightValue": "true",
              "operator": {
                "type": "string",
                "operation": "equals"
              }
            }
          ],
          "combinator": "and"
        },
        "options": {}
      },
      "id": "312cc4ab-2f71-4733-80c8-3d8c12471ed4",
      "name": "M4 – Escalate to Lawyer?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2,
      "position": [
        160,
        3856
      ],
      "notesInFlow": true,
      "notes": "Routes high-priority unresponsive cases to lawyer escalation."
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://api.telegram.org/bot{{$vars.TELEGRAM_BOT_TOKEN}}/sendMessage",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={{ JSON.stringify({\n  chat_id: $vars.LAWYER_TELEGRAM_CHAT_ID,\n  text: `🚨 ESCALATION REQUIRED\\n\\nCase: ${$json['Case ID']}\\nClient: ${$json['Client Name']}\\nPhone: ${$json['Phone']}\\nStatus: ${$json['Status']}\\nPriority: ${$json.followup_priority?.toUpperCase()}\\nReason: ${$json.followup_reason}\\n\\nNote: ${$json.escalation_note}\\n\\nImmediate action required.`,\n  parse_mode: 'HTML'\n}) }}",
        "options": {}
      },
      "id": "d83fee6a-9631-4a50-829d-c8b8afb30767",
      "name": "M4 – Escalate Alert to Lawyer",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4,
      "position": [
        384,
        3856
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "Sends escalation alert to lawyer via Telegram for high-priority unresponsive cases."
    },
    {
      "parameters": {
        "operation": "update",
        "documentId": {
          "__rl": true,
          "mode": "id",
          "value": "REPLACE_GOOGLE_SHEET_ID_CRM"
        },
        "sheetName": {
          "__rl": true,
          "mode": "name",
          "value": "Cases"
        },
        "columns": {
          "mappingMode": "defineBelow",
          "value": {
            "Case ID": "={{ $json['Case ID'] }}",
            "Last Contacted": "={{ new Date().toISOString().split('T')[0] }}",
            "Follow-up Count": "={{ parseInt($json['Follow-up Count'] || '0') + 1 }}"
          },
          "schema": [
            {
              "id": "Case ID",
              "displayName": "Case ID",
              "required": true,
              "defaultMatch": true,
              "canBeUsedToMatch": true
            },
            {
              "id": "Last Contacted",
              "displayName": "Last Contacted",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Follow-up Count",
              "displayName": "Follow-up Count",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            }
          ]
        },
        "options": {}
      },
      "id": "215ad594-3948-4467-8f02-af7dada870d1",
      "name": "M4 – Update Follow-up Count in CRM",
      "type": "n8n-nodes-base.googleSheets",
      "typeVersion": 4,
      "position": [
        160,
        4048
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "Increments follow-up count and updates last contacted date in CRM."
    },
    {
      "parameters": {
        "httpMethod": "POST",
        "path": "law-case-status",
        "responseMode": "responseNode",
        "options": {}
      },
      "id": "fa1fe7b1-681f-42e3-858e-6e434bdb1868",
      "name": "M4 – Case Status Update Webhook",
      "type": "n8n-nodes-base.webhook",
      "typeVersion": 2,
      "position": [
        -1312,
        4608
      ],
      "notesInFlow": true,
      "webhookId": "17b594f4-f672-4afd-b101-bd0bdd9c6d68",
      "notes": "Receives case status updates. POST body: { case_id, new_status, notes, lawyer_name }. Status values: New, Active, Pending Documents, Scheduled, Hearing Scheduled, Awaiting Judgment, Judgment Received, Closed, On Hold."
    },
    {
      "parameters": {
        "respondWith": "json",
        "responseBody": "{\"status\":\"updated\"}",
        "options": {}
      },
      "id": "a64ee2b1-e398-44c6-bf61-bbc0b287055a",
      "name": "M4 – Ack Status Update",
      "type": "n8n-nodes-base.respondToWebhook",
      "typeVersion": 1,
      "position": [
        -1088,
        4512
      ]
    },
    {
      "parameters": {
        "jsCode": "\nconst b = $json.body || $json;\nconst statusMessages = {\n  'New': 'Your case has been registered with our firm.',\n  'Active': 'Your case is now being actively handled by our legal team.',\n  'Pending Documents': 'We require additional documents from you. Please check the details below.',\n  'Scheduled': 'A consultation has been scheduled for your case.',\n  'Hearing Scheduled': 'A hearing has been scheduled for your case. Please ensure you are available.',\n  'Awaiting Judgment': 'Your case has been heard. We are awaiting the judgment.',\n  'Judgment Received': 'A judgment has been received in your case. Please contact us to discuss next steps.',\n  'Closed': 'Your case has been closed. Thank you for trusting us with your legal matter.',\n  'On Hold': 'Your case is temporarily on hold. Our team will contact you with further updates.'\n};\n\nconst newStatus = b.new_status || 'Active';\nconst clientMessage = statusMessages[newStatus] || `Your case status has been updated to: ${newStatus}`;\n\nreturn [{json: {\n  case_id: b.case_id,\n  new_status: newStatus,\n  notes: b.notes || '',\n  lawyer_name: b.lawyer_name || '',\n  client_message: clientMessage,\n  update_timestamp: new Date().toISOString()\n}}];\n"
      },
      "id": "862eb3c1-209c-4b31-a0f1-45f851685077",
      "name": "M4 – Process Status Update",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        -1088,
        4704
      ],
      "notesInFlow": true,
      "notes": "Processes case status update and selects appropriate client notification message."
    },
    {
      "parameters": {
        "operation": "getAll",
        "documentId": {
          "__rl": true,
          "mode": "id",
          "value": "REPLACE_GOOGLE_SHEET_ID_CRM"
        }
      },
      "id": "dbcf9c2a-0fd4-4f18-9491-1553beb43bfc",
      "name": "M4 – Fetch Case Details",
      "type": "n8n-nodes-base.googleSheets",
      "typeVersion": 4,
      "position": [
        -864,
        4704
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "Fetches current case record to get client contact details."
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/{{$vars.WHATSAPP_PHONE_NUMBER_ID}}/messages",
        "sendHeaders": true,
        "headerParameters": {
          "parameters": [
            {
              "name": "Authorization",
              "value": "=Bearer {{$vars.WHATSAPP_ACCESS_TOKEN}}"
            }
          ]
        },
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={{ JSON.stringify({\n  messaging_product: \"whatsapp\",\n  to: $json['Phone'],\n  type: \"text\",\n  text: {\n    body: `Dear ${$json['Client Name']},\\n\\n📋 Case Update (${$('M4 – Process Status Update').item.json.case_id})\\nStatus: ${$('M4 – Process Status Update').item.json.new_status}\\n\\n${$('M4 – Process Status Update').item.json.client_message}\\n\\n${$('M4 – Process Status Update').item.json.notes ? 'Note: ' + $('M4 – Process Status Update').item.json.notes : ''}\\n\\nContact: ${$vars.FIRM_PHONE}\\n${$vars.FIRM_NAME}`\n  }\n}) }}",
        "options": {}
      },
      "id": "8170d102-3bae-4df0-898c-90e78469f2f5",
      "name": "M4 – Notify Client of Status Change",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4,
      "position": [
        -640,
        4704
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "Sends WhatsApp notification to client about case status change."
    },
    {
      "parameters": {
        "httpMethod": "POST",
        "path": "law-contract-review",
        "responseMode": "responseNode",
        "options": {}
      },
      "id": "ee53d28f-c473-4e08-a46d-782bacf90256",
      "name": "M5 – Contract Review Webhook",
      "type": "n8n-nodes-base.webhook",
      "typeVersion": 2,
      "position": [
        -1312,
        5024
      ],
      "notesInFlow": true,
      "webhookId": "56e1d26b-d359-4072-9a6e-2f3c87210c4e",
      "notes": "Receives contract documents for AI review. POST multipart/form-data: file + { intake_id, contract_type, client_name, review_urgency }."
    },
    {
      "parameters": {
        "respondWith": "json",
        "responseBody": "{\"status\":\"received\",\"message\":\"Contract received for review. AI analysis will be ready for lawyer review within 5-10 minutes.\"}",
        "options": {}
      },
      "id": "ed3b0e0a-db97-4a3d-853a-54358eff226f",
      "name": "M5 – Ack Contract Upload",
      "type": "n8n-nodes-base.respondToWebhook",
      "typeVersion": 1,
      "position": [
        -1088,
        4928
      ],
      "notesInFlow": true,
      "notes": "Immediately ACKs contract review request."
    },
    {
      "parameters": {
        "jsCode": "\nconst body = $json.body || $json;\nconst binaryData = $binary || {};\nconst fileKey = Object.keys(binaryData)[0] || 'data';\nconst file = binaryData[fileKey] || {};\n\nreturn [{\n  json: {\n    review_id: 'REV-' + Date.now(),\n    intake_id: body.intake_id || '',\n    client_name: body.client_name || 'Unknown',\n    contract_type: body.contract_type || 'Contract',\n    review_urgency: body.review_urgency || 'Normal',\n    original_filename: file.fileName || 'contract',\n    mime_type: file.mimeType || 'application/pdf',\n    upload_timestamp: new Date().toISOString(),\n    review_status: 'pending_ai_review'\n  },\n  binary: binaryData\n}];\n"
      },
      "id": "7b9a3764-9570-4ad1-93ca-29151693abd8",
      "name": "M5 – Prepare Contract Metadata",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        -1088,
        5120
      ],
      "notesInFlow": true,
      "notes": "Prepares contract metadata for AI review pipeline."
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://documentai.googleapis.com/v1/projects/{{$vars.GOOGLE_PROJECT_ID}}/locations/us/processors/{{$vars.DOCUMENT_AI_PROCESSOR_ID}}:process",
        "sendHeaders": true,
        "headerParameters": {
          "parameters": [
            {
              "name": "Authorization",
              "value": "=Bearer {{$vars.GOOGLE_DOCUMENT_AI_TOKEN}}"
            }
          ]
        },
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={{ JSON.stringify({\n  rawDocument: {\n    content: $binary.data?.data || '',\n    mimeType: $json.mime_type || 'application/pdf'\n  }\n}) }}",
        "options": {}
      },
      "id": "e4670a95-f96a-43b7-b18a-63823bc6adf5",
      "name": "M5 – Extract Contract Text (OCR)",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4,
      "position": [
        -864,
        5120
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "Extracts contract text via Google Document AI. Same OCR endpoint as M3. Replace with your OCR provider."
    },
    {
      "parameters": {
        "jsCode": "\nconst meta = $('M5 – Prepare Contract Metadata').item.json;\nlet text = '';\ntry {\n  const resp = $json;\n  text = resp?.document?.text || resp?.text || resp?.content || '';\n} catch(e) {\n  text = '';\n}\nconst truncated = text.substring(0, 10000);\nreturn [{json: {\n  ...meta,\n  contract_text: truncated,\n  was_truncated: text.length > 10000,\n  text_length: text.length\n}}];\n"
      },
      "id": "4ee8e66a-bbe0-4721-9bb2-c9d7aa57da6d",
      "name": "M5 – Prepare Contract Text",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        -640,
        5120
      ],
      "notesInFlow": true,
      "notes": "Extracts text from OCR response and prepares for contract review LLM."
    },
    {
      "parameters": {
        "model": "llama-3.3-70b-versatile",
        "options": {
          "maxTokens": 3000,
          "temperature": 0.1
        }
      },
      "id": "e4fa9440-b0e4-4ac7-96bb-b70ee2079c3d",
      "name": "M5 – Groq LLM (Contract Review)",
      "type": "@n8n/n8n-nodes-langchain.lmChatOpenAi",
      "typeVersion": 1,
      "position": [
        -352,
        5344
      ],
      "notesInFlow": true,
      "notes": "Groq LLaMA for contract clause analysis."
    },
    {
      "parameters": {
        "prompt": "You are a legal document review assistant helping prepare a contract summary for lawyer review. Analyze the following contract and return a structured JSON review ONLY — no prose, no markdown fences.\n\nContract Type: {{ $json.contract_type }}\nClient: {{ $json.client_name }}\n{{ $json.was_truncated ? '⚠️ Document was truncated at 10,000 characters.' : '' }}\n\nContract Text:\n{{ $json.contract_text }}\n\nReturn this exact JSON:\n{\n  \"contract_overview\": {\n    \"contract_type\": \"<identified contract type>\",\n    \"parties\": [{\"role\": \"<role>\", \"name\": \"<party name>\"}],\n    \"effective_date\": \"<date or 'Not specified'>\",\n    \"expiry_date\": \"<date or 'Not specified'>\",\n    \"governing_law\": \"<jurisdiction>\"\n  },\n  \"clause_analysis\": [\n    {\n      \"clause_name\": \"<clause name>\",\n      \"finding\": \"<what the clause says in plain English>\",\n      \"risk_level\": \"<High|Medium|Low|Neutral>\",\n      \"explanation\": \"<why this is notable or risky>\",\n      \"recommendation\": \"<what the lawyer should consider>\"\n    }\n  ],\n  \"key_obligations\": {\n    \"party_a\": [\"<obligation 1>\", \"<obligation 2>\"],\n    \"party_b\": [\"<obligation 1>\", \"<obligation 2>\"]\n  },\n  \"important_dates_and_deadlines\": [\n    {\"description\": \"<event>\", \"date\": \"<date>\", \"urgency\": \"<High|Medium|Low>\"}\n  ],\n  \"payment_terms\": {\n    \"amount\": \"<amount or 'Not specified'>\",\n    \"schedule\": \"<payment schedule>\",\n    \"penalties\": \"<penalty clauses if any>\"\n  },\n  \"termination_conditions\": [\"<condition 1>\", \"<condition 2>\"],\n  \"risk_summary\": {\n    \"overall_risk\": \"<High|Medium|Low>\",\n    \"high_risk_clauses\": [\"<clause name 1>\"],\n    \"missing_standard_clauses\": [\"<missing clause if any>\"],\n    \"red_flags\": [\"<red flag if any>\"]\n  },\n  \"executive_summary\": \"<3-5 sentence plain-language summary for lawyer review>\",\n  \"requires_immediate_attention\": <true|false>,\n  \"attention_reason\": \"<reason if true>\",\n  \"ai_disclaimer\": \"This contract review is AI-generated for lawyer review assistance only. It does not constitute legal advice. All findings MUST be independently reviewed and verified by a qualified lawyer before any legal action or advice is given to the client.\"\n}\n\nCRITICAL: This output is for LAWYER REVIEW ONLY. Never present AI findings as final legal advice. Mark all findings clearly as preliminary AI analysis requiring lawyer verification."
      },
      "id": "7e80d4b6-d750-4333-b3aa-d8e8ded9a4fe",
      "name": "M5 – AI Contract Clause Review",
      "type": "@n8n/n8n-nodes-langchain.chainLlm",
      "typeVersion": 1,
      "position": [
        -416,
        5120
      ],
      "notesInFlow": true,
      "notes": "Performs AI contract clause analysis using Groq LLaMA. All output is marked for lawyer review only."
    },
    {
      "parameters": {
        "jsCode": "\nconst meta = $('M5 – Prepare Contract Text').item.json;\nlet review = {};\ntry {\n  const raw = $json.text || '{}';\n  review = JSON.parse(raw.replace(/```json/g,'').replace(/```/g,'').trim());\n} catch(e) {\n  review = {\n    executive_summary: 'AI review parsing failed. Manual review required.',\n    risk_summary: { overall_risk: 'Unknown', high_risk_clauses: [], red_flags: [] },\n    clause_analysis: [],\n    ai_disclaimer: 'Parsing failed. Full manual review required.',\n    requires_immediate_attention: true,\n    attention_reason: 'AI parsing failure — manual review required'\n  };\n}\nreturn [{json: {\n  ...meta,\n  ...review,\n  review_generated_at: new Date().toISOString(),\n  review_status: 'pending_lawyer_approval'\n}}];\n"
      },
      "id": "551e06ba-a2ec-49b6-8fa9-c6dc1958ebe3",
      "name": "M5 – Parse Contract Review",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        -64,
        5120
      ],
      "notesInFlow": true,
      "notes": "Parses contract review JSON with fallback. Status set to pending_lawyer_approval."
    },
    {
      "parameters": {
        "operation": "appendOrUpdate",
        "documentId": {
          "__rl": true,
          "mode": "id",
          "value": "REPLACE_GOOGLE_SHEET_ID_CRM"
        },
        "sheetName": {
          "__rl": true,
          "mode": "name",
          "value": "Contract Reviews"
        },
        "columns": {
          "mappingMode": "defineBelow",
          "value": {
            "Review ID": "={{ $json.review_id }}",
            "Intake ID": "={{ $json.intake_id }}",
            "Client Name": "={{ $json.client_name }}",
            "Contract Type": "={{ $json.contract_type }}",
            "Overall Risk": "={{ $json.risk_summary?.overall_risk || 'Unknown' }}",
            "Requires Attention": "={{ $json.requires_immediate_attention ? 'YES' : 'No' }}",
            "Executive Summary": "={{ $json.executive_summary }}",
            "High Risk Clauses": "={{ ($json.risk_summary?.high_risk_clauses || []).join(', ') }}",
            "Red Flags": "={{ ($json.risk_summary?.red_flags || []).join(', ') }}",
            "Review Status": "Pending Lawyer Approval",
            "Review Generated At": "={{ $json.review_generated_at }}",
            "Lawyer Reviewed": "No",
            "Lawyer Notes": "",
            "AI Disclaimer": "={{ $json.ai_disclaimer }}"
          },
          "schema": [
            {
              "id": "Review ID",
              "displayName": "Review ID",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": true
            },
            {
              "id": "Intake ID",
              "displayName": "Intake ID",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Client Name",
              "displayName": "Client Name",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Contract Type",
              "displayName": "Contract Type",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Overall Risk",
              "displayName": "Overall Risk",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Requires Attention",
              "displayName": "Requires Attention",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Executive Summary",
              "displayName": "Executive Summary",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "High Risk Clauses",
              "displayName": "High Risk Clauses",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Red Flags",
              "displayName": "Red Flags",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Review Status",
              "displayName": "Review Status",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Review Generated At",
              "displayName": "Review Generated At",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Lawyer Reviewed",
              "displayName": "Lawyer Reviewed",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Lawyer Notes",
              "displayName": "Lawyer Notes",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "AI Disclaimer",
              "displayName": "AI Disclaimer",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            }
          ]
        },
        "options": {}
      },
      "id": "d1bfe8f3-6cd7-48dc-bc63-f8d2369adfdf",
      "name": "M5 – Store Contract Review in Sheets",
      "type": "n8n-nodes-base.googleSheets",
      "typeVersion": 4,
      "position": [
        160,
        5120
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "Stores AI contract review in Contract Reviews tab. Status: Pending Lawyer Approval."
    },
    {
      "parameters": {
        "subject": "=🔍 AI Contract Review Ready – {{ $json.client_name }} | Risk: {{ $json.risk_summary?.overall_risk }} | REQUIRES LAWYER APPROVAL",
        "message": "=<h2>⚠️ AI Contract Review – Lawyer Approval Required</h2>\n<p style=\"background:#fff3cd;padding:10px;border-radius:4px\"><strong>IMPORTANT:</strong> This review was generated by AI for administrative assistance only. All findings MUST be independently verified by a qualified lawyer. Do not share directly with the client without lawyer review and approval.</p>\n\n<table border=\"1\" cellpadding=\"6\">\n<tr><th>Review ID</th><td>{{ $json.review_id }}</td></tr>\n<tr><th>Client</th><td>{{ $json.client_name }}</td></tr>\n<tr><th>Contract Type</th><td>{{ $json.contract_type }}</td></tr>\n<tr><th>Overall Risk</th><td style=\"color:{{ $json.risk_summary?.overall_risk === 'High' ? 'red' : $json.risk_summary?.overall_risk === 'Medium' ? 'orange' : 'green' }}\"><strong>{{ $json.risk_summary?.overall_risk }}</strong></td></tr>\n<tr><th>Immediate Attention</th><td>{{ $json.requires_immediate_attention ? '⚠️ YES – ' + $json.attention_reason : 'No' }}</td></tr>\n</table>\n\n<h3>Executive Summary</h3><p>{{ $json.executive_summary }}</p>\n\n<h3>High Risk Clauses</h3><ul>{{ ($json.risk_summary?.high_risk_clauses || []).map(c => `<li>${c}</li>`).join('') || '<li>None identified</li>' }}</ul>\n\n<h3>Red Flags</h3><ul>{{ ($json.risk_summary?.red_flags || []).map(f => `<li>${f}</li>`).join('') || '<li>None identified</li>' }}</ul>\n\n<h3>Clause Analysis ({{ ($json.clause_analysis || []).length }} clauses)</h3>\n{{ ($json.clause_analysis || []).map(c => `<div style=\"margin:8px 0;padding:8px;border-left:4px solid ${c.risk_level==='High'?'red':c.risk_level==='Medium'?'orange':'green'}\"><strong>${c.clause_name}</strong> [${c.risk_level} Risk]<br>${c.finding}<br><em>Note: ${c.explanation}</em></div>`).join('') }}\n\n<h3>Payment Terms</h3><p>{{ $json.payment_terms?.amount || 'Not specified' }} | {{ $json.payment_terms?.schedule || 'N/A' }}</p>\n\n<h3>Termination Conditions</h3><ul>{{ ($json.termination_conditions || []).map(t => `<li>${t}</li>`).join('') }}</ul>\n\n<p><strong>To approve and send to client, reply to this email or update the review status in the CRM.</strong></p>\n<p><em>{{ $json.ai_disclaimer }}</em></p>",
        "options": {}
      },
      "id": "b67c06bd-90bd-4760-8bd8-48aad5bc12a2",
      "name": "M5 – Send Review to Lawyer (Human Approval)",
      "type": "n8n-nodes-base.gmail",
      "typeVersion": 2,
      "position": [
        384,
        5024
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "webhookId": "fb2e8171-401a-4e13-96b4-dfd2fb19319d",
      "onError": "continueErrorOutput",
      "notes": "HUMAN APPROVAL CHECKPOINT: Sends full AI contract review to lawyer. Client is NOT notified until lawyer approves. Lawyer must update status in CRM to 'Lawyer Approved' to trigger client notification."
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://api.telegram.org/bot{{$vars.TELEGRAM_BOT_TOKEN}}/sendMessage",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={{ JSON.stringify({\n  chat_id: $vars.LAWYER_TELEGRAM_CHAT_ID,\n  text: `📄 Contract Review Ready\\n\\nClient: ${$json.client_name}\\nType: ${$json.contract_type}\\nRisk: ${$json.risk_summary?.overall_risk || 'Unknown'}\\n${$json.requires_immediate_attention ? '⚠️ URGENT: ' + $json.attention_reason : ''}\\n\\nReview ID: ${$json.review_id}\\n\\nCheck email for full AI analysis. APPROVAL REQUIRED before client delivery.`,\n  parse_mode: 'HTML'\n}) }}",
        "options": {}
      },
      "id": "d0bc547b-20c0-47cb-979c-ec4f1a07bf57",
      "name": "M5 – Telegram Alert – Contract Review Ready",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4,
      "position": [
        384,
        5216
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "Sends Telegram alert to lawyer that contract review is ready for approval."
    },
    {
      "parameters": {
        "httpMethod": "POST",
        "path": "law-review-approve",
        "responseMode": "responseNode",
        "options": {}
      },
      "id": "a80f4e9e-645f-42a5-90eb-2a9b82f7ef22",
      "name": "M5 – Review Approval Webhook",
      "type": "n8n-nodes-base.webhook",
      "typeVersion": 2,
      "position": [
        -1312,
        5536
      ],
      "notesInFlow": true,
      "webhookId": "2e2f5ae0-1bb5-442d-8f48-bc9bb2374f65",
      "notes": "Lawyer triggers this after reviewing contract AI output. POST: { review_id, approved, lawyer_notes, send_to_client }. This is the human-in-the-loop approval gate."
    },
    {
      "parameters": {
        "respondWith": "json",
        "responseBody": "{\"status\":\"approval_received\"}",
        "options": {}
      },
      "id": "927b50c3-d4f7-4704-b288-0a80937f0c9e",
      "name": "M5 – Ack Approval",
      "type": "n8n-nodes-base.respondToWebhook",
      "typeVersion": 1,
      "position": [
        -1088,
        5440
      ]
    },
    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": false,
            "leftValue": "",
            "typeValidation": "strict"
          },
          "conditions": [
            {
              "id": "b2840516-5cb5-416d-a0bc-2d8c8a718e5c",
              "leftValue": "={{ String(($json.body || $json).approved) }}",
              "rightValue": "true",
              "operator": {
                "type": "string",
                "operation": "equals"
              }
            }
          ],
          "combinator": "and"
        },
        "options": {}
      },
      "id": "b4da3d6b-054f-419e-bb49-4fe210a4866f",
      "name": "M5 – Approved by Lawyer?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2,
      "position": [
        -1088,
        5632
      ],
      "notesInFlow": true,
      "notes": "Routes based on lawyer approval. True = approved, send client summary. False = rejected, notify admin."
    },
    {
      "parameters": {
        "operation": "getAll",
        "documentId": {
          "__rl": true,
          "mode": "id",
          "value": "REPLACE_GOOGLE_SHEET_ID_CRM"
        }
      },
      "id": "343f2b52-fc76-4100-b30d-1802e5e392f2",
      "name": "M5 – Fetch Approved Review",
      "type": "n8n-nodes-base.googleSheets",
      "typeVersion": 4,
      "position": [
        -864,
        5632
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "Fetches the approved review record to send client summary."
    },
    {
      "parameters": {
        "subject": "=Contract Review Summary – {{ $json['Client Name'] }}",
        "message": "=<h2>Contract Review Summary</h2>\n<p>Dear Team,</p>\n<p>The lawyer-approved contract review summary for <strong>{{ $json['Client Name'] }}</strong> is ready.</p>\n<p><strong>Contract:</strong> {{ $json['Contract Type'] }}<br>\n<strong>Overall Risk Assessment:</strong> {{ $json['Overall Risk'] }}</p>\n<h3>Key Findings</h3>\n<p>{{ $json['Executive Summary'] }}</p>\n<h3>Points Requiring Your Attention</h3>\n<p>{{ $json['High Risk Clauses'] ? 'Key clauses: ' + $json['High Risk Clauses'] : 'No high-risk clauses identified.' }}</p>\n<p><strong>Lawyer Notes:</strong> {{ $json['Lawyer Notes'] || 'None' }}</p>\n<p><em>⚠️ This summary has been prepared and reviewed by our legal team. For specific legal advice, please schedule a consultation.</em></p>\n<p>Review ID: {{ $json['Review ID'] }}</p>",
        "options": {}
      },
      "id": "4e64f57b-4980-498a-b84c-93a6482d54f3",
      "name": "M5 – Send Lawyer-Approved Summary to Client",
      "type": "n8n-nodes-base.gmail",
      "typeVersion": 2,
      "position": [
        -640,
        5632
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "webhookId": "e29abf91-56c9-4233-bde9-0ec2ef63912f",
      "onError": "continueErrorOutput",
      "notes": "Sends lawyer-approved contract summary to client. Template — update To field to client email from CRM."
    },
    {
      "parameters": {
        "httpMethod": "POST",
        "path": "whatsapp-inbound",
        "responseMode": "responseNode",
        "options": {}
      },
      "id": "96f40d8f-bb26-4105-8c1f-17a8c10e5c38",
      "name": "WA – Inbound WhatsApp Webhook",
      "type": "n8n-nodes-base.webhook",
      "typeVersion": 2,
      "position": [
        -1312,
        5952
      ],
      "notesInFlow": true,
      "webhookId": "25933330-2a95-40ab-8c9b-3d74cea0bbfe",
      "notes": "Meta WhatsApp Business webhook for inbound messages. Configure in Meta Developer Portal → WhatsApp → Webhook. Verify token = REPLACE_WA_VERIFY_TOKEN."
    },
    {
      "parameters": {
        "path": "whatsapp-verify",
        "responseMode": "responseNode",
        "options": {}
      },
      "id": "773221df-dd0a-44a8-8b8a-aee3db4b393e",
      "name": "WA – Handle Webhook Verification (GET)",
      "type": "n8n-nodes-base.webhook",
      "typeVersion": 2,
      "position": [
        -1312,
        6384
      ],
      "notesInFlow": true,
      "webhookId": "592e7309-c1ea-4e05-8448-469bef405225",
      "notes": "Handles Meta's GET verification challenge. Returns hub.challenge when hub.verify_token matches."
    },
    {
      "parameters": {
        "respondWith": "text",
        "responseBody": "={{ $json.query?.['hub.challenge'] || '' }}",
        "options": {}
      },
      "id": "fc122370-5708-480b-bd2e-ad61e2bc1d44",
      "name": "WA – Respond to Verification",
      "type": "n8n-nodes-base.respondToWebhook",
      "typeVersion": 1,
      "position": [
        -1088,
        6384
      ],
      "notesInFlow": true,
      "notes": "Returns hub.challenge for Meta webhook verification. Customize to check hub.verify_token == REPLACE_WA_VERIFY_TOKEN."
    },
    {
      "parameters": {
        "respondWith": "json",
        "responseBody": "{\"status\":\"ok\"}",
        "options": {}
      },
      "id": "979436e2-1289-486f-a393-d135fc5cb051",
      "name": "WA – Ack WhatsApp 200",
      "type": "n8n-nodes-base.respondToWebhook",
      "typeVersion": 1,
      "position": [
        -1088,
        5856
      ],
      "notesInFlow": true,
      "notes": "Immediately returns 200 to Meta (required within 20s or Meta retries)."
    },
    {
      "parameters": {
        "jsCode": "\nconst body = $json.body || $json;\n// Meta WhatsApp webhook structure\nconst entry = (body.entry || [])[0] || {};\nconst changes = (entry.changes || [])[0] || {};\nconst value = changes.value || {};\nconst messages = value.messages || [];\nconst msg = messages[0] || {};\nconst contact = (value.contacts || [])[0] || {};\n\nif (!msg.id) {\n  // Not a message event (could be status update) — return empty to skip\n  return [];\n}\n\nreturn [{json: {\n  message_id: msg.id,\n  from_phone: msg.from || '',\n  from_name: contact.profile?.name || 'Unknown',\n  message_type: msg.type || 'text',\n  message_text: msg.text?.body || msg.interactive?.button_reply?.title || '',\n  timestamp: msg.timestamp || '',\n  phone_number_id: value.metadata?.phone_number_id || '',\n  received_at: new Date().toISOString()\n}}];\n"
      },
      "id": "6fbe9446-e351-4258-b655-3ebdea757464",
      "name": "WA – Parse Inbound Message",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        -1088,
        6048
      ],
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "Parses Meta WhatsApp inbound webhook payload. Returns empty array for non-message events (e.g. delivery receipts)."
    },
    {
      "parameters": {
        "model": "llama-3.3-70b-versatile",
        "options": {
          "maxTokens": 500,
          "temperature": 0.6
        }
      },
      "id": "32c66494-0d60-4c08-a103-a22286f23e1c",
      "name": "WA – Groq LLM (Receptionist)",
      "type": "@n8n/n8n-nodes-langchain.lmChatOpenAi",
      "typeVersion": 1,
      "position": [
        -800,
        6272
      ],
      "notesInFlow": true,
      "notes": "Groq LLaMA for WhatsApp AI receptionist responses."
    },
    {
      "parameters": {
        "prompt": "You are a professional AI receptionist for {{ $vars.FIRM_NAME }}, a law firm. Respond to the client's WhatsApp message in a helpful, professional, and empathetic manner.\n\nClient Name: {{ $json.from_name }}\nClient Phone: {{ $json.from_phone }}\nClient Message: {{ $json.message_text }}\nCurrent Time: {{ $now.toISO() }}\n\nFirm Contact: {{ $vars.FIRM_PHONE }}\nFirm Address: {{ $vars.FIRM_ADDRESS }}\n\nRULES:\n1. Be warm, professional, and empathetic\n2. NEVER provide legal advice or legal opinions\n3. For legal questions, always direct to schedule a consultation\n4. You CAN help with: appointment scheduling, firm information, document submission guidance, general process information\n5. For urgent matters, provide the emergency contact\n6. Keep response under 200 words for WhatsApp\n7. Respond in the same language as the client message (support Hindi, English, and regional languages)\n8. If the client wants to submit a case, ask them to share: Name, Phone, Email, and brief description of their legal issue\n\nRespond naturally as a receptionist. Do not add any JSON or formatting — plain text only."
      },
      "id": "2bba735b-c014-4853-9e77-4294c2ae2361",
      "name": "WA – AI Receptionist Response",
      "type": "@n8n/n8n-nodes-langchain.chainLlm",
      "typeVersion": 1,
      "position": [
        -864,
        6048
      ],
      "notesInFlow": true,
      "notes": "AI receptionist for WhatsApp. Does not provide legal advice. Directs clients to appropriate actions."
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/{{$json.phone_number_id}}/messages",
        "sendHeaders": true,
        "headerParameters": {
          "parameters": [
            {
              "name": "Authorization",
              "value": "=Bearer {{$vars.WHATSAPP_ACCESS_TOKEN}}"
            }
          ]
        },
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={{ JSON.stringify({\n  messaging_product: \"whatsapp\",\n  to: $('WA – Parse Inbound Message').item.json.from_phone,\n  type: \"text\",\n  text: { body: $json.text || 'Thank you for contacting us. A team member will reach out shortly.' }\n}) }}",
        "options": {}
      },
      "id": "07623c43-a58b-4986-872e-9689c336a104",
      "name": "WA – Send AI Reply to Client",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4,
      "position": [
        -512,
        6048
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "Sends AI-generated receptionist reply to client via WhatsApp."
    },
    {
      "parameters": {
        "jsCode": "\nconst exec = $json.execution;\nconst wf = $json.workflow;\nconst err = exec.error || {};\nreturn [{\n  json: {\n    alert_type: \"WORKFLOW_ERROR\",\n    workflow_name: wf.name,\n    workflow_id: wf.id,\n    execution_id: exec.id,\n    execution_url: exec.url,\n    failed_node: exec.lastNodeExecuted,\n    error_message: err.message || 'Unknown error',\n    error_name: err.name || 'Error',\n    timestamp: new Date().toISOString(),\n    alert_text: `⚠️ Workflow Error\\nWorkflow: ${wf.name}\\nFailed Node: ${exec.lastNodeExecuted}\\nError: ${err.message}\\nExecution: ${exec.url}`\n  }\n}];\n"
      },
      "id": "105536f1-3c4c-48ea-b10a-60e7d36faac4",
      "name": "Format Error Alert1",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        -1312,
        -2752
      ],
      "notesInFlow": true,
      "notes": "Formats error data into a structured alert payload."
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://api.telegram.org/bot{{$vars.TELEGRAM_BOT_TOKEN}}/sendMessage",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={{ JSON.stringify({ chat_id: $vars.TELEGRAM_ALERT_CHAT_ID, text: $json.alert_text, parse_mode: 'HTML' }) }}",
        "options": {}
      },
      "id": "bbad910a-89c8-48a6-bd2f-718215e23038",
      "name": "Send Error to Telegram1",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4,
      "position": [
        -1088,
        -2752
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "notes": "Sends error alert to Telegram. Configure TELEGRAM_BOT_TOKEN and TELEGRAM_ALERT_CHAT_ID in n8n env vars."
    },
    {
      "parameters": {
        "httpMethod": "POST",
        "path": "law-intake",
        "responseMode": "responseNode",
        "options": {}
      },
      "id": "83839ff1-16c6-4008-bab2-3064342da8b6",
      "name": "M1 – Intake Webhook1",
      "type": "n8n-nodes-base.webhook",
      "typeVersion": 2,
      "position": [
        -1312,
        -2192
      ],
      "notesInFlow": true,
      "webhookId": "e4fa934c-123c-4bbe-8da5-fd27a6769317",
      "notes": "Receives intake submissions from website forms, WhatsApp API, or any HTTP source. POST body must include: name, phone, email, case_category, location, issue_summary, urgency, preferred_time, source."
    },
    {
      "parameters": {
        "jsCode": "\nconst b = $json.body || $json;\nconst required = ['name','phone','email','issue_summary'];\nconst missing = required.filter(f => !b[f] || String(b[f]).trim() === '');\nif (missing.length) {\n  throw new Error('VALIDATION_ERROR: Missing required fields: ' + missing.join(', '));\n}\nconst phone = String(b.phone).replace(/\\D/g,'');\nif (phone.length < 10) throw new Error('VALIDATION_ERROR: Invalid phone number');\nreturn [{json: {\n  name: String(b.name).trim(),\n  phone: phone,\n  email: String(b.email || '').trim().toLowerCase(),\n  case_category: String(b.case_category || 'General Enquiry').trim(),\n  location: String(b.location || '').trim(),\n  issue_summary: String(b.issue_summary).trim(),\n  urgency: String(b.urgency || 'Normal').trim(),\n  preferred_time: String(b.preferred_time || '').trim(),\n  source: String(b.source || 'website').trim(),\n  intake_id: 'INT-' + Date.now(),\n  received_at: new Date().toISOString()\n}}];\n"
      },
      "id": "f4158637-51a3-41ee-87f8-23e9536d9b45",
      "name": "M1 – Validate Intake Input1",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        -1088,
        -2192
      ],
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "Validates required intake fields. Throws VALIDATION_ERROR for invalid inputs."
    },
    {
      "parameters": {
        "respondWith": "json",
        "responseBody": "={{ JSON.stringify({ error: 'VALIDATION_ERROR', message: $json.error.message.replace('VALIDATION_ERROR: ','') }) }}",
        "options": {}
      },
      "id": "e3325c4f-effd-4793-a940-1e5f0dfef9ef",
      "name": "M1 – Intake Validation Error Response1",
      "type": "n8n-nodes-base.respondToWebhook",
      "typeVersion": 1,
      "position": [
        -800,
        -2096
      ],
      "notesInFlow": true,
      "notes": "Returns 400 for invalid intake submissions."
    },
    {
      "parameters": {
        "model": "llama-3.3-70b-versatile",
        "options": {
          "temperature": 0.1
        }
      },
      "id": "1d4efc06-0d58-4a15-ad81-5bdcad2db315",
      "name": "M1 – Groq LLM (Classify)1",
      "type": "@n8n/n8n-nodes-langchain.lmChatOpenAi",
      "typeVersion": 1,
      "position": [
        -800,
        -2272
      ],
      "notesInFlow": true,
      "notes": "Groq LLM via OpenAI-compatible credential. Set base URL to api.groq.com/openai/v1 in credential."
    },
    {
      "parameters": {
        "prompt": "You are a legal intake classification assistant for a professional law firm. Analyze the following client enquiry and return a JSON object ONLY — no prose, no markdown fences.\n\nClient Information:\nName: {{ $json.name }}\nCase Category: {{ $json.case_category }}\nLocation: {{ $json.location }}\nIssue Summary: {{ $json.issue_summary }}\nUrgency: {{ $json.urgency }}\nSource: {{ $json.source }}\n\nReturn exactly this JSON:\n{\n  \"legal_matter_type\": \"<one of: Criminal, Civil, Family, Property, Corporate, Employment, Consumer, Intellectual Property, Immigration, Taxation, Other>\",\n  \"sub_category\": \"<specific sub-type e.g. Divorce, NDA, FIR, etc.>\",\n  \"qualification_score\": \"<Hot|Warm|Cold>\",\n  \"qualification_reason\": \"<1-2 sentence reason>\",\n  \"urgency_level\": \"<High|Medium|Low>\",\n  \"estimated_complexity\": \"<Simple|Moderate|Complex>\",\n  \"recommended_lawyer_type\": \"<e.g. Criminal Defense Advocate, Corporate Lawyer, etc.>\",\n  \"key_facts\": [\"<fact 1>\", \"<fact 2>\"],\n  \"missing_info\": [\"<missing item 1 if any>\"],\n  \"intake_risk_flags\": [\"<flag if any, e.g. urgent court date, criminal matter, etc.>\"],\n  \"ai_disclaimer\": \"This classification is for administrative intake purposes only and does not constitute legal advice.\"\n}\n\nIMPORTANT: Never provide legal advice. Never make legal judgments. This is an administrative classification only."
      },
      "id": "d351a411-2893-4f7d-9627-cdc105a2a7eb",
      "name": "M1 – Classify & Qualify Lead1",
      "type": "@n8n/n8n-nodes-langchain.chainLlm",
      "typeVersion": 1,
      "position": [
        -864,
        -2496
      ],
      "notesInFlow": true,
      "notes": "Uses Groq LLaMA to classify the legal matter and qualify the lead. Output is in $json.text."
    },
    {
      "parameters": {
        "jsCode": "\nconst prev = $('M1 – Validate Intake Input1').item.json;\nlet classification = {};\ntry {\n  const raw = $json.text || '{}';\n  const cleaned = raw.replace(/```json/g,'').replace(/```/g,'').trim();\n  classification = JSON.parse(cleaned);\n} catch(e) {\n  classification = {\n    legal_matter_type: 'Other',\n    sub_category: 'General',\n    qualification_score: 'Warm',\n    qualification_reason: 'Auto-classified due to parse error',\n    urgency_level: prev.urgency === 'High' ? 'High' : 'Medium',\n    estimated_complexity: 'Moderate',\n    recommended_lawyer_type: 'General Advocate',\n    key_facts: [],\n    missing_info: [],\n    intake_risk_flags: [],\n    ai_disclaimer: 'Classification failed — manual review required.'\n  };\n}\nreturn [{json: {\n  ...prev,\n  ...classification,\n  classification_timestamp: new Date().toISOString()\n}}];\n"
      },
      "id": "0ea4c729-5b77-45f2-aea3-8651a40addf2",
      "name": "M1 – Parse Classification JSON1",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        -512,
        -2384
      ],
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "Parses LLM JSON output with fallback for malformed responses."
    },
    {
      "parameters": {
        "operation": "appendOrUpdate",
        "documentId": {
          "__rl": true,
          "mode": "id",
          "value": "REPLACE_GOOGLE_SHEET_ID_CRM"
        },
        "sheetName": {
          "__rl": true,
          "mode": "name",
          "value": "Leads"
        },
        "columns": {
          "mappingMode": "defineBelow",
          "value": {
            "Intake ID": "={{ $json.intake_id }}",
            "Received At": "={{ $json.received_at }}",
            "Source": "={{ $json.source }}",
            "Name": "={{ $json.name }}",
            "Phone": "={{ $json.phone }}",
            "Email": "={{ $json.email }}",
            "Case Category": "={{ $json.case_category }}",
            "Location": "={{ $json.location }}",
            "Issue Summary": "={{ $json.issue_summary }}",
            "Urgency": "={{ $json.urgency }}",
            "Preferred Time": "={{ $json.preferred_time }}",
            "Legal Matter Type": "={{ $json.legal_matter_type }}",
            "Sub Category": "={{ $json.sub_category }}",
            "Qualification Score": "={{ $json.qualification_score }}",
            "Qualification Reason": "={{ $json.qualification_reason }}",
            "Urgency Level": "={{ $json.urgency_level }}",
            "Complexity": "={{ $json.estimated_complexity }}",
            "Recommended Lawyer": "={{ $json.recommended_lawyer_type }}",
            "Status": "New",
            "Assigned To": "",
            "Case ID": ""
          },
          "schema": [
            {
              "id": "Intake ID",
              "displayName": "Intake ID",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": true
            },
            {
              "id": "Received At",
              "displayName": "Received At",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Source",
              "displayName": "Source",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Name",
              "displayName": "Name",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Phone",
              "displayName": "Phone",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Email",
              "displayName": "Email",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Case Category",
              "displayName": "Case Category",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Location",
              "displayName": "Location",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Issue Summary",
              "displayName": "Issue Summary",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Urgency",
              "displayName": "Urgency",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Preferred Time",
              "displayName": "Preferred Time",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Legal Matter Type",
              "displayName": "Legal Matter Type",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Sub Category",
              "displayName": "Sub Category",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Qualification Score",
              "displayName": "Qualification Score",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Qualification Reason",
              "displayName": "Qualification Reason",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Urgency Level",
              "displayName": "Urgency Level",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Complexity",
              "displayName": "Complexity",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Recommended Lawyer",
              "displayName": "Recommended Lawyer",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Status",
              "displayName": "Status",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Assigned To",
              "displayName": "Assigned To",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Case ID",
              "displayName": "Case ID",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            }
          ]
        },
        "options": {}
      },
      "id": "a0530b22-cd30-4146-bd2d-71b443032626",
      "name": "M1 – Store Lead in CRM Sheet1",
      "type": "n8n-nodes-base.googleSheets",
      "typeVersion": 4,
      "position": [
        -288,
        -2384
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "Appends new lead to Google Sheets CRM tab 'Leads'. Configure REPLACE_GOOGLE_SHEET_ID_CRM."
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://api.telegram.org/bot{{$vars.TELEGRAM_BOT_TOKEN}}/sendMessage",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={{ JSON.stringify({\n  chat_id: $vars.LAWYER_TELEGRAM_CHAT_ID,\n  text: `🔔 NEW LEAD — ${$json.qualification_score.toUpperCase()}\\n\\n👤 ${$json.name}\\n📞 ${$json.phone}\\n📧 ${$json.email}\\n\\n⚖️ Matter: ${$json.legal_matter_type} › ${$json.sub_category}\\n🚨 Urgency: ${$json.urgency_level}\\n📍 Location: ${$json.location}\\n\\n📝 Issue:\\n${$json.issue_summary}\\n\\n🤖 Recommended Lawyer: ${$json.recommended_lawyer_type}\\n📋 Intake ID: ${$json.intake_id}\\n\\n⚠️ AI classification only — not legal advice.`,\n  parse_mode: 'HTML'\n}) }}",
        "options": {}
      },
      "id": "8fa2f21e-2a9b-4d9b-96cb-74a09d6fcfee",
      "name": "M1 – Notify Lawyer via Telegram1",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4,
      "position": [
        -64,
        -2528
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "Sends Telegram notification to lawyer/team. Set LAWYER_TELEGRAM_CHAT_ID in env vars."
    },
    {
      "parameters": {
        "subject": "=New {{ $json.qualification_score }} Lead: {{ $json.legal_matter_type }} – {{ $json.name }}",
        "message": "=<h2>New Client Intake – {{ $json.qualification_score }} Lead</h2>\n<table border=\"1\" cellpadding=\"6\">\n<tr><th>Intake ID</th><td>{{ $json.intake_id }}</td></tr>\n<tr><th>Name</th><td>{{ $json.name }}</td></tr>\n<tr><th>Phone</th><td>{{ $json.phone }}</td></tr>\n<tr><th>Email</th><td>{{ $json.email }}</td></tr>\n<tr><th>Legal Matter</th><td>{{ $json.legal_matter_type }} › {{ $json.sub_category }}</td></tr>\n<tr><th>Urgency</th><td>{{ $json.urgency_level }}</td></tr>\n<tr><th>Location</th><td>{{ $json.location }}</td></tr>\n<tr><th>Preferred Time</th><td>{{ $json.preferred_time }}</td></tr>\n<tr><th>Recommended Lawyer</th><td>{{ $json.recommended_lawyer_type }}</td></tr>\n</table>\n<h3>Issue Summary</h3><p>{{ $json.issue_summary }}</p>\n<h3>AI Classification Notes</h3><p>{{ $json.qualification_reason }}</p>\n<p><em>⚠️ AI-generated classification for administrative purposes only. Not legal advice.</em></p>",
        "options": {}
      },
      "id": "e85156cf-c799-4dc3-8046-01487ae2d308",
      "name": "M1 – Notify Lawyer via Email1",
      "type": "n8n-nodes-base.gmail",
      "typeVersion": 2,
      "position": [
        -64,
        -2336
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "webhookId": "92042d66-8616-492f-993f-79130e8778c8",
      "onError": "continueErrorOutput",
      "notes": "Sends email notification to lawyer. Configure LAWYER_EMAIL in n8n env vars."
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/{{$vars.WHATSAPP_PHONE_NUMBER_ID}}/messages",
        "sendHeaders": true,
        "headerParameters": {
          "parameters": [
            {
              "name": "Authorization",
              "value": "=Bearer {{$vars.WHATSAPP_ACCESS_TOKEN}}"
            }
          ]
        },
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={{ JSON.stringify({\n  messaging_product: \"whatsapp\",\n  to: $json.phone,\n  type: \"text\",\n  text: {\n    body: `Dear ${$json.name},\\n\\nThank you for reaching out to our law firm. We have received your enquiry (Ref: ${$json.intake_id}) regarding ${$json.legal_matter_type}.\\n\\nOur team will review your case and contact you within 24 hours.\\n\\nIf this is urgent, please call: ${$vars.FIRM_PHONE}\\n\\nRegards,\\n${$vars.FIRM_NAME}`\n  }\n}) }}",
        "options": {}
      },
      "id": "81a63390-c524-43e8-82da-80a23d091f78",
      "name": "M1 – Send WhatsApp Ack to Client1",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4,
      "position": [
        -64,
        -2144
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "Sends WhatsApp acknowledgement to client. Configure WHATSAPP_PHONE_NUMBER_ID, WHATSAPP_ACCESS_TOKEN, FIRM_PHONE, FIRM_NAME in env vars."
    },
    {
      "parameters": {
        "respondWith": "json",
        "responseBody": "={{ JSON.stringify({ success: true, intake_id: $('M1 – Parse Classification JSON1').item.json.intake_id, message: 'Intake received. Our team will contact you within 24 hours.', qualification: $('M1 – Parse Classification JSON1').item.json.qualification_score }) }}",
        "options": {}
      },
      "id": "9459c6f5-b461-4a3d-9833-f0abd086a4b5",
      "name": "M1 – Intake Success Response1",
      "type": "n8n-nodes-base.respondToWebhook",
      "typeVersion": 1,
      "position": [
        160,
        -2528
      ],
      "notesInFlow": true,
      "notes": "Returns 200 with intake ID to the submitter."
    },
    {
      "parameters": {
        "httpMethod": "POST",
        "path": "sarvam-voice-callback",
        "responseMode": "responseNode",
        "options": {}
      },
      "id": "8dc2b241-1325-4d05-93a0-fb4f9022ff5e",
      "name": "M2 – Voice Webhook (Sarvam Callback)1",
      "type": "n8n-nodes-base.webhook",
      "typeVersion": 2,
      "position": [
        -1312,
        -1776
      ],
      "notesInFlow": true,
      "webhookId": "cf13338f-0e84-4a35-a98a-fac0dd5b22c6",
      "notes": "Receives callbacks from Sarvam AI voice calls. Sarvam posts conversation data here after each call session."
    },
    {
      "parameters": {
        "respondWith": "json",
        "responseBody": "{\"status\":\"received\"}",
        "options": {}
      },
      "id": "be37d885-54eb-4d0f-b43a-6fbd2a4991c3",
      "name": "M2 – Ack Sarvam ",
      "type": "n8n-nodes-base.respondToWebhook",
      "typeVersion": 1,
      "position": [
        -1088,
        -1872
      ],
      "notesInFlow": true,
      "notes": "Immediately ACKs Sarvam webhook. Processing continues asynchronously."
    },
    {
      "parameters": {
        "jsCode": "\nconst b = $json.body || $json;\n// Sarvam callback structure (adapt to actual Sarvam response schema)\nconst transcript = b.transcript || b.conversation_transcript || '';\nconst callData = {\n  call_id: b.call_id || b.session_id || 'CALL-' + Date.now(),\n  caller_phone: b.from || b.caller_number || '',\n  call_duration_sec: b.duration || 0,\n  language_detected: b.language || 'en-IN',\n  transcript: transcript,\n  call_status: b.status || 'completed',\n  call_timestamp: b.timestamp || new Date().toISOString(),\n  // Fields to be extracted by AI below\n  client_name: b.extracted?.name || '',\n  client_email: b.extracted?.email || '',\n  case_summary: b.extracted?.case_summary || transcript.substring(0, 500),\n  preferred_time: b.extracted?.preferred_time || '',\n  urgency: b.extracted?.urgency || 'Normal',\n  appointment_requested: b.extracted?.appointment_requested || false\n};\nreturn [{json: callData}];\n"
      },
      "id": "08f221b9-6f27-41ca-b77d-c9ec7e7a1c65",
      "name": "M2 – Parse Voice Call Data1",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        -1088,
        -1680
      ],
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "Parses Sarvam AI voice callback. Adapt field mappings to match your Sarvam plan's actual webhook schema."
    },
    {
      "parameters": {
        "model": "llama-3.3-70b-versatile",
        "options": {
          "temperature": 0.1
        }
      },
      "id": "d504493c-0a75-44ac-8e50-ada36e28dc2f",
      "name": "M2 – Groq LLM (Voice Extract)1",
      "type": "@n8n/n8n-nodes-langchain.lmChatOpenAi",
      "typeVersion": 1,
      "position": [
        -800,
        -1456
      ],
      "notesInFlow": true,
      "notes": "Groq LLM for extracting structured data from voice transcript."
    },
    {
      "parameters": {
        "prompt": "You are a legal receptionist AI assistant. Extract structured client information from this voice call transcript. Return JSON ONLY.\n\nCall Transcript:\n{{ $json.transcript }}\n\nCaller Phone: {{ $json.caller_phone }}\nLanguage: {{ $json.language_detected }}\n\nReturn this exact JSON:\n{\n  \"client_name\": \"<extracted or 'Unknown'>\",\n  \"client_email\": \"<extracted or ''>\",\n  \"case_category\": \"<one of: Criminal, Civil, Family, Property, Corporate, Employment, Consumer, Other>\",\n  \"issue_summary\": \"<2-3 sentence summary of the legal issue discussed>\",\n  \"urgency\": \"<High|Normal|Low>\",\n  \"preferred_consultation_time\": \"<extracted time preference or ''>\",\n  \"appointment_requested\": <true|false>,\n  \"language_spoken\": \"<language identified>\",\n  \"key_points\": [\"<point 1>\", \"<point 2>\"],\n  \"follow_up_required\": <true|false>,\n  \"follow_up_notes\": \"<any specific follow-up needed>\",\n  \"ai_disclaimer\": \"Extracted from voice call for administrative purposes only. Not legal advice.\"\n}\n\nIMPORTANT: Do not provide legal advice. Do not make legal judgments. Administrative extraction only."
      },
      "id": "5d1c222b-c4c2-4d2d-b75d-14b614af4c52",
      "name": "M2 – Extract Client Info from Transcript1",
      "type": "@n8n/n8n-nodes-langchain.chainLlm",
      "typeVersion": 1,
      "position": [
        -864,
        -1680
      ],
      "notesInFlow": true,
      "notes": "Extracts structured client data from voice transcript using Groq LLaMA."
    },
    {
      "parameters": {
        "jsCode": "\nconst voiceData = $('M2 – Parse Voice Call Data1').item.json;\nlet extracted = {};\ntry {\n  const raw = $json.text || '{}';\n  extracted = JSON.parse(raw.replace(/```json/g,'').replace(/```/g,'').trim());\n} catch(e) {\n  extracted = { client_name: 'Unknown', issue_summary: voiceData.transcript?.substring(0,300) || '' };\n}\nreturn [{json: {\n  ...voiceData,\n  ...extracted,\n  // Merge: AI extraction overrides raw parse where available\n  client_name: extracted.client_name || voiceData.client_name || 'Unknown',\n  case_category: extracted.case_category || 'General Enquiry',\n  issue_summary: extracted.issue_summary || voiceData.case_summary,\n  urgency: extracted.urgency || voiceData.urgency,\n  preferred_time: extracted.preferred_consultation_time || voiceData.preferred_time,\n  appointment_requested: extracted.appointment_requested || voiceData.appointment_requested,\n  source: 'voice_call',\n  intake_id: 'CALL-' + voiceData.call_id\n}}];\n"
      },
      "id": "0e9be908-4030-42ea-a9bc-211a12071d15",
      "name": "M2 – Merge Voice + AI Extraction1",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        -512,
        -1680
      ],
      "notesInFlow": true,
      "notes": "Merges raw voice call data with AI-extracted structured fields."
    },
    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": false,
            "leftValue": "",
            "typeValidation": "strict"
          },
          "conditions": [
            {
              "id": "e23801f3-5a0c-4609-9849-c2fc65d3eb1c",
              "leftValue": "={{ String($json.appointment_requested) }}",
              "rightValue": "true",
              "operator": {
                "type": "string",
                "operation": "equals"
              }
            }
          ],
          "combinator": "and"
        },
        "options": {}
      },
      "id": "998a79e8-c6eb-4fe7-8d8a-950f95756624",
      "name": "M2 – Appointment Requested?1",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2,
      "position": [
        -288,
        -1680
      ],
      "notesInFlow": true,
      "notes": "Routes to appointment booking if client requested one during voice call."
    },
    {
      "parameters": {
        "operation": "appendOrUpdate",
        "documentId": {
          "__rl": true,
          "mode": "id",
          "value": "REPLACE_GOOGLE_SHEET_ID_CRM"
        },
        "sheetName": {
          "__rl": true,
          "mode": "name",
          "value": "Leads"
        },
        "columns": {
          "mappingMode": "defineBelow",
          "value": {
            "Intake ID": "={{ $json.intake_id }}",
            "Received At": "={{ $json.call_timestamp }}",
            "Source": "voice_call",
            "Name": "={{ $json.client_name }}",
            "Phone": "={{ $json.caller_phone }}",
            "Email": "={{ $json.client_email }}",
            "Case Category": "={{ $json.case_category }}",
            "Issue Summary": "={{ $json.issue_summary }}",
            "Urgency": "={{ $json.urgency }}",
            "Preferred Time": "={{ $json.preferred_time }}",
            "Status": "New – Voice",
            "Legal Matter Type": "",
            "Sub Category": "",
            "Qualification Score": "Warm"
          },
          "schema": [
            {
              "id": "Intake ID",
              "displayName": "Intake ID",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": true
            },
            {
              "id": "Received At",
              "displayName": "Received At",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Source",
              "displayName": "Source",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Name",
              "displayName": "Name",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Phone",
              "displayName": "Phone",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Email",
              "displayName": "Email",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Case Category",
              "displayName": "Case Category",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Issue Summary",
              "displayName": "Issue Summary",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Urgency",
              "displayName": "Urgency",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Preferred Time",
              "displayName": "Preferred Time",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Status",
              "displayName": "Status",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Legal Matter Type",
              "displayName": "Legal Matter Type",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Sub Category",
              "displayName": "Sub Category",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Qualification Score",
              "displayName": "Qualification Score",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            }
          ]
        },
        "options": {}
      },
      "id": "7341a16a-456d-41c7-aa91-40bd90903a03",
      "name": "M2 – Store Voice Lead in CRM1",
      "type": "n8n-nodes-base.googleSheets",
      "typeVersion": 4,
      "position": [
        -64,
        -1776
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "Stores voice call lead in CRM. Both appointment and non-appointment paths converge here."
    },
    {
      "parameters": {
        "operation": "getAll",
        "calendar": {
          "__rl": true,
          "mode": "id",
          "value": "REPLACE_GOOGLE_CALENDAR_ID"
        },
        "options": {}
      },
      "id": "5b326197-35f1-4ca7-8d1c-c2aba335507e",
      "name": "M2 – Check Calendar Availability1",
      "type": "n8n-nodes-base.googleCalendar",
      "typeVersion": 1,
      "position": [
        -64,
        -1584
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "Fetches existing calendar events to check lawyer availability. Configure REPLACE_GOOGLE_CALENDAR_ID."
    },
    {
      "parameters": {
        "calendar": {
          "__rl": true,
          "mode": "id",
          "value": "REPLACE_GOOGLE_CALENDAR_ID"
        },
        "start": "={{ $now.plus({days:1}).set({hour:10,minute:0,second:0}).toISO() }}",
        "end": "={{ $now.plus({days:1}).set({hour:11,minute:0,second:0}).toISO() }}",
        "additionalFields": {
          "attendees": "={{ $('M2 – Merge Voice + AI Extraction1').item.json.client_email ? [$('M2 – Merge Voice + AI Extraction1').item.json.client_email] : [] }}",
          "description": "=Client: {{ $('M2 – Merge Voice + AI Extraction1').item.json.client_name }}\nPhone: {{ $('M2 – Merge Voice + AI Extraction1').item.json.caller_phone }}\nCase: {{ $('M2 – Merge Voice + AI Extraction1').item.json.issue_summary }}\n\nIntake ID: {{ $('M2 – Merge Voice + AI Extraction1').item.json.intake_id }}",
          "summary": "=Consultation: {{ $('M2 – Merge Voice + AI Extraction1').item.json.client_name }} – {{ $('M2 – Merge Voice + AI Extraction1').item.json.case_category }}"
        }
      },
      "id": "9f18e71d-51aa-4a7e-a6f3-03909661b0d2",
      "name": "M2 – Create Appointment in Calendar1",
      "type": "n8n-nodes-base.googleCalendar",
      "typeVersion": 1,
      "position": [
        160,
        -1584
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "Creates calendar appointment. In production, implement availability-check logic before booking."
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/{{$vars.WHATSAPP_PHONE_NUMBER_ID}}/messages",
        "sendHeaders": true,
        "headerParameters": {
          "parameters": [
            {
              "name": "Authorization",
              "value": "=Bearer {{$vars.WHATSAPP_ACCESS_TOKEN}}"
            }
          ]
        },
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={{ JSON.stringify({\n  messaging_product: \"whatsapp\",\n  to: $('M2 – Merge Voice + AI Extraction1').item.json.caller_phone,\n  type: \"text\",\n  text: {\n    body: `Dear ${$('M2 – Merge Voice + AI Extraction1').item.json.client_name},\\n\\nYour consultation has been scheduled!\\n\\n📅 Date: ${$json.start?.split('T')[0] || 'TBD'}\\n⏰ Time: 10:00 AM\\n📍 ${$vars.FIRM_ADDRESS}\\n\\nPlease bring any relevant documents.\\n\\nRef: ${$('M2 – Merge Voice + AI Extraction1').item.json.intake_id}\\n\\n${$vars.FIRM_NAME}`\n  }\n}) }}",
        "options": {}
      },
      "id": "d0e657c1-70f5-4d0e-bc32-0cea0999c4eb",
      "name": "M2 – Send Appointment Confirmation WA1",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4,
      "position": [
        384,
        -1584
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "Sends WhatsApp appointment confirmation to client."
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://api.sarvam.ai/v1/phone-calls",
        "sendHeaders": true,
        "headerParameters": {
          "parameters": [
            {
              "name": "api-subscription-key",
              "value": "={{ $vars.SARVAM_API_KEY }}"
            },
            {
              "name": "Content-Type",
              "value": "application/json"
            }
          ]
        },
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={{ JSON.stringify({\n  agent_id: $vars.SARVAM_AGENT_ID,\n  phone_number: $json.phone,\n  metadata: {\n    client_name: $json.name,\n    case_ref: $json.intake_id,\n    callback_url: $vars.N8N_WEBHOOK_BASE + '/webhook/sarvam-voice-callback'\n  }\n}) }}",
        "options": {}
      },
      "id": "cf607135-4c29-4039-8597-e86be0e9eb5b",
      "name": "M2 – Initiate Sarvam Outbound Call1",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4,
      "position": [
        -1312,
        -1344
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "Initiates outbound Sarvam AI call. Configure SARVAM_API_KEY, SARVAM_AGENT_ID in env vars. Sarvam agent must be pre-configured in Sarvam dashboard with multilingual support (Hindi, Tamil, Telugu, etc.). N8N_WEBHOOK_BASE = your ngrok/production URL."
    },
    {
      "parameters": {
        "httpMethod": "POST",
        "path": "law-document-upload",
        "responseMode": "responseNode",
        "options": {}
      },
      "id": "b39aee7b-24e7-47d3-8fd8-79cee207829c",
      "name": "M3 – Document Upload Webhook1",
      "type": "n8n-nodes-base.webhook",
      "typeVersion": 2,
      "position": [
        -1312,
        -1024
      ],
      "notesInFlow": true,
      "webhookId": "61480fcf-fe42-48f4-999e-7e82fc530ec6",
      "notes": "Receives document uploads. Accepts multipart/form-data with file + metadata (intake_id, doc_type, client_name, case_category). Supports PDF, DOCX, TXT."
    },
    {
      "parameters": {
        "respondWith": "json",
        "responseBody": "{\"status\":\"received\",\"message\":\"Document received. Processing will complete within 2-3 minutes.\"}",
        "options": {}
      },
      "id": "72c3cf60-dbb0-42f8-b36c-e20e4da5e935",
      "name": "M3 – Ack Document Upload1",
      "type": "n8n-nodes-base.respondToWebhook",
      "typeVersion": 1,
      "position": [
        -1088,
        -1120
      ],
      "notesInFlow": true,
      "notes": "Immediately ACKs document upload webhook."
    },
    {
      "parameters": {
        "jsCode": "\n// Extract metadata from form fields\nconst body = $json.body || $json;\nconst binaryData = $binary;\n\n// Get file info\nconst fileKey = Object.keys(binaryData || {})[0] || 'data';\nconst file = binaryData[fileKey] || {};\n\nreturn [{\n  json: {\n    doc_id: 'DOC-' + Date.now(),\n    intake_id: body.intake_id || '',\n    client_name: body.client_name || 'Unknown',\n    case_category: body.case_category || 'General',\n    doc_type: body.doc_type || 'Unknown',\n    original_filename: file.fileName || 'document',\n    mime_type: file.mimeType || 'application/octet-stream',\n    file_size: file.fileSize || 0,\n    upload_timestamp: new Date().toISOString(),\n    extraction_note: 'Text extraction via external OCR API or n8n Extract from File node required. Configure EXTRACT_FROM_FILE_NODE or OCR_API_URL.'\n  },\n  binary: binaryData\n}];\n"
      },
      "id": "8a920e4f-bdb9-44c9-9a87-b3d33b005875",
      "name": "M3 – Extract Document Text1",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        -1088,
        -928
      ],
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "Prepares document metadata. Binary file is passed through for OCR/extraction. For PDF text extraction, use n8n Extract from File node or connect to OCR API (e.g., Google Document AI, Textract)."
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://documentai.googleapis.com/v1/projects/{{$vars.GOOGLE_PROJECT_ID}}/locations/us/processors/{{$vars.DOCUMENT_AI_PROCESSOR_ID}}:process",
        "sendHeaders": true,
        "headerParameters": {
          "parameters": [
            {
              "name": "Authorization",
              "value": "=Bearer {{$vars.GOOGLE_DOCUMENT_AI_TOKEN}}"
            }
          ]
        },
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={{ JSON.stringify({\n  rawDocument: {\n    content: $binary.data?.data || '',\n    mimeType: $json.mime_type || 'application/pdf'\n  }\n}) }}",
        "options": {
          "response": {
            "response": {}
          }
        }
      },
      "id": "0623d339-5a4d-439b-bc89-57ca26b1c9e3",
      "name": "M3 – OCR / Text Extraction API1",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4,
      "position": [
        -864,
        -928
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "PLACEHOLDER: Google Document AI OCR. Replace with your OCR provider (AWS Textract, Mistral OCR API, or n8n Extract from File node for simple PDFs). Configure GOOGLE_PROJECT_ID, DOCUMENT_AI_PROCESSOR_ID, GOOGLE_DOCUMENT_AI_TOKEN. For simple text PDFs, use Extract from File node instead."
    },
    {
      "parameters": {
        "jsCode": "\nconst meta = $('M3 – Extract Document Text1').item.json;\n// Try to extract text from Document AI response\nlet extractedText = '';\ntry {\n  const resp = $json;\n  extractedText = resp?.document?.text || resp?.text || resp?.content || '';\n  if (!extractedText && resp?.pages) {\n    extractedText = resp.pages.map(p => p.paragraphs?.map(para => \n      para.layout?.textAnchor?.textSegments?.map(seg => \n        resp.document?.text?.substring(seg.startIndex || 0, seg.endIndex || 0)\n      ).join(' ')\n    ).join('\\n')).join('\\n');\n  }\n} catch(e) {\n  extractedText = 'EXTRACTION_FAILED: ' + e.message;\n}\n\n// Truncate for LLM context window (keep ~8000 chars)\nconst truncated = extractedText.substring(0, 8000);\nconst wasTruncated = extractedText.length > 8000;\n\nreturn [{json: {\n  ...meta,\n  extracted_text: truncated,\n  was_truncated: wasTruncated,\n  text_length: extractedText.length,\n  extraction_success: !extractedText.startsWith('EXTRACTION_FAILED')\n}}];\n"
      },
      "id": "17b81934-0653-401a-ac70-307aa00e64ff",
      "name": "M3 – Prepare Extracted Text1",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        -640,
        -928
      ],
      "notesInFlow": true,
      "notes": "Extracts text from OCR API response. Truncates to 8000 chars for LLM context window."
    },
    {
      "parameters": {
        "model": "llama-3.3-70b-versatile",
        "options": {
          "maxTokens": 2048,
          "temperature": 0.1
        }
      },
      "id": "de60290a-8253-4ea9-84df-a091b68a2b13",
      "name": "M3 – Groq LLM (Summarize)1",
      "type": "@n8n/n8n-nodes-langchain.lmChatOpenAi",
      "typeVersion": 1,
      "position": [
        -352,
        -816
      ],
      "notesInFlow": true,
      "notes": "Groq LLaMA for legal document summarization."
    },
    {
      "parameters": {
        "prompt": "You are a legal document analysis assistant. Analyze the following legal document and return a structured JSON summary ONLY — no prose, no markdown fences.\n\nDocument Type: {{ $json.doc_type }}\nCase Category: {{ $json.case_category }}\nClient: {{ $json.client_name }}\n\nDocument Text:\n{{ $json.extracted_text }}\n\n{{ $json.was_truncated ? '⚠️ Note: Document was truncated due to length. Analysis is based on first 8000 characters.' : '' }}\n\nReturn this exact JSON:\n{\n  \"document_type\": \"<identified document type>\",\n  \"parties_involved\": [{\"role\": \"<Party Role>\", \"name\": \"<Party Name>\"}],\n  \"case_type\": \"<legal matter type>\",\n  \"key_facts\": [\"<fact 1>\", \"<fact 2>\", \"<fact 3>\"],\n  \"important_dates\": [{\"event\": \"<event description>\", \"date\": \"<date>\"}],\n  \"deadlines\": [{\"deadline\": \"<description>\", \"date\": \"<date>\", \"urgency\": \"<High|Medium|Low>\"}],\n  \"required_actions\": [\"<action 1>\", \"<action 2>\"],\n  \"missing_information\": [\"<missing item if any>\"],\n  \"missing_documents\": [\"<document needed if any>\"],\n  \"document_summary\": \"<2-3 paragraph plain-language summary of the document>\",\n  \"complexity_assessment\": \"<Simple|Moderate|Complex>\",\n  \"immediate_attention_required\": <true|false>,\n  \"attention_reason\": \"<reason if immediate attention required, else ''>\",\n  \"ai_disclaimer\": \"This summary is AI-generated for administrative review purposes only and does not constitute legal advice. All findings must be reviewed and verified by a qualified lawyer.\"\n}\n\nIMPORTANT: Do not provide legal advice. This is for administrative document processing only."
      },
      "id": "97ccd83e-1231-4ed3-bd5f-102cbb5560ab",
      "name": "M3 – AI Document Summarization1",
      "type": "@n8n/n8n-nodes-langchain.chainLlm",
      "typeVersion": 1,
      "position": [
        -416,
        -1040
      ],
      "notesInFlow": true,
      "notes": "Generates structured legal document summary using Groq LLaMA."
    },
    {
      "parameters": {
        "jsCode": "\nconst meta = $('M3 – Prepare Extracted Text1').item.json;\nlet summary = {};\ntry {\n  const raw = $json.text || '{}';\n  summary = JSON.parse(raw.replace(/```json/g,'').replace(/```/g,'').trim());\n} catch(e) {\n  summary = {\n    document_type: meta.doc_type,\n    document_summary: 'AI summary generation failed. Manual review required.',\n    ai_disclaimer: 'Parsing failed. Manual review required.',\n    missing_information: [],\n    missing_documents: [],\n    required_actions: ['Manual review required'],\n    immediate_attention_required: false\n  };\n}\nreturn [{json: {\n  ...meta,\n  ...summary,\n  summary_generated_at: new Date().toISOString()\n}}];\n"
      },
      "id": "b10bdc43-afae-4175-b4e8-1bb338507719",
      "name": "M3 – Parse Document Summary1",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        -64,
        -928
      ],
      "notesInFlow": true,
      "notes": "Parses LLM document summary with fallback."
    },
    {
      "parameters": {
        "name": "={{ $json.original_filename }}",
        "driveId": {
          "__rl": true,
          "mode": "myDrive",
          "value": "myDrive"
        },
        "folderId": {
          "__rl": true,
          "mode": "list",
          "value": "root",
          "cachedResultName": "/ (Root folder)"
        },
        "options": {}
      },
      "id": "1acf320b-7b0a-40d4-b5b1-7de6bba7943b",
      "name": "M3 – Upload Original Doc to Drive1",
      "type": "n8n-nodes-base.googleDrive",
      "typeVersion": 3,
      "position": [
        160,
        -1024
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "Uploads original document to Google Drive. Configure REPLACE_GDRIVE_DOCUMENTS_FOLDER_ID."
    },
    {
      "parameters": {
        "operation": "appendOrUpdate",
        "documentId": {
          "__rl": true,
          "mode": "id",
          "value": "REPLACE_GOOGLE_SHEET_ID_CRM"
        },
        "sheetName": {
          "__rl": true,
          "mode": "name",
          "value": "Documents"
        },
        "columns": {
          "mappingMode": "defineBelow",
          "value": {
            "Doc ID": "={{ $json.doc_id }}",
            "Intake ID": "={{ $json.intake_id }}",
            "Client Name": "={{ $json.client_name }}",
            "Original Filename": "={{ $json.original_filename }}",
            "Doc Type": "={{ $json.document_type }}",
            "Case Type": "={{ $json.case_type || $json.case_category }}",
            "Upload Timestamp": "={{ $json.upload_timestamp }}",
            "Summary Generated At": "={{ $json.summary_generated_at }}",
            "Key Summary": "={{ $json.document_summary }}",
            "Missing Docs": "={{ ($json.missing_documents || []).join(', ') }}",
            "Required Actions": "={{ ($json.required_actions || []).join(', ') }}",
            "Immediate Attention": "={{ $json.immediate_attention_required ? 'YES' : 'No' }}",
            "Complexity": "={{ $json.complexity_assessment }}",
            "AI Disclaimer": "={{ $json.ai_disclaimer }}"
          },
          "schema": [
            {
              "id": "Doc ID",
              "displayName": "Doc ID",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": true
            },
            {
              "id": "Intake ID",
              "displayName": "Intake ID",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Client Name",
              "displayName": "Client Name",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Original Filename",
              "displayName": "Original Filename",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Doc Type",
              "displayName": "Doc Type",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Case Type",
              "displayName": "Case Type",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Upload Timestamp",
              "displayName": "Upload Timestamp",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Summary Generated At",
              "displayName": "Summary Generated At",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Key Summary",
              "displayName": "Key Summary",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Missing Docs",
              "displayName": "Missing Docs",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Required Actions",
              "displayName": "Required Actions",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Immediate Attention",
              "displayName": "Immediate Attention",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Complexity",
              "displayName": "Complexity",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "AI Disclaimer",
              "displayName": "AI Disclaimer",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            }
          ]
        },
        "options": {}
      },
      "id": "7ee4ccee-ddda-4e14-8b83-c83c7440d118",
      "name": "M3 – Store Summary in Sheets1",
      "type": "n8n-nodes-base.googleSheets",
      "typeVersion": 4,
      "position": [
        160,
        -832
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "Stores document AI summary in the Documents tab of the CRM sheet."
    },
    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": false,
            "leftValue": "",
            "typeValidation": "strict"
          },
          "conditions": [
            {
              "id": "a01e816b-a89d-42de-b114-26066935b16b",
              "leftValue": "={{ ($json.missing_documents || []).length > 0 || ($json.missing_information || []).length > 0 }}",
              "rightValue": "true",
              "operator": {
                "type": "string",
                "operation": "equals"
              }
            }
          ],
          "combinator": "and"
        },
        "options": {}
      },
      "id": "aa0f845d-d7c2-403b-9519-5305588388c6",
      "name": "M3 – Missing Docs?1",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2,
      "position": [
        384,
        -832
      ],
      "notesInFlow": true,
      "notes": "Checks if AI identified missing documents or information."
    },
    {
      "parameters": {
        "subject": "=⚠️ Missing Documents Alert – {{ $json.client_name }} ({{ $json.doc_id }})",
        "message": "=<h2>Missing Documents Detected</h2>\n<p><strong>Client:</strong> {{ $json.client_name }}<br>\n<strong>Doc ID:</strong> {{ $json.doc_id }}<br>\n<strong>Intake ID:</strong> {{ $json.intake_id }}</p>\n<h3>Missing Documents</h3><ul>{{ $json.missing_documents?.map(d => `<li>${d}</li>`).join('') || '<li>None</li>' }}</ul>\n<h3>Missing Information</h3><ul>{{ $json.missing_information?.map(i => `<li>${i}</li>`).join('') || '<li>None</li>' }}</ul>\n<h3>Required Actions</h3><ul>{{ $json.required_actions?.map(a => `<li>${a}</li>`).join('') || '<li>None</li>' }}</ul>\n<h3>Document Summary</h3><p>{{ $json.document_summary }}</p>\n<p><em>⚠️ AI-generated for review purposes. Not legal advice.</em></p>",
        "options": {}
      },
      "id": "f3704f24-5f48-4116-b8cd-9f21307400d3",
      "name": "M3 – Notify Lawyer – Missing Docs1",
      "type": "n8n-nodes-base.gmail",
      "typeVersion": 2,
      "position": [
        608,
        -832
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "webhookId": "4eca87b3-f9e5-4b8e-8717-b528315a54d0",
      "onError": "continueErrorOutput",
      "notes": "Notifies lawyer when AI detects missing documents or information."
    },
    {
      "parameters": {
        "rule": {
          "interval": [
            {
              "field": "cronExpression",
              "expression": "0 9 * * 1-6"
            }
          ]
        }
      },
      "id": "a64ea16b-e250-4fda-9f43-0cc8ed695142",
      "name": "M4 – Daily Follow-up Scheduler1",
      "type": "n8n-nodes-base.scheduleTrigger",
      "typeVersion": 1,
      "position": [
        -1312,
        -496
      ],
      "notesInFlow": true,
      "notes": "Triggers daily at 9 AM Monday-Saturday to process follow-up sequences and case status updates."
    },
    {
      "parameters": {
        "operation": "getAll",
        "documentId": {
          "__rl": true,
          "mode": "id",
          "value": "REPLACE_GOOGLE_SHEET_ID_CRM"
        }
      },
      "id": "e2008fb5-56fe-4a07-b8c8-b1f167f04dc4",
      "name": "M4 – Fetch Active Cases from CRM1",
      "type": "n8n-nodes-base.googleSheets",
      "typeVersion": 4,
      "position": [
        -1088,
        -496
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "Fetches all cases from the Cases tab. Cases sheet columns: Case ID, Client Name, Phone, Email, Lawyer, Status, Next Hearing, Deadline, Pending Docs, Last Contacted, Follow-up Count."
    },
    {
      "parameters": {
        "jsCode": "\nconst now = new Date();\nconst today = now.toISOString().split('T')[0];\n\nconst casesNeedingFollowup = $input.all()\n  .map(item => item.json)\n  .filter(c => {\n    const status = (c['Status'] || '').toLowerCase();\n    // Skip closed/completed cases\n    if (['closed','completed','cancelled'].includes(status)) return false;\n    \n    const deadline = c['Deadline'] ? new Date(c['Deadline']) : null;\n    const nextHearing = c['Next Hearing'] ? new Date(c['Next Hearing']) : null;\n    const lastContacted = c['Last Contacted'] ? new Date(c['Last Contacted']) : null;\n    const followupCount = parseInt(c['Follow-up Count'] || '0');\n    \n    // Max 3 follow-ups per case\n    if (followupCount >= 3) return false;\n    \n    // Urgent: deadline within 7 days\n    if (deadline) {\n      const daysToDeadline = Math.ceil((deadline - now) / (1000*60*60*24));\n      if (daysToDeadline <= 7 && daysToDeadline > 0) return true;\n    }\n    \n    // Hearing within 3 days\n    if (nextHearing) {\n      const daysToHearing = Math.ceil((nextHearing - now) / (1000*60*60*24));\n      if (daysToHearing <= 3 && daysToHearing > 0) return true;\n    }\n    \n    // Not contacted in 7+ days\n    if (!lastContacted) return true;\n    const daysSinceContact = Math.ceil((now - lastContacted) / (1000*60*60*24));\n    if (daysSinceContact >= 7) return true;\n    \n    // Has pending docs and not contacted recently\n    if (c['Pending Docs'] && c['Pending Docs'] !== '' && daysSinceContact >= 3) return true;\n    \n    return false;\n  })\n  .map(c => {\n    const deadline = c['Deadline'] ? new Date(c['Deadline']) : null;\n    const nextHearing = c['Next Hearing'] ? new Date(c['Next Hearing']) : null;\n    \n    let reason = 'routine';\n    let priority = 'normal';\n    \n    if (deadline && Math.ceil((deadline - now)/(1000*60*60*24)) <= 3) {\n      reason = 'urgent_deadline'; priority = 'high';\n    } else if (nextHearing && Math.ceil((nextHearing - now)/(1000*60*60*24)) <= 3) {\n      reason = 'hearing_reminder'; priority = 'high';\n    } else if (c['Pending Docs']) {\n      reason = 'pending_documents'; priority = 'medium';\n    }\n    \n    return { ...c, followup_reason: reason, followup_priority: priority };\n  });\n\nreturn casesNeedingFollowup.map(c => ({json: c}));\n"
      },
      "id": "81aed5c8-4308-463f-885d-9516b63315c1",
      "name": "M4 – Filter Cases Needing Follow-up1",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        -864,
        -496
      ],
      "notesInFlow": true,
      "notes": "Filters cases requiring follow-up based on deadlines, hearings, last contact date, and pending documents."
    },
    {
      "parameters": {
        "model": "llama-3.3-70b-versatile",
        "options": {
          "temperature": 0.5
        }
      },
      "id": "b7135e0d-07df-445e-a497-e4a782e7b9a8",
      "name": "M4 – Groq LLM (Follow-up Msg)1",
      "type": "@n8n/n8n-nodes-langchain.lmChatOpenAi",
      "typeVersion": 1,
      "position": [
        -576,
        -272
      ],
      "notesInFlow": true,
      "notes": "Groq LLaMA for generating personalized follow-up messages."
    },
    {
      "parameters": {
        "prompt": "Generate a personalized, professional client follow-up WhatsApp message for a law firm. Return JSON ONLY.\n\nClient: {{ $json['Client Name'] }}\nCase Status: {{ $json['Status'] }}\nFollow-up Reason: {{ $json.followup_reason }}\nPriority: {{ $json.followup_priority }}\nNext Hearing: {{ $json['Next Hearing'] || 'Not scheduled' }}\nDeadline: {{ $json['Deadline'] || 'Not set' }}\nPending Documents: {{ $json['Pending Docs'] || 'None' }}\nLawyer: {{ $json['Lawyer'] || 'Our team' }}\nFirm Name: {{ $vars.FIRM_NAME }}\nFirm Phone: {{ $vars.FIRM_PHONE }}\n\nReturn this JSON:\n{\n  \"whatsapp_message\": \"<personalized WhatsApp message in simple English, max 300 chars, professional and empathetic>\",\n  \"email_subject\": \"<email subject line>\",\n  \"email_body_html\": \"<short HTML email body, professional, with case status update>\",\n  \"escalate_to_lawyer\": <true if high priority and 3rd follow-up>,\n  \"escalation_note\": \"<note for lawyer if escalating>\"\n}\n\nRules:\n- Be warm, professional, and empathetic\n- Do NOT discuss legal strategy or provide legal advice\n- Refer to upcoming dates/deadlines if present\n- For pending documents, clearly list what is needed\n- Keep WhatsApp message under 300 characters"
      },
      "id": "eff0a215-6b39-4544-b57c-b7c692ab646c",
      "name": "M4 – Generate Personalized Follow-up1",
      "type": "@n8n/n8n-nodes-langchain.chainLlm",
      "typeVersion": 1,
      "position": [
        -640,
        -496
      ],
      "notesInFlow": true,
      "notes": "Generates personalized follow-up message using AI. Each case gets a unique, contextual message."
    },
    {
      "parameters": {
        "jsCode": "\nconst caseData = $('M4 – Filter Cases Needing Follow-up1').item.json;\nlet msg = {};\ntry {\n  const raw = $json.text || '{}';\n  msg = JSON.parse(raw.replace(/```json/g,'').replace(/```/g,'').trim());\n} catch(e) {\n  const clientName = caseData['Client Name'] || 'Client';\n  msg = {\n    whatsapp_message: `Dear ${clientName}, this is a follow-up from ${$vars.FIRM_NAME} regarding your case. Please contact us at ${$vars.FIRM_PHONE}.`,\n    email_subject: `Follow-up – Case Update`,\n    email_body_html: `<p>Dear ${clientName},<br>Please contact us for an update on your case.</p>`,\n    escalate_to_lawyer: false,\n    escalation_note: ''\n  };\n}\nreturn [{json: {...caseData, ...msg, processed_at: new Date().toISOString()}}];\n"
      },
      "id": "217131da-82d1-4a54-9173-0bf7e340818d",
      "name": "M4 – Parse Follow-up Message1",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        -288,
        -496
      ],
      "notesInFlow": true,
      "notes": "Parses AI-generated follow-up message with fallback template."
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/{{$vars.WHATSAPP_PHONE_NUMBER_ID}}/messages",
        "sendHeaders": true,
        "headerParameters": {
          "parameters": [
            {
              "name": "Authorization",
              "value": "=Bearer {{$vars.WHATSAPP_ACCESS_TOKEN}}"
            }
          ]
        },
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={{ JSON.stringify({\n  messaging_product: \"whatsapp\",\n  to: $json['Phone'],\n  type: \"text\",\n  text: { body: $json.whatsapp_message }\n}) }}",
        "options": {}
      },
      "id": "06dffa26-9797-459c-888c-b3ee2af443aa",
      "name": "M4 – Send Follow-up WhatsApp1",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4,
      "position": [
        -64,
        -592
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "Sends personalized follow-up WhatsApp message to client."
    },
    {
      "parameters": {
        "subject": "={{ $json.email_subject }}",
        "message": "={{ $json.email_body_html }}",
        "options": {}
      },
      "id": "a7d23d57-c62b-4d6c-b14f-dc285de50dbe",
      "name": "M4 – Send Follow-up Email1",
      "type": "n8n-nodes-base.gmail",
      "typeVersion": 2,
      "position": [
        -64,
        -400
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "webhookId": "678f7709-bccc-42f2-96bf-e157cfd4a7c2",
      "onError": "continueErrorOutput",
      "notes": "Sends follow-up email to client."
    },
    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": false,
            "leftValue": "",
            "typeValidation": "strict"
          },
          "conditions": [
            {
              "id": "23b3c36b-fd72-4ae2-8a63-94afe7335ee1",
              "leftValue": "={{ String($json.escalate_to_lawyer) }}",
              "rightValue": "true",
              "operator": {
                "type": "string",
                "operation": "equals"
              }
            }
          ],
          "combinator": "and"
        },
        "options": {}
      },
      "id": "06c5f797-666d-4c92-ab9f-95cdf653f38a",
      "name": "M4 – Escalate to Lawyer?1",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2,
      "position": [
        160,
        -592
      ],
      "notesInFlow": true,
      "notes": "Routes high-priority unresponsive cases to lawyer escalation."
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://api.telegram.org/bot{{$vars.TELEGRAM_BOT_TOKEN}}/sendMessage",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={{ JSON.stringify({\n  chat_id: $vars.LAWYER_TELEGRAM_CHAT_ID,\n  text: `🚨 ESCALATION REQUIRED\\n\\nCase: ${$json['Case ID']}\\nClient: ${$json['Client Name']}\\nPhone: ${$json['Phone']}\\nStatus: ${$json['Status']}\\nPriority: ${$json.followup_priority?.toUpperCase()}\\nReason: ${$json.followup_reason}\\n\\nNote: ${$json.escalation_note}\\n\\nImmediate action required.`,\n  parse_mode: 'HTML'\n}) }}",
        "options": {}
      },
      "id": "4becd474-b1c2-4e2d-80e5-80ae8f5c04f3",
      "name": "M4 – Escalate Alert to Lawyer1",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4,
      "position": [
        384,
        -592
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "Sends escalation alert to lawyer via Telegram for high-priority unresponsive cases."
    },
    {
      "parameters": {
        "operation": "update",
        "documentId": {
          "__rl": true,
          "mode": "id",
          "value": "REPLACE_GOOGLE_SHEET_ID_CRM"
        },
        "sheetName": {
          "__rl": true,
          "mode": "name",
          "value": "Cases"
        },
        "columns": {
          "mappingMode": "defineBelow",
          "value": {
            "Case ID": "={{ $json['Case ID'] }}",
            "Last Contacted": "={{ new Date().toISOString().split('T')[0] }}",
            "Follow-up Count": "={{ parseInt($json['Follow-up Count'] || '0') + 1 }}"
          },
          "schema": [
            {
              "id": "Case ID",
              "displayName": "Case ID",
              "required": true,
              "defaultMatch": true,
              "canBeUsedToMatch": true
            },
            {
              "id": "Last Contacted",
              "displayName": "Last Contacted",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Follow-up Count",
              "displayName": "Follow-up Count",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            }
          ]
        },
        "options": {}
      },
      "id": "86ac91a4-ff13-498a-9411-ed0a0fea1f3d",
      "name": "M4 – Update Follow-up Count in CRM1",
      "type": "n8n-nodes-base.googleSheets",
      "typeVersion": 4,
      "position": [
        160,
        -400
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "Increments follow-up count and updates last contacted date in CRM."
    },
    {
      "parameters": {
        "httpMethod": "POST",
        "path": "law-case-status",
        "responseMode": "responseNode",
        "options": {}
      },
      "id": "cb567d2e-c1df-4e00-a210-7d426ac6778f",
      "name": "M4 – Case Status Update Webhook1",
      "type": "n8n-nodes-base.webhook",
      "typeVersion": 2,
      "position": [
        -1312,
        -80
      ],
      "notesInFlow": true,
      "webhookId": "ec33569c-cff3-4773-a25a-66d1d4bb7f47",
      "notes": "Receives case status updates. POST body: { case_id, new_status, notes, lawyer_name }. Status values: New, Active, Pending Documents, Scheduled, Hearing Scheduled, Awaiting Judgment, Judgment Received, Closed, On Hold."
    },
    {
      "parameters": {
        "respondWith": "json",
        "responseBody": "{\"status\":\"updated\"}",
        "options": {}
      },
      "id": "2fd391f4-259a-40ed-88a7-543d91314c60",
      "name": "M4 – Ack Status Update1",
      "type": "n8n-nodes-base.respondToWebhook",
      "typeVersion": 1,
      "position": [
        -1088,
        -176
      ]
    },
    {
      "parameters": {
        "jsCode": "\nconst b = $json.body || $json;\nconst statusMessages = {\n  'New': 'Your case has been registered with our firm.',\n  'Active': 'Your case is now being actively handled by our legal team.',\n  'Pending Documents': 'We require additional documents from you. Please check the details below.',\n  'Scheduled': 'A consultation has been scheduled for your case.',\n  'Hearing Scheduled': 'A hearing has been scheduled for your case. Please ensure you are available.',\n  'Awaiting Judgment': 'Your case has been heard. We are awaiting the judgment.',\n  'Judgment Received': 'A judgment has been received in your case. Please contact us to discuss next steps.',\n  'Closed': 'Your case has been closed. Thank you for trusting us with your legal matter.',\n  'On Hold': 'Your case is temporarily on hold. Our team will contact you with further updates.'\n};\n\nconst newStatus = b.new_status || 'Active';\nconst clientMessage = statusMessages[newStatus] || `Your case status has been updated to: ${newStatus}`;\n\nreturn [{json: {\n  case_id: b.case_id,\n  new_status: newStatus,\n  notes: b.notes || '',\n  lawyer_name: b.lawyer_name || '',\n  client_message: clientMessage,\n  update_timestamp: new Date().toISOString()\n}}];\n"
      },
      "id": "95e7e3d0-b41e-4bca-9a00-fad02a0b91af",
      "name": "M4 – Process Status Update1",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        -1088,
        32
      ],
      "notesInFlow": true,
      "notes": "Processes case status update and selects appropriate client notification message."
    },
    {
      "parameters": {
        "operation": "getAll",
        "documentId": {
          "__rl": true,
          "mode": "id",
          "value": "REPLACE_GOOGLE_SHEET_ID_CRM"
        }
      },
      "id": "80ef9945-aadb-4e64-94e8-9e82279eab27",
      "name": "M4 – Fetch Case Details1",
      "type": "n8n-nodes-base.googleSheets",
      "typeVersion": 4,
      "position": [
        -864,
        32
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "Fetches current case record to get client contact details."
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/{{$vars.WHATSAPP_PHONE_NUMBER_ID}}/messages",
        "sendHeaders": true,
        "headerParameters": {
          "parameters": [
            {
              "name": "Authorization",
              "value": "=Bearer {{$vars.WHATSAPP_ACCESS_TOKEN}}"
            }
          ]
        },
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={{ JSON.stringify({\n  messaging_product: \"whatsapp\",\n  to: $json['Phone'],\n  type: \"text\",\n  text: {\n    body: `Dear ${$json['Client Name']},\\n\\n📋 Case Update (${$('M4 – Process Status Update1').item.json.case_id})\\nStatus: ${$('M4 – Process Status Update1').item.json.new_status}\\n\\n${$('M4 – Process Status Update1').item.json.client_message}\\n\\n${$('M4 – Process Status Update1').item.json.notes ? 'Note: ' + $('M4 – Process Status Update1').item.json.notes : ''}\\n\\nContact: ${$vars.FIRM_PHONE}\\n${$vars.FIRM_NAME}`\n  }\n}) }}",
        "options": {}
      },
      "id": "e8fcf145-7e9a-41e6-97b3-63569d7bb277",
      "name": "M4 – Notify Client of Status Change1",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4,
      "position": [
        -640,
        32
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "Sends WhatsApp notification to client about case status change."
    },
    {
      "parameters": {
        "httpMethod": "POST",
        "path": "law-contract-review",
        "responseMode": "responseNode",
        "options": {}
      },
      "id": "14cf8170-7ff7-466b-899c-7c6c4ce28e40",
      "name": "M5 – Contract Review Webhook1",
      "type": "n8n-nodes-base.webhook",
      "typeVersion": 2,
      "position": [
        -1312,
        1232
      ],
      "notesInFlow": true,
      "webhookId": "362b429b-2ca2-4c64-8a26-66be48ea9f1f",
      "notes": "Receives contract documents for AI review. POST multipart/form-data: file + { intake_id, contract_type, client_name, review_urgency }."
    },
    {
      "parameters": {
        "respondWith": "json",
        "responseBody": "{\"status\":\"received\",\"message\":\"Contract received for review. AI analysis will be ready for lawyer review within 5-10 minutes.\"}",
        "options": {}
      },
      "id": "234bb8d0-cd62-4fea-a874-8a7eb31bda17",
      "name": "M5 – Ack Contract Upload1",
      "type": "n8n-nodes-base.respondToWebhook",
      "typeVersion": 1,
      "position": [
        -1088,
        1136
      ],
      "notesInFlow": true,
      "notes": "Immediately ACKs contract review request."
    },
    {
      "parameters": {
        "jsCode": "\nconst body = $json.body || $json;\nconst binaryData = $binary || {};\nconst fileKey = Object.keys(binaryData)[0] || 'data';\nconst file = binaryData[fileKey] || {};\n\nreturn [{\n  json: {\n    review_id: 'REV-' + Date.now(),\n    intake_id: body.intake_id || '',\n    client_name: body.client_name || 'Unknown',\n    contract_type: body.contract_type || 'Contract',\n    review_urgency: body.review_urgency || 'Normal',\n    original_filename: file.fileName || 'contract',\n    mime_type: file.mimeType || 'application/pdf',\n    upload_timestamp: new Date().toISOString(),\n    review_status: 'pending_ai_review'\n  },\n  binary: binaryData\n}];\n"
      },
      "id": "57a2ae11-b5ef-47b1-b0c4-9a1b51c75f02",
      "name": "M5 – Prepare Contract Metadata1",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        -1088,
        1328
      ],
      "notesInFlow": true,
      "notes": "Prepares contract metadata for AI review pipeline."
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://documentai.googleapis.com/v1/projects/{{$vars.GOOGLE_PROJECT_ID}}/locations/us/processors/{{$vars.DOCUMENT_AI_PROCESSOR_ID}}:process",
        "sendHeaders": true,
        "headerParameters": {
          "parameters": [
            {
              "name": "Authorization",
              "value": "=Bearer {{$vars.GOOGLE_DOCUMENT_AI_TOKEN}}"
            }
          ]
        },
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={{ JSON.stringify({\n  rawDocument: {\n    content: $binary.data?.data || '',\n    mimeType: $json.mime_type || 'application/pdf'\n  }\n}) }}",
        "options": {}
      },
      "id": "2538d406-b247-4629-9c29-514efbeb7773",
      "name": "M5 – Extract Contract Text (OCR)1",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4,
      "position": [
        -864,
        1328
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "Extracts contract text via Google Document AI. Same OCR endpoint as M3. Replace with your OCR provider."
    },
    {
      "parameters": {
        "jsCode": "\nconst meta = $('M5 – Prepare Contract Metadata1').item.json;\nlet text = '';\ntry {\n  const resp = $json;\n  text = resp?.document?.text || resp?.text || resp?.content || '';\n} catch(e) {\n  text = '';\n}\nconst truncated = text.substring(0, 10000);\nreturn [{json: {\n  ...meta,\n  contract_text: truncated,\n  was_truncated: text.length > 10000,\n  text_length: text.length\n}}];\n"
      },
      "id": "fa595952-be46-42b9-be50-fac26ed327e0",
      "name": "M5 – Prepare Contract Text1",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        -640,
        1328
      ],
      "notesInFlow": true,
      "notes": "Extracts text from OCR response and prepares for contract review LLM."
    },
    {
      "parameters": {
        "model": "llama-3.3-70b-versatile",
        "options": {
          "maxTokens": 3000,
          "temperature": 0.1
        }
      },
      "id": "20fcbd97-4d7c-4321-9fc2-56503796578e",
      "name": "M5 – Groq LLM (Contract Review)1",
      "type": "@n8n/n8n-nodes-langchain.lmChatOpenAi",
      "typeVersion": 1,
      "position": [
        -352,
        1552
      ],
      "notesInFlow": true,
      "notes": "Groq LLaMA for contract clause analysis."
    },
    {
      "parameters": {
        "prompt": "You are a legal document review assistant helping prepare a contract summary for lawyer review. Analyze the following contract and return a structured JSON review ONLY — no prose, no markdown fences.\n\nContract Type: {{ $json.contract_type }}\nClient: {{ $json.client_name }}\n{{ $json.was_truncated ? '⚠️ Document was truncated at 10,000 characters.' : '' }}\n\nContract Text:\n{{ $json.contract_text }}\n\nReturn this exact JSON:\n{\n  \"contract_overview\": {\n    \"contract_type\": \"<identified contract type>\",\n    \"parties\": [{\"role\": \"<role>\", \"name\": \"<party name>\"}],\n    \"effective_date\": \"<date or 'Not specified'>\",\n    \"expiry_date\": \"<date or 'Not specified'>\",\n    \"governing_law\": \"<jurisdiction>\"\n  },\n  \"clause_analysis\": [\n    {\n      \"clause_name\": \"<clause name>\",\n      \"finding\": \"<what the clause says in plain English>\",\n      \"risk_level\": \"<High|Medium|Low|Neutral>\",\n      \"explanation\": \"<why this is notable or risky>\",\n      \"recommendation\": \"<what the lawyer should consider>\"\n    }\n  ],\n  \"key_obligations\": {\n    \"party_a\": [\"<obligation 1>\", \"<obligation 2>\"],\n    \"party_b\": [\"<obligation 1>\", \"<obligation 2>\"]\n  },\n  \"important_dates_and_deadlines\": [\n    {\"description\": \"<event>\", \"date\": \"<date>\", \"urgency\": \"<High|Medium|Low>\"}\n  ],\n  \"payment_terms\": {\n    \"amount\": \"<amount or 'Not specified'>\",\n    \"schedule\": \"<payment schedule>\",\n    \"penalties\": \"<penalty clauses if any>\"\n  },\n  \"termination_conditions\": [\"<condition 1>\", \"<condition 2>\"],\n  \"risk_summary\": {\n    \"overall_risk\": \"<High|Medium|Low>\",\n    \"high_risk_clauses\": [\"<clause name 1>\"],\n    \"missing_standard_clauses\": [\"<missing clause if any>\"],\n    \"red_flags\": [\"<red flag if any>\"]\n  },\n  \"executive_summary\": \"<3-5 sentence plain-language summary for lawyer review>\",\n  \"requires_immediate_attention\": <true|false>,\n  \"attention_reason\": \"<reason if true>\",\n  \"ai_disclaimer\": \"This contract review is AI-generated for lawyer review assistance only. It does not constitute legal advice. All findings MUST be independently reviewed and verified by a qualified lawyer before any legal action or advice is given to the client.\"\n}\n\nCRITICAL: This output is for LAWYER REVIEW ONLY. Never present AI findings as final legal advice. Mark all findings clearly as preliminary AI analysis requiring lawyer verification."
      },
      "id": "ba0f5029-009d-464b-bd7a-f5a8322042f4",
      "name": "M5 – AI Contract Clause Review1",
      "type": "@n8n/n8n-nodes-langchain.chainLlm",
      "typeVersion": 1,
      "position": [
        -416,
        1328
      ],
      "notesInFlow": true,
      "notes": "Performs AI contract clause analysis using Groq LLaMA. All output is marked for lawyer review only."
    },
    {
      "parameters": {
        "jsCode": "\nconst meta = $('M5 – Prepare Contract Text1').item.json;\nlet review = {};\ntry {\n  const raw = $json.text || '{}';\n  review = JSON.parse(raw.replace(/```json/g,'').replace(/```/g,'').trim());\n} catch(e) {\n  review = {\n    executive_summary: 'AI review parsing failed. Manual review required.',\n    risk_summary: { overall_risk: 'Unknown', high_risk_clauses: [], red_flags: [] },\n    clause_analysis: [],\n    ai_disclaimer: 'Parsing failed. Full manual review required.',\n    requires_immediate_attention: true,\n    attention_reason: 'AI parsing failure — manual review required'\n  };\n}\nreturn [{json: {\n  ...meta,\n  ...review,\n  review_generated_at: new Date().toISOString(),\n  review_status: 'pending_lawyer_approval'\n}}];\n"
      },
      "id": "a88587de-5156-4efc-a634-d354dd8ef96f",
      "name": "M5 – Parse Contract Review1",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        -64,
        1328
      ],
      "notesInFlow": true,
      "notes": "Parses contract review JSON with fallback. Status set to pending_lawyer_approval."
    },
    {
      "parameters": {
        "operation": "appendOrUpdate",
        "documentId": {
          "__rl": true,
          "mode": "id",
          "value": "REPLACE_GOOGLE_SHEET_ID_CRM"
        },
        "sheetName": {
          "__rl": true,
          "mode": "name",
          "value": "Contract Reviews"
        },
        "columns": {
          "mappingMode": "defineBelow",
          "value": {
            "Review ID": "={{ $json.review_id }}",
            "Intake ID": "={{ $json.intake_id }}",
            "Client Name": "={{ $json.client_name }}",
            "Contract Type": "={{ $json.contract_type }}",
            "Overall Risk": "={{ $json.risk_summary?.overall_risk || 'Unknown' }}",
            "Requires Attention": "={{ $json.requires_immediate_attention ? 'YES' : 'No' }}",
            "Executive Summary": "={{ $json.executive_summary }}",
            "High Risk Clauses": "={{ ($json.risk_summary?.high_risk_clauses || []).join(', ') }}",
            "Red Flags": "={{ ($json.risk_summary?.red_flags || []).join(', ') }}",
            "Review Status": "Pending Lawyer Approval",
            "Review Generated At": "={{ $json.review_generated_at }}",
            "Lawyer Reviewed": "No",
            "Lawyer Notes": "",
            "AI Disclaimer": "={{ $json.ai_disclaimer }}"
          },
          "schema": [
            {
              "id": "Review ID",
              "displayName": "Review ID",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": true
            },
            {
              "id": "Intake ID",
              "displayName": "Intake ID",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Client Name",
              "displayName": "Client Name",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Contract Type",
              "displayName": "Contract Type",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Overall Risk",
              "displayName": "Overall Risk",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Requires Attention",
              "displayName": "Requires Attention",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Executive Summary",
              "displayName": "Executive Summary",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "High Risk Clauses",
              "displayName": "High Risk Clauses",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Red Flags",
              "displayName": "Red Flags",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Review Status",
              "displayName": "Review Status",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Review Generated At",
              "displayName": "Review Generated At",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Lawyer Reviewed",
              "displayName": "Lawyer Reviewed",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "Lawyer Notes",
              "displayName": "Lawyer Notes",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            },
            {
              "id": "AI Disclaimer",
              "displayName": "AI Disclaimer",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": false
            }
          ]
        },
        "options": {}
      },
      "id": "6cd0fa22-25ec-41b9-a871-d78a7c2ddf69",
      "name": "M5 – Store Contract Review in Sheets1",
      "type": "n8n-nodes-base.googleSheets",
      "typeVersion": 4,
      "position": [
        160,
        1328
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "Stores AI contract review in Contract Reviews tab. Status: Pending Lawyer Approval."
    },
    {
      "parameters": {
        "subject": "=🔍 AI Contract Review Ready – {{ $json.client_name }} | Risk: {{ $json.risk_summary?.overall_risk }} | REQUIRES LAWYER APPROVAL",
        "message": "=<h2>⚠️ AI Contract Review – Lawyer Approval Required</h2>\n<p style=\"background:#fff3cd;padding:10px;border-radius:4px\"><strong>IMPORTANT:</strong> This review was generated by AI for administrative assistance only. All findings MUST be independently verified by a qualified lawyer. Do not share directly with the client without lawyer review and approval.</p>\n\n<table border=\"1\" cellpadding=\"6\">\n<tr><th>Review ID</th><td>{{ $json.review_id }}</td></tr>\n<tr><th>Client</th><td>{{ $json.client_name }}</td></tr>\n<tr><th>Contract Type</th><td>{{ $json.contract_type }}</td></tr>\n<tr><th>Overall Risk</th><td style=\"color:{{ $json.risk_summary?.overall_risk === 'High' ? 'red' : $json.risk_summary?.overall_risk === 'Medium' ? 'orange' : 'green' }}\"><strong>{{ $json.risk_summary?.overall_risk }}</strong></td></tr>\n<tr><th>Immediate Attention</th><td>{{ $json.requires_immediate_attention ? '⚠️ YES – ' + $json.attention_reason : 'No' }}</td></tr>\n</table>\n\n<h3>Executive Summary</h3><p>{{ $json.executive_summary }}</p>\n\n<h3>High Risk Clauses</h3><ul>{{ ($json.risk_summary?.high_risk_clauses || []).map(c => `<li>${c}</li>`).join('') || '<li>None identified</li>' }}</ul>\n\n<h3>Red Flags</h3><ul>{{ ($json.risk_summary?.red_flags || []).map(f => `<li>${f}</li>`).join('') || '<li>None identified</li>' }}</ul>\n\n<h3>Clause Analysis ({{ ($json.clause_analysis || []).length }} clauses)</h3>\n{{ ($json.clause_analysis || []).map(c => `<div style=\"margin:8px 0;padding:8px;border-left:4px solid ${c.risk_level==='High'?'red':c.risk_level==='Medium'?'orange':'green'}\"><strong>${c.clause_name}</strong> [${c.risk_level} Risk]<br>${c.finding}<br><em>Note: ${c.explanation}</em></div>`).join('') }}\n\n<h3>Payment Terms</h3><p>{{ $json.payment_terms?.amount || 'Not specified' }} | {{ $json.payment_terms?.schedule || 'N/A' }}</p>\n\n<h3>Termination Conditions</h3><ul>{{ ($json.termination_conditions || []).map(t => `<li>${t}</li>`).join('') }}</ul>\n\n<p><strong>To approve and send to client, reply to this email or update the review status in the CRM.</strong></p>\n<p><em>{{ $json.ai_disclaimer }}</em></p>",
        "options": {}
      },
      "id": "9e5467ce-b176-4613-9d95-8853444dbf8a",
      "name": "M5 – Send Review to Lawyer (Human Approval)1",
      "type": "n8n-nodes-base.gmail",
      "typeVersion": 2,
      "position": [
        384,
        1232
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "webhookId": "4afa57f1-d9eb-4040-b506-c58058f40efa",
      "onError": "continueErrorOutput",
      "notes": "HUMAN APPROVAL CHECKPOINT: Sends full AI contract review to lawyer. Client is NOT notified until lawyer approves. Lawyer must update status in CRM to 'Lawyer Approved' to trigger client notification."
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://api.telegram.org/bot{{$vars.TELEGRAM_BOT_TOKEN}}/sendMessage",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={{ JSON.stringify({\n  chat_id: $vars.LAWYER_TELEGRAM_CHAT_ID,\n  text: `📄 Contract Review Ready\\n\\nClient: ${$json.client_name}\\nType: ${$json.contract_type}\\nRisk: ${$json.risk_summary?.overall_risk || 'Unknown'}\\n${$json.requires_immediate_attention ? '⚠️ URGENT: ' + $json.attention_reason : ''}\\n\\nReview ID: ${$json.review_id}\\n\\nCheck email for full AI analysis. APPROVAL REQUIRED before client delivery.`,\n  parse_mode: 'HTML'\n}) }}",
        "options": {}
      },
      "id": "d3d17b1c-3797-430a-8b73-517a5f7e62b9",
      "name": "M5 – Telegram Alert – Contract Review Ready1",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4,
      "position": [
        384,
        1424
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "Sends Telegram alert to lawyer that contract review is ready for approval."
    },
    {
      "parameters": {
        "httpMethod": "POST",
        "path": "law-review-approve",
        "responseMode": "responseNode",
        "options": {}
      },
      "id": "163c61a6-fa21-4f5e-b453-31af8c5b8921",
      "name": "M5 – Review Approval Webhook1",
      "type": "n8n-nodes-base.webhook",
      "typeVersion": 2,
      "position": [
        -1312,
        2272
      ],
      "notesInFlow": true,
      "webhookId": "640e927d-d5ad-4cf3-bf9d-3c9d3be87101",
      "notes": "Lawyer triggers this after reviewing contract AI output. POST: { review_id, approved, lawyer_notes, send_to_client }. This is the human-in-the-loop approval gate."
    },
    {
      "parameters": {
        "respondWith": "json",
        "responseBody": "{\"status\":\"approval_received\"}",
        "options": {}
      },
      "id": "4dbbaf4e-1d33-4f45-8c5b-44dd8f39a819",
      "name": "M5 – Ack Approval1",
      "type": "n8n-nodes-base.respondToWebhook",
      "typeVersion": 1,
      "position": [
        -1088,
        2176
      ]
    },
    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": false,
            "leftValue": "",
            "typeValidation": "strict"
          },
          "conditions": [
            {
              "id": "b2840516-5cb5-416d-a0bc-2d8c8a718e5c",
              "leftValue": "={{ String(($json.body || $json).approved) }}",
              "rightValue": "true",
              "operator": {
                "type": "string",
                "operation": "equals"
              }
            }
          ],
          "combinator": "and"
        },
        "options": {}
      },
      "id": "14a636de-56b3-4357-a756-541c2b11f5de",
      "name": "M5 – Approved by Lawyer?1",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2,
      "position": [
        -1088,
        2368
      ],
      "notesInFlow": true,
      "notes": "Routes based on lawyer approval. True = approved, send client summary. False = rejected, notify admin."
    },
    {
      "parameters": {
        "operation": "getAll",
        "documentId": {
          "__rl": true,
          "mode": "id",
          "value": "REPLACE_GOOGLE_SHEET_ID_CRM"
        }
      },
      "id": "92669c63-bca2-4dc6-8a39-b94dbf07de08",
      "name": "M5 – Fetch Approved Review1",
      "type": "n8n-nodes-base.googleSheets",
      "typeVersion": 4,
      "position": [
        -864,
        2368
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "Fetches the approved review record to send client summary."
    },
    {
      "parameters": {
        "subject": "=Contract Review Summary – {{ $json['Client Name'] }}",
        "message": "=<h2>Contract Review Summary</h2>\n<p>Dear Team,</p>\n<p>The lawyer-approved contract review summary for <strong>{{ $json['Client Name'] }}</strong> is ready.</p>\n<p><strong>Contract:</strong> {{ $json['Contract Type'] }}<br>\n<strong>Overall Risk Assessment:</strong> {{ $json['Overall Risk'] }}</p>\n<h3>Key Findings</h3>\n<p>{{ $json['Executive Summary'] }}</p>\n<h3>Points Requiring Your Attention</h3>\n<p>{{ $json['High Risk Clauses'] ? 'Key clauses: ' + $json['High Risk Clauses'] : 'No high-risk clauses identified.' }}</p>\n<p><strong>Lawyer Notes:</strong> {{ $json['Lawyer Notes'] || 'None' }}</p>\n<p><em>⚠️ This summary has been prepared and reviewed by our legal team. For specific legal advice, please schedule a consultation.</em></p>\n<p>Review ID: {{ $json['Review ID'] }}</p>",
        "options": {}
      },
      "id": "467070b5-6de0-48dd-b4a3-75b4f7c02182",
      "name": "M5 – Send Lawyer-Approved Summary to Client1",
      "type": "n8n-nodes-base.gmail",
      "typeVersion": 2,
      "position": [
        -640,
        2368
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "webhookId": "c3e607b7-9b91-467c-8fbf-a68f8ef2ada2",
      "onError": "continueErrorOutput",
      "notes": "Sends lawyer-approved contract summary to client. Template — update To field to client email from CRM."
    },
    {
      "parameters": {
        "httpMethod": "POST",
        "path": "whatsapp-inbound",
        "responseMode": "responseNode",
        "options": {}
      },
      "id": "5a3f8c9e-36ab-4e1a-a1f1-37a5863f79b9",
      "name": "WA – Inbound WhatsApp Webhook1",
      "type": "n8n-nodes-base.webhook",
      "typeVersion": 2,
      "position": [
        -1312,
        3424
      ],
      "notesInFlow": true,
      "webhookId": "43fcfa24-ddd2-4e62-bd61-87c65534adce",
      "notes": "Meta WhatsApp Business webhook for inbound messages. Configure in Meta Developer Portal → WhatsApp → Webhook. Verify token = REPLACE_WA_VERIFY_TOKEN."
    },
    {
      "parameters": {
        "path": "whatsapp-verify",
        "responseMode": "responseNode",
        "options": {}
      },
      "id": "1fe0237a-9e11-4df5-bc0e-6e3c36c29fdb",
      "name": "WA – Handle Webhook Verification (GET)1",
      "type": "n8n-nodes-base.webhook",
      "typeVersion": 2,
      "position": [
        -1312,
        4288
      ],
      "notesInFlow": true,
      "webhookId": "4b879a9d-2014-4864-83d4-4eaebcfc933f",
      "notes": "Handles Meta's GET verification challenge. Returns hub.challenge when hub.verify_token matches."
    },
    {
      "parameters": {
        "respondWith": "text",
        "responseBody": "={{ $json.query?.['hub.challenge'] || '' }}",
        "options": {}
      },
      "id": "e5d355c5-ddfa-4ebb-b600-40065474d767",
      "name": "WA – Respond to Verification1",
      "type": "n8n-nodes-base.respondToWebhook",
      "typeVersion": 1,
      "position": [
        -1088,
        4288
      ],
      "notesInFlow": true,
      "notes": "Returns hub.challenge for Meta webhook verification. Customize to check hub.verify_token == REPLACE_WA_VERIFY_TOKEN."
    },
    {
      "parameters": {
        "respondWith": "json",
        "responseBody": "{\"status\":\"ok\"}",
        "options": {}
      },
      "id": "743f79ba-3a46-4dee-89e3-ea76418ee38d",
      "name": "WA – Ack WhatsApp ",
      "type": "n8n-nodes-base.respondToWebhook",
      "typeVersion": 1,
      "position": [
        -1088,
        3328
      ],
      "notesInFlow": true,
      "notes": "Immediately returns 200 to Meta (required within 20s or Meta retries)."
    },
    {
      "parameters": {
        "jsCode": "\nconst body = $json.body || $json;\n// Meta WhatsApp webhook structure\nconst entry = (body.entry || [])[0] || {};\nconst changes = (entry.changes || [])[0] || {};\nconst value = changes.value || {};\nconst messages = value.messages || [];\nconst msg = messages[0] || {};\nconst contact = (value.contacts || [])[0] || {};\n\nif (!msg.id) {\n  // Not a message event (could be status update) — return empty to skip\n  return [];\n}\n\nreturn [{json: {\n  message_id: msg.id,\n  from_phone: msg.from || '',\n  from_name: contact.profile?.name || 'Unknown',\n  message_type: msg.type || 'text',\n  message_text: msg.text?.body || msg.interactive?.button_reply?.title || '',\n  timestamp: msg.timestamp || '',\n  phone_number_id: value.metadata?.phone_number_id || '',\n  received_at: new Date().toISOString()\n}}];\n"
      },
      "id": "60b65a5f-11e1-41ff-b1a9-f977194cd72e",
      "name": "WA – Parse Inbound Message1",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        -1088,
        3520
      ],
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "Parses Meta WhatsApp inbound webhook payload. Returns empty array for non-message events (e.g. delivery receipts)."
    },
    {
      "parameters": {
        "model": "llama-3.3-70b-versatile",
        "options": {
          "maxTokens": 500,
          "temperature": 0.6
        }
      },
      "id": "0cf0cb26-ed8e-4072-824f-f962f19f7e78",
      "name": "WA – Groq LLM (Receptionist)1",
      "type": "@n8n/n8n-nodes-langchain.lmChatOpenAi",
      "typeVersion": 1,
      "position": [
        -800,
        3648
      ],
      "notesInFlow": true,
      "notes": "Groq LLaMA for WhatsApp AI receptionist responses."
    },
    {
      "parameters": {
        "prompt": "You are a professional AI receptionist for {{ $vars.FIRM_NAME }}, a law firm. Respond to the client's WhatsApp message in a helpful, professional, and empathetic manner.\n\nClient Name: {{ $json.from_name }}\nClient Phone: {{ $json.from_phone }}\nClient Message: {{ $json.message_text }}\nCurrent Time: {{ $now.toISO() }}\n\nFirm Contact: {{ $vars.FIRM_PHONE }}\nFirm Address: {{ $vars.FIRM_ADDRESS }}\n\nRULES:\n1. Be warm, professional, and empathetic\n2. NEVER provide legal advice or legal opinions\n3. For legal questions, always direct to schedule a consultation\n4. You CAN help with: appointment scheduling, firm information, document submission guidance, general process information\n5. For urgent matters, provide the emergency contact\n6. Keep response under 200 words for WhatsApp\n7. Respond in the same language as the client message (support Hindi, English, and regional languages)\n8. If the client wants to submit a case, ask them to share: Name, Phone, Email, and brief description of their legal issue\n\nRespond naturally as a receptionist. Do not add any JSON or formatting — plain text only."
      },
      "id": "7f74d6d3-c36f-4d39-98ed-6eae62967d91",
      "name": "WA – AI Receptionist Response1",
      "type": "@n8n/n8n-nodes-langchain.chainLlm",
      "typeVersion": 1,
      "position": [
        -864,
        3424
      ],
      "notesInFlow": true,
      "notes": "AI receptionist for WhatsApp. Does not provide legal advice. Directs clients to appropriate actions."
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v19.0/{{$json.phone_number_id}}/messages",
        "sendHeaders": true,
        "headerParameters": {
          "parameters": [
            {
              "name": "Authorization",
              "value": "=Bearer {{$vars.WHATSAPP_ACCESS_TOKEN}}"
            }
          ]
        },
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={{ JSON.stringify({\n  messaging_product: \"whatsapp\",\n  to: $('WA – Parse Inbound Message1').item.json.from_phone,\n  type: \"text\",\n  text: { body: $json.text || 'Thank you for contacting us. A team member will reach out shortly.' }\n}) }}",
        "options": {}
      },
      "id": "cac16275-f320-4644-bd1d-eeb34e6998ce",
      "name": "WA – Send AI Reply to Client1",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4,
      "position": [
        -512,
        3520
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000,
      "notesInFlow": true,
      "onError": "continueErrorOutput",
      "notes": "Sends AI-generated receptionist reply to client via WhatsApp."
    }
  ],
  "pinData": {},
  "connections": {
    "Error Trigger": {
      "main": [
        [
          {
            "node": "Format Error Alert",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Format Error Alert": {
      "main": [
        [
          {
            "node": "Send Error to Telegram",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M1 – Intake Webhook": {
      "main": [
        [
          {
            "node": "M1 – Validate Intake Input",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M1 – Validate Intake Input": {
      "main": [
        [
          {
            "node": "M1 – Classify & Qualify Lead",
            "type": "main",
            "index": 0
          }
        ],
        [
          {
            "node": "M1 – Intake Validation Error Response",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M1 – Groq LLM (Classify)": {
      "ai_languageModel": [
        [
          {
            "node": "M1 – Classify & Qualify Lead",
            "type": "ai_languageModel",
            "index": 0
          }
        ]
      ]
    },
    "M1 – Classify & Qualify Lead": {
      "main": [
        [
          {
            "node": "M1 – Parse Classification JSON",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M1 – Parse Classification JSON": {
      "main": [
        [
          {
            "node": "M1 – Store Lead in CRM Sheet",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M1 – Store Lead in CRM Sheet": {
      "main": [
        [
          {
            "node": "M1 – Notify Lawyer via Telegram",
            "type": "main",
            "index": 0
          },
          {
            "node": "M1 – Notify Lawyer via Email",
            "type": "main",
            "index": 0
          },
          {
            "node": "M1 – Send WhatsApp Ack to Client",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M1 – Notify Lawyer via Telegram": {
      "main": [
        [
          {
            "node": "M1 – Intake Success Response",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M2 – Voice Webhook (Sarvam Callback)": {
      "main": [
        [
          {
            "node": "M2 – Ack Sarvam 200",
            "type": "main",
            "index": 0
          },
          {
            "node": "M2 – Parse Voice Call Data",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M2 – Groq LLM (Voice Extract)": {
      "ai_languageModel": [
        [
          {
            "node": "M2 – Extract Client Info from Transcript",
            "type": "ai_languageModel",
            "index": 0
          }
        ]
      ]
    },
    "M2 – Parse Voice Call Data": {
      "main": [
        [
          {
            "node": "M2 – Extract Client Info from Transcript",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M2 – Extract Client Info from Transcript": {
      "main": [
        [
          {
            "node": "M2 – Merge Voice + AI Extraction",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M2 – Merge Voice + AI Extraction": {
      "main": [
        [
          {
            "node": "M2 – Appointment Requested?",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M2 – Appointment Requested?": {
      "main": [
        [
          {
            "node": "M2 – Store Voice Lead in CRM",
            "type": "main",
            "index": 0
          },
          {
            "node": "M2 – Check Calendar Availability",
            "type": "main",
            "index": 0
          }
        ],
        [
          {
            "node": "M2 – Store Voice Lead in CRM",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M2 – Check Calendar Availability": {
      "main": [
        [
          {
            "node": "M2 – Create Appointment in Calendar",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M2 – Create Appointment in Calendar": {
      "main": [
        [
          {
            "node": "M2 – Send Appointment Confirmation WA",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M3 – Document Upload Webhook": {
      "main": [
        [
          {
            "node": "M3 – Ack Document Upload",
            "type": "main",
            "index": 0
          },
          {
            "node": "M3 – Extract Document Text",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M3 – Extract Document Text": {
      "main": [
        [
          {
            "node": "M3 – OCR / Text Extraction API",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M3 – OCR / Text Extraction API": {
      "main": [
        [
          {
            "node": "M3 – Prepare Extracted Text",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M3 – Groq LLM (Summarize)": {
      "ai_languageModel": [
        [
          {
            "node": "M3 – AI Document Summarization",
            "type": "ai_languageModel",
            "index": 0
          }
        ]
      ]
    },
    "M3 – Prepare Extracted Text": {
      "main": [
        [
          {
            "node": "M3 – AI Document Summarization",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M3 – AI Document Summarization": {
      "main": [
        [
          {
            "node": "M3 – Parse Document Summary",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M3 – Parse Document Summary": {
      "main": [
        [
          {
            "node": "M3 – Upload Original Doc to Drive",
            "type": "main",
            "index": 0
          },
          {
            "node": "M3 – Store Summary in Sheets",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M3 – Store Summary in Sheets": {
      "main": [
        [
          {
            "node": "M3 – Missing Docs?",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M3 – Missing Docs?": {
      "main": [
        [
          {
            "node": "M3 – Notify Lawyer – Missing Docs",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M4 – Daily Follow-up Scheduler": {
      "main": [
        [
          {
            "node": "M4 – Fetch Active Cases from CRM",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M4 – Fetch Active Cases from CRM": {
      "main": [
        [
          {
            "node": "M4 – Filter Cases Needing Follow-up",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M4 – Groq LLM (Follow-up Msg)": {
      "ai_languageModel": [
        [
          {
            "node": "M4 – Generate Personalized Follow-up",
            "type": "ai_languageModel",
            "index": 0
          }
        ]
      ]
    },
    "M4 – Filter Cases Needing Follow-up": {
      "main": [
        [
          {
            "node": "M4 – Generate Personalized Follow-up",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M4 – Generate Personalized Follow-up": {
      "main": [
        [
          {
            "node": "M4 – Parse Follow-up Message",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M4 – Parse Follow-up Message": {
      "main": [
        [
          {
            "node": "M4 – Send Follow-up WhatsApp",
            "type": "main",
            "index": 0
          },
          {
            "node": "M4 – Send Follow-up Email",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M4 – Send Follow-up WhatsApp": {
      "main": [
        [
          {
            "node": "M4 – Escalate to Lawyer?",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M4 – Escalate to Lawyer?": {
      "main": [
        [
          {
            "node": "M4 – Escalate Alert to Lawyer",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M4 – Send Follow-up Email": {
      "main": [
        [
          {
            "node": "M4 – Update Follow-up Count in CRM",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M4 – Case Status Update Webhook": {
      "main": [
        [
          {
            "node": "M4 – Ack Status Update",
            "type": "main",
            "index": 0
          },
          {
            "node": "M4 – Process Status Update",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M4 – Process Status Update": {
      "main": [
        [
          {
            "node": "M4 – Fetch Case Details",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M4 – Fetch Case Details": {
      "main": [
        [
          {
            "node": "M4 – Notify Client of Status Change",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M5 – Contract Review Webhook": {
      "main": [
        [
          {
            "node": "M5 – Ack Contract Upload",
            "type": "main",
            "index": 0
          },
          {
            "node": "M5 – Prepare Contract Metadata",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M5 – Prepare Contract Metadata": {
      "main": [
        [
          {
            "node": "M5 – Extract Contract Text (OCR)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M5 – Extract Contract Text (OCR)": {
      "main": [
        [
          {
            "node": "M5 – Prepare Contract Text",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M5 – Groq LLM (Contract Review)": {
      "ai_languageModel": [
        [
          {
            "node": "M5 – AI Contract Clause Review",
            "type": "ai_languageModel",
            "index": 0
          }
        ]
      ]
    },
    "M5 – Prepare Contract Text": {
      "main": [
        [
          {
            "node": "M5 – AI Contract Clause Review",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M5 – AI Contract Clause Review": {
      "main": [
        [
          {
            "node": "M5 – Parse Contract Review",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M5 – Parse Contract Review": {
      "main": [
        [
          {
            "node": "M5 – Store Contract Review in Sheets",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M5 – Store Contract Review in Sheets": {
      "main": [
        [
          {
            "node": "M5 – Send Review to Lawyer (Human Approval)",
            "type": "main",
            "index": 0
          },
          {
            "node": "M5 – Telegram Alert – Contract Review Ready",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M5 – Review Approval Webhook": {
      "main": [
        [
          {
            "node": "M5 – Ack Approval",
            "type": "main",
            "index": 0
          },
          {
            "node": "M5 – Approved by Lawyer?",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M5 – Approved by Lawyer?": {
      "main": [
        [
          {
            "node": "M5 – Fetch Approved Review",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M5 – Fetch Approved Review": {
      "main": [
        [
          {
            "node": "M5 – Send Lawyer-Approved Summary to Client",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "WA – Handle Webhook Verification (GET)": {
      "main": [
        [
          {
            "node": "WA – Respond to Verification",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "WA – Inbound WhatsApp Webhook": {
      "main": [
        [
          {
            "node": "WA – Ack WhatsApp 200",
            "type": "main",
            "index": 0
          },
          {
            "node": "WA – Parse Inbound Message",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "WA – Groq LLM (Receptionist)": {
      "ai_languageModel": [
        [
          {
            "node": "WA – AI Receptionist Response",
            "type": "ai_languageModel",
            "index": 0
          }
        ]
      ]
    },
    "WA – Parse Inbound Message": {
      "main": [
        [
          {
            "node": "WA – AI Receptionist Response",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "WA – AI Receptionist Response": {
      "main": [
        [
          {
            "node": "WA – Send AI Reply to Client",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Format Error Alert1": {
      "main": [
        [
          {
            "node": "Send Error to Telegram1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M1 – Intake Webhook1": {
      "main": [
        [
          {
            "node": "M1 – Validate Intake Input1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M1 – Validate Intake Input1": {
      "main": [
        [
          {
            "node": "M1 – Classify & Qualify Lead1",
            "type": "main",
            "index": 0
          }
        ],
        [
          {
            "node": "M1 – Intake Validation Error Response1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M1 – Groq LLM (Classify)1": {
      "ai_languageModel": [
        [
          {
            "node": "M1 – Classify & Qualify Lead1",
            "type": "ai_languageModel",
            "index": 0
          }
        ]
      ]
    },
    "M1 – Classify & Qualify Lead1": {
      "main": [
        [
          {
            "node": "M1 – Parse Classification JSON1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M1 – Parse Classification JSON1": {
      "main": [
        [
          {
            "node": "M1 – Store Lead in CRM Sheet1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M1 – Store Lead in CRM Sheet1": {
      "main": [
        [
          {
            "node": "M1 – Notify Lawyer via Telegram1",
            "type": "main",
            "index": 0
          },
          {
            "node": "M1 – Notify Lawyer via Email1",
            "type": "main",
            "index": 0
          },
          {
            "node": "M1 – Send WhatsApp Ack to Client1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M1 – Notify Lawyer via Telegram1": {
      "main": [
        [
          {
            "node": "M1 – Intake Success Response1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M2 – Voice Webhook (Sarvam Callback)1": {
      "main": [
        [
          {
            "node": "M2 – Ack Sarvam ",
            "type": "main",
            "index": 0
          },
          {
            "node": "M2 – Parse Voice Call Data1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M2 – Parse Voice Call Data1": {
      "main": [
        [
          {
            "node": "M2 – Extract Client Info from Transcript1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M2 – Groq LLM (Voice Extract)1": {
      "ai_languageModel": [
        [
          {
            "node": "M2 – Extract Client Info from Transcript1",
            "type": "ai_languageModel",
            "index": 0
          }
        ]
      ]
    },
    "M2 – Extract Client Info from Transcript1": {
      "main": [
        [
          {
            "node": "M2 – Merge Voice + AI Extraction1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M2 – Merge Voice + AI Extraction1": {
      "main": [
        [
          {
            "node": "M2 – Appointment Requested?1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M2 – Appointment Requested?1": {
      "main": [
        [
          {
            "node": "M2 – Store Voice Lead in CRM1",
            "type": "main",
            "index": 0
          },
          {
            "node": "M2 – Check Calendar Availability1",
            "type": "main",
            "index": 0
          }
        ],
        [
          {
            "node": "M2 – Store Voice Lead in CRM1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M2 – Check Calendar Availability1": {
      "main": [
        [
          {
            "node": "M2 – Create Appointment in Calendar1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M2 – Create Appointment in Calendar1": {
      "main": [
        [
          {
            "node": "M2 – Send Appointment Confirmation WA1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M3 – Document Upload Webhook1": {
      "main": [
        [
          {
            "node": "M3 – Ack Document Upload1",
            "type": "main",
            "index": 0
          },
          {
            "node": "M3 – Extract Document Text1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M3 – Extract Document Text1": {
      "main": [
        [
          {
            "node": "M3 – OCR / Text Extraction API1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M3 – OCR / Text Extraction API1": {
      "main": [
        [
          {
            "node": "M3 – Prepare Extracted Text1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M3 – Prepare Extracted Text1": {
      "main": [
        [
          {
            "node": "M3 – AI Document Summarization1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M3 – Groq LLM (Summarize)1": {
      "ai_languageModel": [
        [
          {
            "node": "M3 – AI Document Summarization1",
            "type": "ai_languageModel",
            "index": 0
          }
        ]
      ]
    },
    "M3 – AI Document Summarization1": {
      "main": [
        [
          {
            "node": "M3 – Parse Document Summary1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M3 – Parse Document Summary1": {
      "main": [
        [
          {
            "node": "M3 – Upload Original Doc to Drive1",
            "type": "main",
            "index": 0
          },
          {
            "node": "M3 – Store Summary in Sheets1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M3 – Store Summary in Sheets1": {
      "main": [
        [
          {
            "node": "M3 – Missing Docs?1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M3 – Missing Docs?1": {
      "main": [
        [
          {
            "node": "M3 – Notify Lawyer – Missing Docs1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M4 – Daily Follow-up Scheduler1": {
      "main": [
        [
          {
            "node": "M4 – Fetch Active Cases from CRM1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M4 – Fetch Active Cases from CRM1": {
      "main": [
        [
          {
            "node": "M4 – Filter Cases Needing Follow-up1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M4 – Filter Cases Needing Follow-up1": {
      "main": [
        [
          {
            "node": "M4 – Generate Personalized Follow-up1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M4 – Groq LLM (Follow-up Msg)1": {
      "ai_languageModel": [
        [
          {
            "node": "M4 – Generate Personalized Follow-up1",
            "type": "ai_languageModel",
            "index": 0
          }
        ]
      ]
    },
    "M4 – Generate Personalized Follow-up1": {
      "main": [
        [
          {
            "node": "M4 – Parse Follow-up Message1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M4 – Parse Follow-up Message1": {
      "main": [
        [
          {
            "node": "M4 – Send Follow-up WhatsApp1",
            "type": "main",
            "index": 0
          },
          {
            "node": "M4 – Send Follow-up Email1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M4 – Send Follow-up WhatsApp1": {
      "main": [
        [
          {
            "node": "M4 – Escalate to Lawyer?1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M4 – Send Follow-up Email1": {
      "main": [
        [
          {
            "node": "M4 – Update Follow-up Count in CRM1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M4 – Escalate to Lawyer?1": {
      "main": [
        [
          {
            "node": "M4 – Escalate Alert to Lawyer1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M4 – Case Status Update Webhook1": {
      "main": [
        [
          {
            "node": "M4 – Ack Status Update1",
            "type": "main",
            "index": 0
          },
          {
            "node": "M4 – Process Status Update1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M4 – Process Status Update1": {
      "main": [
        [
          {
            "node": "M4 – Fetch Case Details1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M4 – Fetch Case Details1": {
      "main": [
        [
          {
            "node": "M4 – Notify Client of Status Change1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M5 – Contract Review Webhook1": {
      "main": [
        [
          {
            "node": "M5 – Ack Contract Upload1",
            "type": "main",
            "index": 0
          },
          {
            "node": "M5 – Prepare Contract Metadata1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M5 – Prepare Contract Metadata1": {
      "main": [
        [
          {
            "node": "M5 – Extract Contract Text (OCR)1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M5 – Extract Contract Text (OCR)1": {
      "main": [
        [
          {
            "node": "M5 – Prepare Contract Text1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M5 – Prepare Contract Text1": {
      "main": [
        [
          {
            "node": "M5 – AI Contract Clause Review1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M5 – Groq LLM (Contract Review)1": {
      "ai_languageModel": [
        [
          {
            "node": "M5 – AI Contract Clause Review1",
            "type": "ai_languageModel",
            "index": 0
          }
        ]
      ]
    },
    "M5 – AI Contract Clause Review1": {
      "main": [
        [
          {
            "node": "M5 – Parse Contract Review1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M5 – Parse Contract Review1": {
      "main": [
        [
          {
            "node": "M5 – Store Contract Review in Sheets1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M5 – Store Contract Review in Sheets1": {
      "main": [
        [
          {
            "node": "M5 – Send Review to Lawyer (Human Approval)1",
            "type": "main",
            "index": 0
          },
          {
            "node": "M5 – Telegram Alert – Contract Review Ready1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M5 – Review Approval Webhook1": {
      "main": [
        [
          {
            "node": "M5 – Ack Approval1",
            "type": "main",
            "index": 0
          },
          {
            "node": "M5 – Approved by Lawyer?1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M5 – Approved by Lawyer?1": {
      "main": [
        [
          {
            "node": "M5 – Fetch Approved Review1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "M5 – Fetch Approved Review1": {
      "main": [
        [
          {
            "node": "M5 – Send Lawyer-Approved Summary to Client1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "WA – Inbound WhatsApp Webhook1": {
      "main": [
        [
          {
            "node": "WA – Ack WhatsApp ",
            "type": "main",
            "index": 0
          },
          {
            "node": "WA – Parse Inbound Message1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "WA – Handle Webhook Verification (GET)1": {
      "main": [
        [
          {
            "node": "WA – Respond to Verification1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "WA – Parse Inbound Message1": {
      "main": [
        [
          {
            "node": "WA – AI Receptionist Response1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "WA – Groq LLM (Receptionist)1": {
      "ai_languageModel": [
        [
          {
            "node": "WA – AI Receptionist Response1",
            "type": "ai_languageModel",
            "index": 0
          }
        ]
      ]
    },
    "WA – AI Receptionist Response1": {
      "main": [
        [
          {
            "node": "WA – Send AI Reply to Client1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  },
  "active": false,
  "settings": {
    "executionOrder": "v1",
    "binaryMode": "separate",
    "availableInMCP": false
  },
  "versionId": "469c1af6-e8a4-4a22-9d00-e3340afc32fa",
  "meta": {
    "instanceId": "d7be0ad17c46a7b031f0f3750524d01445ec77dce72bafc60f2147272cc89d93"
  },
  "id": "pdSJIxH038m7LNop",
  "tags": []
}
