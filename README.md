# Agentforce Dynamic Welcome & Error Message Control

This repository demonstrates how to **dynamically control the welcome and error messages** displayed by an Agentforce Service Agent using:

- **AgentScript linked variables** pointing to custom `MessagingSession` fields
- A **Salesforce Routing Flow** that detects the user's locale and writes locale-specific messages to the `MessagingSession` record *before* the agent is invoked
- An **Embedded Messaging Channel** that passes the `locale` parameter from the web snippet into the flow

## How It Works

```
Website snippet  →  locale param  →  Routing Flow  →  MessagingSession.Greeting_Message__c
                                                   →  MessagingSession.Error_Message__c
                                                   →  Route to Agentforce Agent
                                                          ↓
                                              AgentScript reads fields via
                                              linked variables and displays
                                              as welcome / error messages
```

### Key Components

| Component | Purpose |
|---|---|
| `Route_to_Service_Agent` Flow | Reads the `locale` input, sets `Greeting_Message__c` and `Error_Message__c` on the `MessagingSession` record, then routes to the agent |
| `Language_Showcase_4` Agent (AgentScript) | Uses `{!@variables.Greeting_Message}` and `{!@variables.Error_Message}` — linked to the custom MessagingSession fields — as the system `welcome` and `error` messages |
| `Hotel_Booking_Channel` Messaging Channel | EmbeddedMessaging channel that passes `locale` as a custom parameter |
| `Hotel_Booking_Channel` EmbeddedServiceConfig | Web deployment config for the chat widget |
| `MessagingSession` custom fields | `Greeting_Message__c`, `Error_Message__c`, `locale__c` store per-session dynamic messages |

---

## Pre-requisites

- Salesforce org with **Agentforce / Einstein Bots** and **Embedded Messaging** enabled
- Salesforce CLI (`sf`) installed: https://developer.salesforce.com/tools/salesforcecli
- The **Language Showcase** Agentforce agent already published in your org (or deploy it from `aiAuthoringBundles`)

---

## Deployment Steps

### 1. Clone this repository

```bash
git clone https://github.com/SalesforceDiariesBySanket/agentforce-dynamic-welcome-error-message.git
cd agentforce-dynamic-welcome-error-message
```

### 2. Authenticate to your org

```bash
sf org login web --alias myOrg
```

### 3. Replace org-specific values

Before deploying, update the following placeholders with values from **your** org:

#### a) Flow — Queue ID
In `force-app/main/default/flows/Route_to_Service_Agent.flow-meta.xml`, find:
```xml
<stringValue>00GgL00000Eh8qHUAR</stringValue>
```
Replace `00GgL00000Eh8qHUAR` with **your** `Hotel_Booking_Queue` record ID.

To find it, run:
```bash
sf data query --query "SELECT Id, Name FROM Group WHERE Type='Queue' AND Name='Hotel Booking Queue'" --target-org myOrg
```

#### b) EmbeddedServiceConfig — Site name
In `force-app/main/default/EmbeddedServiceConfig/Hotel_Booking_Channel.EmbeddedServiceConfig-meta.xml`, find:
```xml
<site>ESW_Hotel_Booking_Channel_17801610904941</site>
```
Replace with **your** Experience Site or ESW Site API name. You can find it in **Setup → Digital Experiences → All Sites**.

#### c) AgentScript — Default Agent User
In `force-app/main/default/aiAuthoringBundles/Language_Showcase_4/Language_Showcase_4.agent`, find:
```yaml
default_agent_user: "case_assist_agent@00dgl00000s90vl1549306696.ext"
```
Replace with the username of your org's **bot / agent user**.

#### d) EmbeddedServiceConfig — Branding Set
In `Hotel_Booking_Channel.EmbeddedServiceConfig-meta.xml`, find:
```xml
<branding>EmbeddedServiceConfig_BrandingSet_19e79dede86</branding>
```
Replace with **your** branding set name, or remove this element to use the default branding.

### 4. Deploy the metadata

Deploy everything at once:

```bash
sf project deploy start \
  --source-dir force-app/main/default/objects/MessagingSession \
  --source-dir force-app/main/default/flows \
  --source-dir force-app/main/default/queues \
  --source-dir force-app/main/default/queueRoutingConfigs \
  --source-dir force-app/main/default/messagingChannels \
  --source-dir force-app/main/default/EmbeddedServiceConfig \
  --source-dir force-app/main/default/aiAuthoringBundles \
  --target-org myOrg
```

> **Note:** Deploy objects/fields first if you get dependency errors. The flow references the custom fields on `MessagingSession`.

### 5. Activate the Flow

If the flow deploys as inactive, activate it in **Setup → Flows → Route to Service Agent**.

### 6. Grab the Embedded Messaging snippet

1. Go to **Setup → Embedded Service → Hotel_Booking_Channel → Publish / Get Code**.
2. Copy the JavaScript snippet and paste it into your website's `<head>` or `<body>`.
3. When initializing the snippet, pass the locale:

```javascript
embeddedservice_bootstrap.settings.language = 'en_US'; // or es_MX, fr, de

// Pass locale as a pre-chat field (hidden)
embeddedservice_bootstrap.prechatAPI.setHiddenPrechatFields({
  locale: navigator.language || 'en_US'
});
```

---

## Supported Locales

| Locale code | Language |
|---|---|
| `en_US` | English |
| `es_MX` | Spanish (Mexico) |
| `fr` | French |
| `de` | German |

To add more locales, update the `CASE()` formulas in the Flow and the `if @variables.locale ==` blocks in the AgentScript.

---

## Folder Structure

```
force-app/main/default/
├── aiAuthoringBundles/
│   └── Language_Showcase_4/          # AgentScript agent definition
├── EmbeddedServiceConfig/
│   └── Hotel_Booking_Channel.*       # Chat widget deployment config
├── flows/
│   └── Route_to_Service_Agent.*      # Routing flow — sets greeting/error + routes
├── messagingChannels/
│   └── Hotel_Booking_Channel.*       # EmbeddedMessaging channel config
├── objects/
│   └── MessagingSession/
│       ├── fields/                   # Custom fields incl. Greeting_Message__c, Error_Message__c
│       └── listViews/
├── queueRoutingConfigs/
│   └── Hotel_Booking_Configuration.* # Queue routing config
└── queues/
    └── Hotel_Booking_Queue.*         # Omni-Channel queue
```

---

## Watch on YouTube

Follow along with the full video walkthrough on the [Salesforce Diaries](https://www.youtube.com/@salesforcediaries2871) YouTube channel.
