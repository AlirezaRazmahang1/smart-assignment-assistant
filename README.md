# smart-assignment-assistant
A Telegram bot that tracks Canvas LMS assignments, analyzes workload with Gemini AI, and sends automatic 24-hour deadline reminders.

# 📚 Smart Assignment Assistant

A personal AI-powered Telegram bot that connects to your Canvas LMS, 
analyzes your assignments with Google Gemini, and makes sure you never 
miss a deadline.{
  "name": "Smart Assignment Assistant – Canvas + Gemini + Telegram",
  "nodes": [
    {
      "parameters": {
        "updates": ["message"],
        "additionalFields": {}
      },
      "id": "telegram-trigger",
      "name": "📱 Telegram Message Trigger",
      "type": "n8n-nodes-base.telegramTrigger",
      "typeVersion": 1.1,
      "position": [180, 300],
      "webhookId": "YOUR_WEBHOOK_ID",
      "credentials": {
        "telegramApi": {
          "id": "TELEGRAM_CREDENTIAL_ID",
          "name": "Telegram Bot"
        }
      }
    },
    {
      "parameters": {
        "rule": {
          "interval": [
            {
              "field": "hours",
              "hoursInterval": 24
            }
          ]
        }
      },
      "id": "schedule-trigger",
      "name": "⏰ Daily 24h Check Trigger",
      "type": "n8n-nodes-base.scheduleTrigger",
      "typeVersion": 1.2,
      "position": [180, 500]
    },
    {
      "parameters": {
        "mode": "chooseBranch",
        "output": "input1"
      },
      "id": "merge-triggers",
      "name": "🔀 Merge Triggers",
      "type": "n8n-nodes-base.merge",
      "typeVersion": 3,
      "position": [420, 400]
    },
    {
      "parameters": {
        "mode": "runOnceForEachItem",
        "jsCode": "// Detect which trigger fired and set context\nconst item = $input.first().json;\n\nlet triggerType = 'schedule';\nlet userMessage = '';\nlet chatId = '';\n\n// If triggered by Telegram message\nif (item.message) {\n  triggerType = 'telegram';\n  userMessage = item.message.text || '';\n  chatId = String(item.message.chat.id);\n} else {\n  // Scheduled daily check – use your personal Chat ID\n  chatId = 'YOUR_TELEGRAM_CHAT_ID';\n}\n\nreturn {\n  triggerType,\n  userMessage: userMessage.toLowerCase().trim(),\n  chatId,\n  timestamp: new Date().toISOString()\n};"
      },
      "id": "detect-trigger",
      "name": "🔍 Detect Trigger Type",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [640, 400]
    },
    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": false,
            "leftValue": "",
            "typeValidation": "loose"
          },
          "conditions": [
            {
              "id": "cond-telegram",
              "leftValue": "={{ $json.triggerType }}",
              "rightValue": "telegram",
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
      "id": "route-trigger",
      "name": "🛤️ Route: Telegram or Schedule?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2,
      "position": [860, 400]
    },
    {
      "parameters": {
        "method": "GET",
        "url": "https://alexandercollege.instructure.com/api/v1/courses",
        "authentication": "genericCredentialType",
        "genericAuthType": "httpHeaderAuth",
        "sendHeaders": true,
        "headerParameters": {
          "parameters": [
            {
              "name": "Authorization",
              "value": "Bearer YOUR_CANVAS_API_TOKEN"
            }
          ]
        },
        "sendQuery": true,
        "queryParameters": {
          "parameters": [
            {
              "name": "enrollment_state",
              "value": "active"
            },
            {
              "name": "per_page",
              "value": "50"
            }
          ]
        },
        "options": {
          "response": {
            "response": {
              "responseFormat": "json"
            }
          }
        }
      },
      "id": "fetch-courses",
      "name": "📚 Fetch Active Canvas Courses",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [1100, 300]
    },
    {
      "parameters": {
        "method": "GET",
        "url": "https://alexandercollege.instructure.com/api/v1/users/self/todo",
        "authentication": "genericCredentialType",
        "genericAuthType": "httpHeaderAuth",
        "sendHeaders": true,
        "headerParameters": {
          "parameters": [
            {
              "name": "Authorization",
              "value": "Bearer YOUR_CANVAS_API_TOKEN"
            }
          ]
        },
        "sendQuery": true,
        "queryParameters": {
          "parameters": [
            {
              "name": "per_page",
              "value": "50"
            }
          ]
        },
        "options": {
          "response": {
            "response": {
              "responseFormat": "json"
            }
          }
        }
      },
      "id": "fetch-assignments",
      "name": "📋 Fetch All Upcoming Assignments",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [1100, 500]
    },
    {
      "parameters": {
        "mode": "combine",
        "combinationMode": "mergeByPosition",
        "options": {}
      },
      "id": "combine-canvas-data",
      "name": "🗂️ Combine Canvas Data",
      "type": "n8n-nodes-base.merge",
      "typeVersion": 3,
      "position": [1340, 400]
    },
    {
      "parameters": {
        "mode": "runOnceForAllItems",
        "jsCode": "// Parse and clean assignment data from Canvas API\nconst items = $input.all();\nconst now = new Date();\n\nconst assignments = [];\n\nfor (const item of items) {\n  const a = item.json;\n  \n  // Handle both /todo and direct assignment endpoints\n  const assignment = a.assignment || a;\n  \n  if (!assignment.name || !assignment.due_at) continue;\n  \n  const dueDate = new Date(assignment.due_at);\n  const diffMs = dueDate - now;\n  const diffHours = diffMs / (1000 * 60 * 60);\n  const diffDays = diffHours / 24;\n  \n  if (diffDays < 0) continue; // Skip past due\n  \n  assignments.push({\n    id: assignment.id,\n    name: assignment.name,\n    course: assignment.course_id || 'Unknown Course',\n    dueDate: assignment.due_at,\n    dueDateFormatted: dueDate.toLocaleDateString('en-CA', {\n      weekday: 'short',\n      year: 'numeric',\n      month: 'short',\n      day: 'numeric',\n      hour: '2-digit',\n      minute: '2-digit'\n    }),\n    daysUntilDue: Math.round(diffDays * 10) / 10,\n    hoursUntilDue: Math.round(diffHours),\n    description: (assignment.description || '').replace(/<[^>]+>/g, '').slice(0, 500),\n    pointsPossible: assignment.points_possible || 'N/A',\n    submissionTypes: (assignment.submission_types || []).join(', '),\n    isDueTomorrow: diffHours >= 20 && diffHours <= 28\n  });\n}\n\n// Sort by due date ascending\nassignments.sort((a, b) => new Date(a.dueDate) - new Date(b.dueDate));\n\nreturn assignments.map(a => ({ json: a }));"
      },
      "id": "parse-assignments",
      "name": "🧹 Parse & Filter Assignments",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [1560, 400]
    },
    {
      "parameters": {
        "mode": "runOnceForAllItems",
        "jsCode": "// Collect all assignments into one batch for Gemini\nconst assignments = $input.all().map(i => i.json);\n\nconst assignmentList = assignments.map((a, idx) => \n  `${idx + 1}. Name: \"${a.name}\"\n   Course ID: ${a.course}\n   Due: ${a.dueDateFormatted} (in ${a.daysUntilDue} days)\n   Points: ${a.pointsPossible}\n   Type: ${a.submissionTypes || 'Not specified'}\n   Description: ${a.description || 'No description available'}`\n).join('\\n\\n');\n\nreturn [{\n  json: {\n    assignments,\n    assignmentListText: assignmentList,\n    totalCount: assignments.length\n  }\n}];"
      },
      "id": "prepare-gemini-prompt",
      "name": "📝 Prepare Gemini Prompt",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [1780, 400]
    },
    {
      "parameters": {
        "method": "POST",
        "url": "https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-flash:generateContent",
        "sendQuery": true,
        "queryParameters": {
          "parameters": [
            {
              "name": "key",
              "value": "YOUR_GOOGLE_GEMINI_API_KEY"
            }
          ]
        },
        "sendBody": true,
        "contentType": "json",
        "body": {
          "contents": [
            {
              "parts": [
                {
                  "text": "You are an academic assistant helping a college student manage their workload. Analyze these Canvas assignments and for EACH one provide:\n1. A priority level: HIGH (due ≤3 days), MEDIUM (4-7 days), LOW (8+ days)\n2. Estimated hours to complete (be realistic: essays 3-8h, coding 4-10h, readings 1-3h, quizzes 1-2h)\n3. A one-line study tip\n\nFormat your response as a JSON array ONLY (no markdown, no explanation), like this:\n[\n  {\n    \"id\": <assignment_id_number>,\n    \"name\": \"Assignment name\",\n    \"priority\": \"HIGH|MEDIUM|LOW\",\n    \"priorityEmoji\": \"🔴|🟡|🟢\",\n    \"estimatedHours\": <number>,\n    \"studyTip\": \"One actionable tip\"\n  }\n]\n\nAssignments to analyze:\n={{ $json.assignmentListText }}"
                }
              ]
            }
          ],
          "generationConfig": {
            "temperature": 0.3,
            "maxOutputTokens": 2048
          }
        },
        "options": {
          "response": {
            "response": {
              "responseFormat": "json"
            }
          }
        }
      },
      "id": "gemini-analyze",
      "name": "🤖 Gemini AI Analysis",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [2000, 400]
    },
    {
      "parameters": {
        "mode": "runOnceForAllItems",
        "jsCode": "// Merge Gemini analysis with original assignment data\nconst geminiResponse = $input.first().json;\n\nlet aiResults = [];\ntry {\n  const rawText = geminiResponse.candidates[0].content.parts[0].text;\n  // Strip any markdown code fences if present\n  const cleaned = rawText.replace(/```json|```/g, '').trim();\n  aiResults = JSON.parse(cleaned);\n} catch(e) {\n  // Fallback if parsing fails\n  aiResults = [];\n}\n\n// Get original assignments from previous node context\n// We'll re-fetch from the gemini prompt node's input\nconst assignments = $('Prepare Gemini Prompt').first().json.assignments;\n\n// Merge AI results with assignment data\nconst enriched = assignments.map(a => {\n  const aiData = aiResults.find(r => r.id == a.id || r.name === a.name) || {\n    priority: a.daysUntilDue <= 3 ? 'HIGH' : a.daysUntilDue <= 7 ? 'MEDIUM' : 'LOW',\n    priorityEmoji: a.daysUntilDue <= 3 ? '🔴' : a.daysUntilDue <= 7 ? '🟡' : '🟢',\n    estimatedHours: 3,\n    studyTip: 'Break this into smaller chunks and start early.'\n  };\n  return { ...a, ...aiData };\n});\n\n// Find assignments due in exactly ~24 hours\nconst dueTomorrow = enriched.filter(a => a.isDueTomorrow);\n\n// Get trigger context from earlier node\nconst triggerContext = $('Detect Trigger Type').first().json;\n\nreturn [{\n  json: {\n    enrichedAssignments: enriched,\n    dueTomorrow,\n    triggerType: triggerContext.triggerType,\n    chatId: triggerContext.chatId,\n    userMessage: triggerContext.userMessage,\n    totalCount: enriched.length\n  }\n}];"
      },
      "id": "merge-ai-results",
      "name": "🔗 Merge AI + Assignment Data",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [2220, 400]
    },
    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": false,
            "leftValue": "",
            "typeValidation": "loose"
          },
          "conditions": [
            {
              "id": "cond-schedule",
              "leftValue": "={{ $json.triggerType }}",
              "rightValue": "schedule",
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
      "id": "route-output",
      "name": "🛤️ Route Output: Schedule or On-Demand?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2,
      "position": [2440, 400]
    },
    {
      "parameters": {
        "mode": "runOnceForEachItem",
        "jsCode": "// Build the 24-hour urgent alert message\nconst data = $input.first().json;\nconst dueTomorrow = data.dueTomorrow || [];\n\nif (dueTomorrow.length === 0) {\n  // Nothing due tomorrow, skip silently\n  return [{ json: { skip: true, chatId: data.chatId } }];\n}\n\nconst lines = dueTomorrow.map(a => \n  `⚠️ *${a.name}*\\n` +\n  `📅 Due: ${a.dueDateFormatted}\\n` +\n  `⏱ Est. time: ~${a.estimatedHours}h\\n` +\n  `💡 ${a.studyTip}`\n).join('\\n\\n─────────────────\\n\\n');\n\nconst message = \n  `🚨 *ASSIGNMENT DUE TOMORROW* 🚨\\n\\n` +\n  lines +\n  `\\n\\n_Reply \"assignments\" anytime for your full schedule._`;\n\nreturn [{ json: { message, chatId: data.chatId, skip: false } }];"
      },
      "id": "build-24h-alert",
      "name": "🚨 Build 24h Urgent Alert",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [2660, 280]
    },
    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": false,
            "typeValidation": "loose"
          },
          "conditions": [
            {
              "id": "skip-check",
              "leftValue": "={{ $json.skip }}",
              "rightValue": true,
              "operator": {
                "type": "boolean",
                "operation": "false"
              }
            }
          ],
          "combinator": "and"
        },
        "options": {}
      },
      "id": "check-skip",
      "name": "❓ Any due tomorrow?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2,
      "position": [2880, 280]
    },
    {
      "parameters": {
        "mode": "runOnceForEachItem",
        "jsCode": "// Build the on-demand full assignment breakdown\nconst data = $input.first().json;\nconst assignments = data.enrichedAssignments || [];\nconst userMsg = data.userMessage || '';\n\nif (assignments.length === 0) {\n  return [{\n    json: {\n      message: '🎉 Great news! You have no upcoming assignments on Canvas right now. Enjoy the break!',\n      chatId: data.chatId\n    }\n  }];\n}\n\n// Build summary lines\nconst lines = assignments.map((a, i) => {\n  const dueLabel = a.daysUntilDue <= 1 \n    ? `⚡ ${a.hoursUntilDue}h left!` \n    : `📅 ${a.daysUntilDue}d away`;\n  \n  return (\n    `${a.priorityEmoji} *${a.name}*\\n` +\n    `${dueLabel} · ${a.dueDateFormatted}\\n` +\n    `⏱ ~${a.estimatedHours}h · Priority: *${a.priority}*\\n` +\n    `💡 ${a.studyTip}`\n  );\n}).join('\\n\\n─────────────────\\n\\n');\n\n// Header summary\nconst highCount = assignments.filter(a => a.priority === 'HIGH').length;\nconst header = \n  `📚 *Your Assignment Dashboard*\\n` +\n  `━━━━━━━━━━━━━━━━━━━━━━\\n` +\n  `📊 Total: ${assignments.length} upcoming` +\n  (highCount > 0 ? ` · 🔴 ${highCount} urgent` : '') +\n  `\\n\\n`;\n\nconst message = header + lines + `\\n\\n_Last updated: ${new Date().toLocaleTimeString('en-CA')}_`;\n\nreturn [{ json: { message, chatId: data.chatId } }];"
      },
      "id": "build-demand-message",
      "name": "📊 Build On-Demand Dashboard",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [2660, 520]
    },
    {
      "parameters": {
        "chatId": "={{ $json.chatId }}",
        "text": "={{ $json.message }}",
        "additionalFields": {
          "parse_mode": "Markdown",
          "disable_web_page_preview": true
        }
      },
      "id": "send-24h-telegram",
      "name": "📲 Send 24h Alert via Telegram",
      "type": "n8n-nodes-base.telegram",
      "typeVersion": 1.2,
      "position": [3100, 280],
      "credentials": {
        "telegramApi": {
          "id": "TELEGRAM_CREDENTIAL_ID",
          "name": "Telegram Bot"
        }
      }
    },
    {
      "parameters": {
        "chatId": "={{ $json.chatId }}",
        "text": "={{ $json.message }}",
        "additionalFields": {
          "parse_mode": "Markdown",
          "disable_web_page_preview": true
        }
      },
      "id": "send-demand-telegram",
      "name": "📲 Send Dashboard via Telegram",
      "type": "n8n-nodes-base.telegram",
      "typeVersion": 1.2,
      "position": [2880, 520],
      "credentials": {
        "telegramApi": {
          "id": "TELEGRAM_CREDENTIAL_ID",
          "name": "Telegram Bot"
        }
      }
    },
    {
      "parameters": {
        "mode": "runOnceForEachItem",
        "jsCode": "// Send a typing acknowledgment for instant response feel\nconst item = $input.first().json;\nreturn [{ json: { chatId: String(item.message?.chat?.id || 'YOUR_TELEGRAM_CHAT_ID') } }];"
      },
      "id": "ack-message",
      "name": "⏳ Quick Ack (Fetching...)",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [1100, 160]
    },
    {
      "parameters": {
        "chatId": "={{ $json.chatId }}",
        "text": "🔄 Fetching your Canvas assignments... Please wait a moment!",
        "additionalFields": {
          "parse_mode": "Markdown"
        }
      },
      "id": "send-ack-telegram",
      "name": "📲 Send Ack Message",
      "type": "n8n-nodes-base.telegram",
      "typeVersion": 1.2,
      "position": [1340, 160],
      "credentials": {
        "telegramApi": {
          "id": "TELEGRAM_CREDENTIAL_ID",
          "name": "Telegram Bot"
        }
      }
    }
  ],
  "connections": {
    "📱 Telegram Message Trigger": {
      "main": [
        [
          { "node": "🔀 Merge Triggers", "type": "main", "index": 0 }
        ]
      ]
    },
    "⏰ Daily 24h Check Trigger": {
      "main": [
        [
          { "node": "🔀 Merge Triggers", "type": "main", "index": 1 }
        ]
      ]
    },
    "🔀 Merge Triggers": {
      "main": [
        [
          { "node": "🔍 Detect Trigger Type", "type": "main", "index": 0 }
        ]
      ]
    },
    "🔍 Detect Trigger Type": {
      "main": [
        [
          { "node": "🛤️ Route: Telegram or Schedule?", "type": "main", "index": 0 }
        ]
      ]
    },
    "🛤️ Route: Telegram or Schedule?": {
      "main": [
        [
          { "node": "⏳ Quick Ack (Fetching...)", "type": "main", "index": 0 },
          { "node": "📚 Fetch Active Canvas Courses", "type": "main", "index": 0 },
          { "node": "📋 Fetch All Upcoming Assignments", "type": "main", "index": 0 }
        ],
        [
          { "node": "📚 Fetch Active Canvas Courses", "type": "main", "index": 0 },
          { "node": "📋 Fetch All Upcoming Assignments", "type": "main", "index": 0 }
        ]
      ]
    },
    "⏳ Quick Ack (Fetching...)": {
      "main": [
        [
          { "node": "📲 Send Ack Message", "type": "main", "index": 0 }
        ]
      ]
    },
    "📚 Fetch Active Canvas Courses": {
      "main": [
        [
          { "node": "🗂️ Combine Canvas Data", "type": "main", "index": 0 }
        ]
      ]
    },
    "📋 Fetch All Upcoming Assignments": {
      "main": [
        [
          { "node": "🗂️ Combine Canvas Data", "type": "main", "index": 1 }
        ]
      ]
    },
    "🗂️ Combine Canvas Data": {
      "main": [
        [
          { "node": "🧹 Parse & Filter Assignments", "type": "main", "index": 0 }
        ]
      ]
    },
    "🧹 Parse & Filter Assignments": {
      "main": [
        [
          { "node": "📝 Prepare Gemini Prompt", "type": "main", "index": 0 }
        ]
      ]
    },
    "📝 Prepare Gemini Prompt": {
      "main": [
        [
          { "node": "🤖 Gemini AI Analysis", "type": "main", "index": 0 }
        ]
      ]
    },
    "🤖 Gemini AI Analysis": {
      "main": [
        [
          { "node": "🔗 Merge AI + Assignment Data", "type": "main", "index": 0 }
        ]
      ]
    },
    "🔗 Merge AI + Assignment Data": {
      "main": [
        [
          { "node": "🛤️ Route Output: Schedule or On-Demand?", "type": "main", "index": 0 }
        ]
      ]
    },
    "🛤️ Route Output: Schedule or On-Demand?": {
      "main": [
        [
          { "node": "🚨 Build 24h Urgent Alert", "type": "main", "index": 0 }
        ],
        [
          { "node": "📊 Build On-Demand Dashboard", "type": "main", "index": 0 }
        ]
      ]
    },
    "🚨 Build 24h Urgent Alert": {
      "main": [
        [
          { "node": "❓ Any due tomorrow?", "type": "main", "index": 0 }
        ]
      ]
    },
    "❓ Any due tomorrow?": {
      "main": [
        [
          { "node": "📲 Send 24h Alert via Telegram", "type": "main", "index": 0 }
        ]
      ]
    },
    "📊 Build On-Demand Dashboard": {
      "main": [
        [
          { "node": "📲 Send Dashboard via Telegram", "type": "main", "index": 0 }
        ]
      ]
    }
  },
  "pinData": {},
  "settings": {
    "executionOrder": "v1",
    "saveManualExecutions": true,
    "callerPolicy": "workflowsFromSameOwner",
    "errorWorkflow": ""
  },
  "staticData": null,
  "tags": [
    { "name": "Canvas", "createdAt": "2024-01-01" },
    { "name": "Telegram", "createdAt": "2024-01-01" },
    { "name": "Gemini", "createdAt": "2024-01-01" },
    { "name": "Assignment Tracker", "createdAt": "2024-01-01" }
  ],
  "triggerCount": 2,
  "updatedAt": "2026-05-15T00:00:00.000Z",
  "versionId": "1"
}


## ✨ Features

- **On-demand dashboard** — text the bot anytime to get a full 
  breakdown of all upcoming assignments
- **Automatic 24h alerts** — the bot silently checks every day and 
  fires an urgent warning when a deadline is tomorrow
- **AI analysis** — Gemini AI reads each assignment and estimates 
  how long it will take, ranks it by priority (🔴 High / 🟡 Medium / 🟢 Low), 
  and gives you a personalized study tip
- **Canvas integration** — pulls live data directly from your 
  institution's Canvas portal using the official API

## 🛠️ Built With

- **n8n** — workflow automation (self-hosted or cloud)
- **Canvas LMS API** — assignment and course data
- **Google Gemini 1.5 Flash** — AI analysis and time estimation
- **Telegram Bot API** — message delivery

## ⚙️ Setup

1. Import `smart_assignment_assistant.json` into your n8n instance
2. Add your credentials:
   - `CANVAS_API_TOKEN` — from Canvas → Account → Settings → New Access Token
   - `GEMINI_API_KEY` — from [aistudio.google.com](https://aistudio.google.com)
   - `TELEGRAM_BOT_TOKEN` — from [@BotFather](https://t.me/BotFather) on Telegram
   - `TELEGRAM_CHAT_ID` — your personal chat ID
3. Set your n8n instance URL (use [ngrok](https://ngrok.com) for local hosting)
4. Activate the workflow and message your bot

## 📁 Files

| File | Description |
|------|-------------|
| `smart_assignment_assistant.json` | The complete n8n workflow |
| `README.md` | This file |

## 💬 Example Bot Output[smart_assignment_assistant.json](https://github.com/user-attachments/files/27840753/smart_assignment_assistant.json)
