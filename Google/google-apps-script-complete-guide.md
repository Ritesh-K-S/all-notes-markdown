# Google Apps Script — Complete Guide (From Zero to Add-on)

> Everything about Google Apps Script: what it is, how it runs, who it runs as, how to create add-ons, how the consent screen works, and the full flow — explained in extreme detail with diagrams.

---

## Table of Contents

1. [What is Google Apps Script?](#1-what-is-google-apps-script)
2. [Where Does Script Code Run?](#2-where-does-script-code-run)
3. [Two Types of Scripts](#3-two-types-of-scripts)
4. [Google Services Available in Apps Script](#4-google-services-available-in-apps-script)
5. [File Access & Creation — Where Do Files Come From?](#5-file-access--creation--where-do-files-come-from)
6. [Execute as Owner vs Execute as User — Full Explanation with Steps](#6-execute-as-owner-vs-execute-as-user--full-explanation-with-steps)
7. [Deployment Options — Web App, Add-on, Chat App](#7-deployment-options--web-app-add-on-chat-app)
8. [Test Deployment vs New Deployment — Complete Guide](#8-test-deployment-vs-new-deployment--complete-guide)
9. [How to Create a Google Workspace Add-on — Step by Step with Code Explanation](#9-how-to-create-a-google-workspace-add-on--step-by-step-with-code-explanation)
10. [Google Chat App — Step by Step](#10-google-chat-app--step-by-step)
11. [Extending a Chat App to Other Products — Gmail Add-on from Same Project](#11-extending-a-chat-app-to-other-products--gmail-add-on-from-same-project)
12. [The Consent Screen — Complete Deep Dive](#12-the-consent-screen--complete-deep-dive)
13. [GCP Project — Auto-Created vs Manual & Sharing Scripts in a Company](#13-gcp-project--auto-created-vs-manual--sharing-scripts-in-a-company)
14. [Common Company Pattern — Shared Sheet as Database](#14-common-company-pattern--shared-sheet-as-database)
15. [Ownership & Best Practices — What Happens When the Developer Leaves](#15-ownership--best-practices--what-happens-when-the-developer-leaves)
16. [Local Development with clasp](#16-local-development-with-clasp)
17. [Quick Decision Guide](#17-quick-decision-guide)

---

## 1. What is Google Apps Script?

Google Apps Script is a **JavaScript-based cloud scripting platform** created by Google.

- You write **JavaScript** code
- It runs on **Google's servers** (not on your computer)
- It can interact with **all Google Workspace products**: Gmail, Sheets, Docs, Drive, Calendar, Chat, Slides, Forms, etc.
- No hosting, no server setup, no deployment headaches — Google handles everything

Think of it as: **"JavaScript that lives inside Google and can control all Google products."**

```
┌─────────────────────────────────────────────────────────────┐
│                   Google Apps Script                         │
│                                                             │
│   You write JavaScript  ──→  Runs on Google's Cloud         │
│                              (not your laptop)              │
│                                                             │
│   Can control:                                              │
│   ┌─────────┐ ┌──────┐ ┌──────┐ ┌───────┐ ┌──────────┐    │
│   │ Gmail   │ │Sheets│ │ Docs │ │ Drive │ │ Calendar │    │
│   └─────────┘ └──────┘ └──────┘ └───────┘ └──────────┘    │
│   ┌─────────┐ ┌──────┐ ┌──────┐ ┌───────┐                  │
│   │ Slides  │ │Forms │ │ Chat │ │ Maps  │                  │
│   └─────────┘ └──────┘ └──────┘ └───────┘                  │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Where Does Script Code Run?

This is a very common confusion. Let's clear it:

```
   ❌ Script does NOT run here:
   ┌──────────────────────────┐
   │  Your laptop / browser   │
   │  (only the editor is     │
   │   in the browser)        │
   └──────────────────────────┘

   ✅ Script RUNS here:
   ┌──────────────────────────┐
   │  Google's Cloud Servers  │
   │  (V8 JavaScript engine)  │
   │  (server-side execution) │
   └──────────────────────────┘
```

**Key point:** When you click "Run" in the Apps Script editor, your browser sends the request to Google's servers. Google's servers execute the JavaScript and return the result to your browser.

---

## 3. Two Types of Scripts

### 3.1 Container-Bound Script

The script lives **inside** a Google file (Sheet, Doc, Form, Slide).

**How to create:**
1. Open any Google Sheet / Doc / Form
2. Go to `Extensions → Apps Script`
3. The script editor opens — this script is **bound** to that file

```
┌───────────────────────────────────┐
│   Google Sheet: "Sales Report"    │
│                                   │
│   ┌───────────────────────────┐   │
│   │  Apps Script (inside)     │   │
│   │  - Can use getActive()    │   │
│   │  - Tied to this sheet     │   │
│   └───────────────────────────┘   │
└───────────────────────────────────┘
```

**Special ability:** Can use `SpreadsheetApp.getActiveSpreadsheet()`, `DocumentApp.getActiveDocument()` — because it knows which file it belongs to.

### 3.2 Standalone Script

The script is its **own independent file** in Google Drive.

**How to create:**
1. Go to [script.google.com](https://script.google.com)
2. Click `+ New Project`
3. This creates a standalone `.gs` file in your Drive

```
┌───────────────────────────────────┐
│   Google Drive                    │
│                                   │
│   📄 Sales Report.gsheet          │
│   📄 My Document.gdoc             │
│   📜 My Script.gs  ← standalone  │
│      (not inside any file)        │
└───────────────────────────────────┘
```

**Cannot use** `getActiveSpreadsheet()` because it doesn't belong to any file. Must use `openById()` or `openByUrl()` to access files.

### Comparison Table

| Feature | Container-Bound | Standalone |
|---|---|---|
| Lives inside a file? | Yes | No |
| Can use `getActive*()` | Yes | No |
| Accessible from script.google.com | Yes (listed there) | Yes |
| Can be deployed as Web App | Yes | Yes |
| Can be deployed as Add-on | Yes | Yes |
| Best for | Automating a specific sheet/doc | General tools, add-ons |

---

## 4. Google Services Available in Apps Script

You can use these built-in services directly — no imports needed:

```javascript
// Spreadsheet Service
SpreadsheetApp.create("New Sheet");                    // Create a new spreadsheet
SpreadsheetApp.openById("sheet_id_here");              // Open existing sheet by ID
SpreadsheetApp.getActiveSpreadsheet();                 // Get the sheet this script is bound to

// Document Service
DocumentApp.create("New Doc");                         // Create a new document
DocumentApp.openById("doc_id_here");                   // Open existing doc by ID

// Drive Service
DriveApp.createFile("test.txt", "Hello");              // Create a file in Drive
DriveApp.getFileById("file_id");                       // Get any file by ID
DriveApp.getFolderById("folder_id");                   // Get a folder

// Gmail Service
GmailApp.sendEmail("to@email.com", "Subject", "Body"); // Send email
GmailApp.getInboxThreads();                            // Read inbox

// Calendar Service
CalendarApp.getDefaultCalendar();                      // Access calendar
CalendarApp.createEvent("Meeting", startDate, endDate); // Create event

// Slides Service
SlidesApp.create("New Presentation");                  // Create slides

// Forms Service
FormApp.create("New Form");                            // Create a form
```

---

## 5. File Access & Creation — Where Do Files Come From?

### 5.1 When You CREATE a File

```javascript
var newSheet = SpreadsheetApp.create("My Report");
// This creates "My Report" in the ROOT of the running user's Google Drive
```

```
Google Drive (of whoever runs the script)
│
├── 📁 My Drive   ← ROOT (file lands here by default)
│   ├── 📄 My Report  ← NEWLY CREATED HERE
│   ├── 📁 Work Stuff
│   ├── 📁 Personal
│   └── ...
```

**To create in a specific folder:**

```javascript
// Step 1: Create the file (goes to root)
var newSheet = SpreadsheetApp.create("My Report");

// Step 2: Get the file object from Drive
var file = DriveApp.getFileById(newSheet.getId());

// Step 3: Get the target folder
var folder = DriveApp.getFolderById("target_folder_id_here");

// Step 4: Move it — add to folder, remove from root
folder.addFile(file);
DriveApp.getRootFolder().removeFile(file);
```

### 5.2 When You ACCESS (Open) a File

```javascript
// By ID — the file can be ANYWHERE in Drive
var sheet = SpreadsheetApp.openById("1BxiMVs0XRA5nFMdKvBd...");

// By URL
var sheet = SpreadsheetApp.openByUrl("https://docs.google.com/spreadsheets/d/1Bxi.../edit");
```

**The user running the script MUST have access** to that file. If they don't, the script throws an error.

### 5.3 The Big Question — Whose Drive?

It depends on the **"Execute as"** setting. This is covered in detail in the next section.

```
┌──────────────────────────────────────────────────────────────────┐
│                    Where do files come from?                     │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  SpreadsheetApp.create("Report")                                │
│  ├── Execute as Owner  → Owner's Drive (root folder)            │
│  └── Execute as User   → That user's Drive (root folder)        │
│                                                                  │
│  SpreadsheetApp.openById("abc123")                              │
│  ├── Execute as Owner  → Owner must have access to that file    │
│  └── Execute as User   → That user must have access             │
│                                                                  │
│  DriveApp.getFilesByName("data.csv")                            │
│  ├── Execute as Owner  → Searches Owner's Drive                 │
│  └── Execute as User   → Searches that user's Drive             │
│                                                                  │
│  GmailApp.getInboxThreads()                                     │
│  ├── Execute as Owner  → Owner's Gmail inbox                    │
│  └── Execute as User   → That user's Gmail inbox                │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 6. Execute as Owner vs Execute as User — Full Explanation with Steps

This is the **most important concept** when sharing scripts in a company.

### 6.1 What Does "Execute as" Mean?

When you deploy a script as a **Web App**, Google asks you:

> "When someone else runs this script, whose Google account should the script use to access Drive, Gmail, Sheets, etc.?"

There are exactly **two options**:

### 6.2 Option 1: Execute as "Me" (the Owner)

**What it means:** No matter who opens the web app, the script always uses the **owner's** Google account to access resources.

```
┌─────────────────────────────────────────────────────────────┐
│                  Execute as: "Me" (Owner)                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Employee A runs script  ──┐                                │
│  Employee B runs script  ──┤──→  Script uses OWNER's        │
│  Employee C runs script  ──┘     Google Account             │
│                                                             │
│  • Creates files in OWNER's Drive                           │
│  • Reads OWNER's Gmail                                      │
│  • Accesses OWNER's Calendar                                │
│  • Users DON'T need permission to underlying resources      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Step-by-step how to set this up:**

1. Go to [script.google.com](https://script.google.com) → Open your project
2. Write your code (e.g., a script that reads a Sheet)
3. Click **Deploy** → **New Deployment**
4. Select type: **Web App**
5. Set these options:
   - **Description:** "My Company Tool v1"
   - **Execute as:** → Select **"Me"** (your email shown here)
   - **Who has access:** → Select **"Anyone"** or **"Anyone within [your org]"**
6. Click **Deploy**
7. Copy the URL and share with users

**Use case:** You have a master spreadsheet in your Drive. You want employees to view/interact with it through a web app, but you don't want to share the spreadsheet directly with them.

### 6.3 Option 2: Execute as "User accessing the web app"

**What it means:** The script runs using the **Google account of whoever opens the URL**. Each user accesses their own resources.

```
┌─────────────────────────────────────────────────────────────┐
│             Execute as: "User accessing the web app"        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Employee A runs script  → Script uses Employee A's account │
│  Employee B runs script  → Script uses Employee B's account │
│  Employee C runs script  → Script uses Employee C's account │
│                                                             │
│  • Creates files in EACH USER's own Drive                   │
│  • Reads EACH USER's own Gmail                              │
│  • Each user must have permission to access resources       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Step-by-step how to set this up:**

1. Go to [script.google.com](https://script.google.com) → Open your project
2. Write your code
3. Click **Deploy** → **New Deployment**
4. Select type: **Web App**
5. Set these options:
   - **Description:** "Personal Drive Organizer v1"
   - **Execute as:** → Select **"User accessing the web app"**
   - **Who has access:** → Select **"Anyone"** or **"Anyone within [your org]"**
6. Click **Deploy**
7. Share the URL

**Use case:** A tool that helps each employee organize their own Google Drive. Each person should only see their own files.

### 6.4 Side-by-Side Comparison

```
           Execute as: "Me" (Owner)         Execute as: "User"
           ─────────────────────────         ──────────────────

   User A ───→ ┌──────────────────┐    User A ───→ ┌──────────────────┐
               │  Script runs     │                │  Script runs     │
   User B ───→ │  as OWNER        │    User B ───→ │  as USER B       │
               │                  │                │                  │
   User C ───→ │  Accesses:       │    User C ───→ │  Accesses:       │
               │  Owner's Drive   │                │  Each user's     │
               │  Owner's Gmail   │                │  own Drive/Gmail │
               │  Owner's Cal     │                │                  │
               └──────────────────┘                └──────────────────┘

   Files created → Owner's Drive       Files created → Each user's Drive
   Files read    → Owner's files       Files read    → Each user's files
   Permissions   → User doesn't need   Permissions   → User MUST have
                   access to files                     access to files
```

### 6.5 Important Note for Add-ons

Google Workspace Add-ons (Gmail, Sheets, Docs add-ons) **always run as the user who installed them**. You do NOT get the "Execute as" choice. This is for security — an add-on reading your Gmail should only read YOUR Gmail, not the developer's.

```
┌─────────────────────────────────────────────────────┐
│              Add-ons — No Choice                    │
│                                                     │
│  Add-ons ALWAYS run as the installing user.         │
│  There is no "Execute as Owner" option.             │
│                                                     │
│  Employee installs Gmail add-on                     │
│  → Add-on reads THAT employee's Gmail               │
│  → Add-on accesses THAT employee's Drive            │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

## 7. Deployment Options — Web App, Add-on, Chat App

```
                    ┌─────────────────────────────────────┐
                    │     How to share your script?        │
                    └──────────────┬──────────────────────┘
                                   │
              ┌────────────────────┼────────────────────────┐
              │                    │                         │
              ▼                    ▼                         ▼
      ┌──────────────┐   ┌────────────────┐    ┌──────────────────┐
      │   Web App     │   │  Add-on        │    │  Chat App        │
      │               │   │                │    │                  │
      │ Users access  │   │ Users install  │    │ Bot in Google    │
      │ via a URL     │   │ inside Gmail/  │    │ Chat spaces      │
      │               │   │ Sheets/Docs    │    │                  │
      │ Has its own   │   │ Shows as       │    │ Responds to      │
      │ HTML page     │   │ sidebar/menu   │    │ messages/cmds    │
      └──────────────┘   └────────────────┘    └──────────────────┘
```

### 7.1 Web App

- Script must have a `doGet(e)` function (for GET requests) or `doPost(e)` (for POST requests)
- Returns HTML to the browser
- Users access via URL
- You choose "Execute as" (Owner or User)

### 7.2 Add-on (Google Workspace Add-on)

- Script uses **Card Service** to build UI
- Users install it inside Google products (Gmail, Sheets, Docs, etc.)
- Always runs as the user
- Needs proper manifest configuration

### 7.3 Chat App

- Script responds to messages in Google Chat
- Uses `onMessage()`, `onAddToSpace()` functions
- Needs Google Cloud Project + Chat API enabled
- Can respond with text or cards

### 7.4 These Are Completely Different Deployment Types

> **⚠️ Critical Concept:** Web App, Add-on, and Chat App are NOT variations of the same thing. They are **completely different deployment types** with different APIs, different UIs, and different distribution methods.

| Feature | Web App | Workspace Add-on | Chat App |
|---|---|---|---|
| **Where it lives** | Standalone URL in browser | Sidebar inside Gmail/Sheets/Docs | Inside Google Chat |
| **UI Framework** | `HtmlService` (HTML/CSS/JS) | `CardService` (pre-built cards) | JSON responses / `CardService` cards |
| **Entry functions** | `doGet(e)` / `doPost(e)` | `onHomepage(e)` / `onGmailMessage(e)` | `onMessage(e)` / `onAddToSpace(e)` |
| **GCP Project needed?** | No (auto-created) | No for testing; Yes for Marketplace | **Yes — always** (must enable Chat API) |
| **"Execute as" choice?** | Yes (Owner or User) | No (always runs as user) | No (always runs as user) |
| **Distribution** | Share the URL | Admin install or Marketplace | Add users in GCP Chat API config |

> **Can one project have multiple types?** YES! A single Apps Script project can be a Chat App AND an Add-on simultaneously. The manifest (`appsscript.json`) can contain both `chat: {}` and `addOns: {}` blocks. They share the same GCP project but have independent distribution channels.

---

## 8. Test Deployment vs New Deployment — Complete Guide

This is a frequently confused topic. When you click **Deploy** in the Apps Script editor, you see two options. They serve very different purposes.

### 8.1 Test Deployment

```
┌─────────────────────────────────────────────────────┐
│              TEST DEPLOYMENT                        │
│                                                     │
│  • Always runs the LATEST saved code (HEAD)         │
│  • No frozen version — changes reflect immediately  │
│  • For internal testing / company-internal use       │
│  • No Google review needed                          │
│  • Gets a Deployment ID (used for admin install)    │
│  • Can be installed by Google Workspace Admin       │
│    for all employees via Admin Console              │
│                                                     │
│  How to create:                                     │
│  Deploy → Test deployments → Install                │
└─────────────────────────────────────────────────────┘
```

**Key behavior:** When you update your code and save, ALL users running the test deployment automatically get the new version. No redeployment needed.

### 8.2 New Deployment (Versioned)

```
┌─────────────────────────────────────────────────────┐
│              NEW DEPLOYMENT                         │
│                                                     │
│  • Freezes the code at that point in time           │
│  • Creates a specific VERSION (v1, v2, v3...)       │
│  • Code changes do NOT affect existing deployments  │
│  • Required for: Web App URLs, Marketplace publish  │
│  • Gets its own unique Deployment ID & URL          │
│                                                     │
│  How to create:                                     │
│  Deploy → New deployment → Select type → Deploy     │
└─────────────────────────────────────────────────────┘
```

**Key behavior:** The deployed version is frozen. Even if you edit the code, the deployed version stays the same. You must create a NEW deployment to publish changes.

### 8.3 New Deployment Types

When you click "New Deployment", you choose a **type**:

```
New Deployment
    │
    ├── Web App
    │   └── Gets a URL immediately (https://script.google.com/macros/s/.../exec)
    │   └── Anyone with the URL can access it
    │   └── No Marketplace, no Google review
    │
    ├── API Executable
    │   └── Called from external apps via Apps Script API
    │
    ├── Add-on
    │   └── For publishing to Google Workspace Marketplace
    │   └── Requires Google review (takes days to weeks)
    │   └── This is NOT needed for company-internal add-ons!
    │
    └── Library
        └── Other Apps Script projects can import this as a library
```

### 8.4 When to Use Which?

| Scenario | Use |
|---|---|
| Testing your add-on internally | **Test Deployment** |
| Company-internal add-on (admin installs for all employees) | **Test Deployment** (admin install via Deployment ID) |
| Chat App (needs Deployment ID for GCP config) | **Test Deployment** or **New Deployment** |
| Web App with a shareable URL | **New Deployment → Web App** |
| Publishing to Marketplace for external users | **New Deployment → Add-on** |
| Background automation (triggers only) | No deployment needed — just set up triggers |

### 8.5 Common Misconception

> **"I need a New Deployment to share my add-on with my company."**
>
> ❌ **Wrong!** For company-internal distribution, use **Test Deployment**. Your Google Workspace admin can install test deployments for all employees via the Admin Console. New Deployment → Add-on type is ONLY for Marketplace publishing (external/public distribution).

```
Company-Internal Add-on                    Public Marketplace Add-on
─────────────────────────                   ─────────────────────────

Test Deployment                            New Deployment → Add-on
    │                                           │
    ▼                                           ▼
Admin Console → Install                    Submit to Google for review
for all employees                          Wait days/weeks for approval
    │                                           │
    ▼                                           ▼
✅ Done! Always latest code.               Published on Marketplace.
                                           Frozen at deployed version.
```

---

## 9. How to Create a Google Workspace Add-on — Step by Step with Code Explanation

Let's build a **Gmail Add-on** that shows the sender's info when you open an email.

### Step 1: Create the Project

1. Go to [script.google.com](https://script.google.com)
2. Click **+ New Project**
3. Name it: "My Gmail Add-on"

### Step 2: Configure the Manifest (`appsscript.json`)

In the Apps Script editor:
1. Click the **gear icon** (Project Settings) on the left
2. Check **"Show `appsscript.json` manifest file in editor"**
3. Click on `appsscript.json` in the file list
4. Replace the content with:

```json
{
  "timeZone": "Asia/Kolkata",
  "dependencies": {},
  "exceptionLogging": "STACKDRIVER",
  "runtimeVersion": "V8",

  "oauthScopes": [
    "https://www.googleapis.com/auth/gmail.readonly",
    "https://www.googleapis.com/auth/script.locale"
  ],

  "addOns": {
    "common": {
      "name": "Email Info Add-on",
      "logoUrl": "https://www.gstatic.com/images/branding/product/2x/apps_script_48dp.png",
      "homepageTrigger": {
        "runFunction": "onHomepage"
      }
    },
    "gmail": {
      "contextualTriggers": [
        {
          "unconditional": {},
          "onTriggerFunction": "onGmailMessage"
        }
      ]
    }
  }
}
```

**Line-by-line explanation:**

```
"timeZone": "Asia/Kolkata"
```
↳ Sets the timezone for date/time operations in the script.

```
"dependencies": {}
```
↳ No external library dependencies. If you use a library, it goes here.

```
"exceptionLogging": "STACKDRIVER"
```
↳ Errors are logged to Google Cloud's Stackdriver (now called Cloud Logging). You can view them in the Apps Script dashboard.

```
"runtimeVersion": "V8"
```
↳ Use the modern V8 JavaScript engine (supports ES6+ features like `let`, `const`, arrow functions, etc.).

```
"oauthScopes": [...]
```
↳ **THIS IS THE KEY PART.** These are the permissions your script needs.
↳ `gmail.readonly` = permission to read the user's Gmail (but not send/delete)
↳ `script.locale` = permission to know the user's language/locale
↳ **These scopes directly control what appears on the consent screen!** (more on this in Section 10)

```
"addOns": { ... }
```
↳ This block tells Google: "This script is an add-on."

```
"common": { ... }
```
↳ Settings that apply to ALL Google products (Gmail, Docs, Sheets, etc.)

```
"name": "Email Info Add-on"
```
↳ The name users see when they install the add-on.

```
"logoUrl": "https://..."
```
↳ The icon shown in the sidebar. Must be a publicly accessible image URL.

```
"homepageTrigger": { "runFunction": "onHomepage" }
```
↳ When the user clicks on the add-on icon (without any email open), call the `onHomepage` function.

```
"gmail": { ... }
```
↳ Settings specific to Gmail.

```
"contextualTriggers": [...]
```
↳ These triggers fire when specific events happen in Gmail.

```
"unconditional": {}
```
↳ Fire the trigger for ALL emails (no conditions). You could add conditions like "only trigger for emails from a specific domain."

```
"onTriggerFunction": "onGmailMessage"
```
↳ When the user opens any email, call the `onGmailMessage` function.

### Step 3: Write the Main Code (`Code.gs`)

Click on `Code.gs` in the file list and write:

```javascript
/**
 * Called when the user opens the add-on without any email selected.
 * This is the "home screen" of the add-on.
 */
function onHomepage(e) {
  // CardService — this is Google's UI framework for add-ons
  // It creates "cards" (like small UI panels) that appear in the sidebar
  
  // Step 1: Create a card header
  var header = CardService.newCardHeader()  // ← creates a header object
    .setTitle("📧 Email Info Add-on")        // ← the big title text
    .setSubtitle("Open an email to see info"); // ← smaller text below title

  // Step 2: Create a text widget
  var welcomeText = CardService.newTextParagraph()  // ← a simple text block
    .setText("Welcome! Open any email in your inbox, and this add-on will show you details about the sender.");

  // Step 3: Create a section and add the text widget to it
  // Cards are organized as: Card → Sections → Widgets
  var section = CardService.newCardSection()  // ← a grouping container
    .addWidget(welcomeText);                   // ← add the text widget to section

  // Step 4: Build the card
  var card = CardService.newCardBuilder()  // ← start building a card
    .setHeader(header)                      // ← attach the header
    .addSection(section)                    // ← attach the section
    .build();                               // ← finalize the card object

  // Step 5: Return the card (Google renders it in the sidebar)
  // Must return an ARRAY of cards
  return [card];
}


/**
 * Called when the user opens an email in Gmail.
 * The event object 'e' contains info about the opened email.
 */
function onGmailMessage(e) {
  // 'e' is the EVENT OBJECT — Google passes this automatically
  // It contains: e.gmail.messageId — the ID of the opened email

  // Step 1: Get the message ID from the event
  var messageId = e.gmail.messageId;
  // ↳ This is a unique ID like "18a3b4c5d6e7f8g9"
  // ↳ Google tells us WHICH email the user opened

  // Step 2: Use GmailApp service to get the full message
  var message = GmailApp.getMessageById(messageId);
  // ↳ GmailApp is a built-in service — no import needed
  // ↳ This fetches the full email object from Gmail's servers
  // ↳ This is where gmail.readonly permission is needed!

  // Step 3: Extract useful information
  var subject = message.getSubject();         // "Meeting Tomorrow"
  var sender = message.getFrom();             // "john@company.com"
  var dateSent = message.getDate();           // Date object
  var snippet = message.getPlainBody()        // Get email body as plain text
    .substring(0, 200);                        // Take first 200 characters only

  // Step 4: Build the UI card with this information
  var header = CardService.newCardHeader()
    .setTitle("Email Details");

  // Create individual text widgets for each piece of info
  var senderWidget = CardService.newKeyValue()     // ← key-value pair widget
    .setTopLabel("From")                            // ← small label on top
    .setContent(sender);                            // ← the actual value

  var subjectWidget = CardService.newKeyValue()
    .setTopLabel("Subject")
    .setContent(subject);

  var dateWidget = CardService.newKeyValue()
    .setTopLabel("Sent On")
    .setContent(dateSent.toLocaleString());        // ← convert date to readable string

  var snippetWidget = CardService.newTextParagraph()
    .setText("📝 Preview: " + snippet + "...");

  // Step 5: Create a section and add all widgets
  var infoSection = CardService.newCardSection()
    .setHeader("📋 Message Info")                  // ← section header
    .addWidget(senderWidget)
    .addWidget(subjectWidget)
    .addWidget(dateWidget)
    .addWidget(snippetWidget);

  // Step 6: Build and return the card
  var card = CardService.newCardBuilder()
    .setHeader(header)
    .addSection(infoSection)
    .build();

  return [card];
}
```

### Step 4: The Card UI Architecture

Understanding the structure of the card UI:

```
┌─────────────────────────────────────┐
│            CARD                     │  ← CardService.newCardBuilder()
│  ┌───────────────────────────────┐  │
│  │          HEADER               │  │  ← .setHeader()
│  │  Title: "Email Details"       │  │
│  └───────────────────────────────┘  │
│                                     │
│  ┌───────────────────────────────┐  │
│  │         SECTION 1             │  │  ← .addSection()
│  │  ┌─────────────────────────┐  │  │
│  │  │  Widget: KeyValue       │  │  │  ← .addWidget()
│  │  │  From: john@company.com │  │  │
│  │  └─────────────────────────┘  │  │
│  │  ┌─────────────────────────┐  │  │
│  │  │  Widget: KeyValue       │  │  │
│  │  │  Subject: Meeting       │  │  │
│  │  └─────────────────────────┘  │  │
│  │  ┌─────────────────────────┐  │  │
│  │  │  Widget: TextParagraph  │  │  │
│  │  │  Preview: Hello, I...   │  │  │
│  │  └─────────────────────────┘  │  │
│  └───────────────────────────────┘  │
│                                     │
│  ┌───────────────────────────────┐  │
│  │         SECTION 2 (optional)  │  │
│  │  More widgets...              │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
```

### Step 5: Test the Add-on

1. In the Apps Script editor, click **Deploy** → **Test deployments**
2. Click **Install** next to "Gmail" (under "Application(s)")
3. Open Gmail in another tab
4. Look at the **right sidebar** — you'll see your add-on icon
5. Click on it → the homepage card appears
6. Open any email → the contextual trigger fires → email details card appears

```
Gmail Interface
┌──────────────────────────────────────────────────────────────┐
│  ☰  Gmail                        🔍 Search mail             │
├──────────────────────────────────────┬───────────────────────┤
│                                      │                       │
│  📥 Inbox                            │  ┌─────────────────┐  │
│  ⭐ Starred                          │  │ YOUR ADD-ON     │  │
│  📤 Sent                             │  │ ─────────────── │  │
│                                      │  │ Email Details   │  │
│  ┌────────────────────────────────┐  │  │                 │  │
│  │ From: john@company.com        │  │  │ From:           │  │
│  │ Subject: Meeting Tomorrow     │  │  │ john@company    │  │
│  │                               │  │  │                 │  │
│  │ Hi, I wanted to discuss...    │  │  │ Subject:        │  │
│  │                               │  │  │ Meeting Tomorrow│  │
│  └────────────────────────────────┘  │  │                 │  │
│                                      │  │ Sent On:        │  │
│                                      │  │ May 23, 2026    │  │
│                                      │  └─────────────────┘  │
│              EMAIL BODY              │    SIDEBAR (Add-on)    │
└──────────────────────────────────────┴───────────────────────┘
```

### Step 6: Publish (covered in Section 13)

---

## 10. Google Chat App — Step by Step

> **⚠️ Important: Google Chat API requires a Google Workspace account.** If you have a personal `@gmail.com` account, the Chat API will NOT work. You will see this error:
>
> *"Google Chat API is only available to Google Workspace users."*
>
> **Workarounds if you don't have Workspace:**
> - Use your company's Google Workspace account (if your company uses Google Workspace)
> - Sign up for a [Google Workspace 14-day free trial](https://workspace.google.com/)
> - Build a Gmail Add-on instead (works with personal Gmail accounts)

### Step 1: Create a Google Cloud Project

1. Go to [console.cloud.google.com](https://console.cloud.google.com/)
2. Click **Select a project** → **New Project**
3. Name it: "My Chat Bot"
4. Click **Create**

### Step 2: Enable Google Chat API

1. In the GCP Console → **APIs & Services** → **Enable APIs**
2. Search for **"Google Chat API"**
3. Click **Enable**

### Step 3: Create the Apps Script Project

1. Go to [script.google.com](https://script.google.com) → **New Project**
2. Name it: "My Chat Bot"

### Step 4: Link to GCP Project

1. In Apps Script editor → **Project Settings** (gear icon)
2. Under "Google Cloud Platform (GCP) Project" → click **Change project**
3. Enter the GCP project number (found in GCP Console → Dashboard)

### Step 5: Write the Chat Handler

```javascript
/**
 * Called when someone sends a message to the bot.
 * 'event' contains all the message info.
 */
function onMessage(event) {
  // event.message.text — the text the user typed
  var userMessage = event.message.text;
  // ↳ e.g., "Hello bot" or "/help"

  // event.user.displayName — who sent the message
  var userName = event.user.displayName;
  // ↳ e.g., "Ritesh Singh"

  // Check if user used a slash command
  if (event.message.slashCommand) {
    // event.message.slashCommand.commandId — the ID of the slash command
    var commandId = event.message.slashCommand.commandId;
    
    switch (commandId) {
      case 1:  // /help command
        return { "text": "Available commands:\n/help - Show this help\n/status - Check status" };
      case 2:  // /status command
        return { "text": "✅ All systems operational!" };
    }
  }

  // Default response — just echo back
  return {
    "text": "Hi " + userName + "! You said: " + userMessage
  };
}

/**
 * Called when the bot is added to a Chat space or DM.
 */
function onAddToSpace(event) {
  // event.space.type — "DM" or "ROOM"
  var spaceType = event.space.type;

  if (spaceType === "DM") {
    return { "text": "👋 Hi! Thanks for adding me. Send me a message!" };
  } else {
    return { "text": "👋 Thanks for adding me to this space! Use /help to see commands." };
  }
}

/**
 * Called when the bot is removed from a space.
 */
function onRemoveFromSpace(event) {
  // Clean up any data if needed
  console.log("Bot removed from space: " + event.space.name);
}
```

### Step 6: Deploy and Configure the Chat App in GCP

**First, create a deployment:**

1. In the Apps Script editor, click **Deploy** → **Test deployments** (for internal use) or **New deployment** (for versioned)
2. Copy the **Deployment ID** (NOT the Script ID — this is a common mistake!)

**Then, configure in GCP:**

1. Go to GCP Console → **APIs & Services** → **Google Chat API** → **Configuration**
2. Fill in:
   - **App name:** "My Company Bot"
   - **Avatar URL:** (any image URL)
   - **Description:** "A helpful company bot"
   - **Functionality:** Check "Receive 1:1 messages" and "Join spaces and group conversations"
   - **Connection settings:** Select **Apps Script**
   - **Deployment ID:** Paste the **Deployment ID** you copied above
   - **Slash commands:** Add `/help` (ID: 1), `/status` (ID: 2)
3. Click **Save**

> **⚠️ Common Mistake:** The GCP Chat API configuration asks for a **Deployment ID**, NOT the Script ID. The Script ID identifies your project; the Deployment ID identifies a specific deployed version of your code. If you enter the Script ID, the bot will not work.

### Step 7: Test

1. Open [chat.google.com](https://chat.google.com)
2. Click **+ New chat** → Search for your bot name
3. Send it a message!

### Step 8: Billing — Does a Chat App Cost Money?

**Short answer: NO — for most use cases, a Chat App built with Apps Script is completely free.**

```
┌─────────────────────────────────────────────────────────────────────┐
│                     BILLING BREAKDOWN                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ✅ FREE (no billing needed):                                       │
│  ├── Google Apps Script execution          (free, with quotas)      │
│  ├── Google Chat API                       (free, no per-call cost) │
│  ├── GCP project creation                  (free)                   │
│  ├── Receiving & responding to messages    (free)                   │
│  ├── Slash commands                        (free)                   │
│  ├── Card-based responses                  (free)                   │
│  ├── Using Google services (Sheets, Drive) (free, within quotas)    │
│  └── Test & New deployments                (free)                   │
│                                                                     │
│  💰 PAID (billing required):                                        │
│  ├── Calling EXTERNAL paid APIs from your script                    │
│  │   (e.g., OpenAI API, Azure AI, custom cloud endpoints)          │
│  ├── Using GCP services beyond free tier                            │
│  │   (e.g., Cloud Functions, BigQuery, Cloud Storage, Vertex AI)   │
│  └── Exceeding Apps Script quotas                                   │
│       (you can't pay to increase — you'd need to move off          │
│        Apps Script to Cloud Functions/Cloud Run)                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Why is it free?**

1. **Google Chat API** itself has no per-call charges — it's included with your Google Workspace subscription. Your company already pays for Workspace (Gmail, Drive, Chat, etc.), and the Chat API is part of that.

2. **Apps Script** is a free runtime provided by Google. It has daily quotas (e.g., 6 min/execution, 90 min total/day for consumer accounts, higher for Workspace), but there is no per-execution cost.

3. **GCP project** is free to create and use for enabling APIs. You only pay if you use paid GCP services (like Vertex AI, Cloud SQL, etc.).

**When does billing become relevant?**

| Scenario | Billing needed? |
|---|---|
| Simple Chat bot (echo, slash commands, read Sheets) | ❌ No |
| Chat bot that calls OpenAI/Gemini API for AI responses | ✅ Yes — for the external API costs |
| Chat bot using a GCP service account to push messages | ❌ No — Chat API is free |
| Chat bot storing data in Google Sheets | ❌ No |
| Chat bot storing data in BigQuery or Cloud SQL | ✅ Yes — for the GCP service |
| RAG-based chatbot using external vector DB | ✅ Yes — for the vector DB/AI API costs |
| Chat bot within Apps Script quotas | ❌ No |
| Need more execution time than Apps Script allows | ⚠️ Must migrate to Cloud Functions (paid) |

> **Bottom line:** If your Chat App only uses Google's built-in services (Sheets, Drive, Gmail, Calendar) and stays within Apps Script quotas, you pay **nothing extra** beyond your existing Google Workspace subscription. Billing only matters when you use external paid APIs or GCP paid services.

> **GCP Billing Account:** Even though the Chat App itself is free, GCP may ask you to set up a billing account to enable certain APIs. You can set it up without being charged — you'll only be billed if you actually consume paid resources.

---

## 11. Extending a Chat App to Other Products — Gmail Add-on from Same Project

You already have a Chat App working in Google Chat. Now you want to add a **Gmail Add-on** (sidebar) to the same project — so users can also interact with your bot from within Gmail.

### 11.1 Key Concept: One Project, Multiple Deployment Types

A single Apps Script project can be **both** a Chat App and a Workspace Add-on simultaneously. They share the same GCP project but have completely independent distribution channels.

```
┌─────────────────────────────────────────────────────────────────┐
│          SINGLE APPS SCRIPT PROJECT                             │
│                                                                 │
│   appsscript.json contains BOTH:                                │
│   ┌─────────────────────┐   ┌──────────────────────────────┐   │
│   │  "chat": { ... }    │   │  "addOns": {                 │   │
│   │                     │   │    "gmail": { ... }           │   │
│   │  → Chat App in      │   │                              │   │
│   │    Google Chat      │   │  → Sidebar in Gmail          │   │
│   └─────────────────────┘   └──────────────────────────────┘   │
│                                                                 │
│   Shared Code:                                                  │
│   ┌───────────────────────────────────────────────────┐         │
│   │  ragSearch(), callExternalAPI(), processQuery()   │         │
│   │  (used by BOTH onMessage and handleGmailQuery)    │         │
│   └───────────────────────────────────────────────────┘         │
│                                                                 │
│   Same GCP Project:                                             │
│   ┌───────────────────────────────────────────────────┐         │
│   │  Chat API enabled + Service Account + OAuth       │         │
│   │  (all features share the same GCP project)        │         │
│   └───────────────────────────────────────────────────┘         │
└─────────────────────────────────────────────────────────────────┘
```

### 11.2 How to Add Gmail Add-on to an Existing Chat App

**Step 1:** Update your `appsscript.json` to include the `addOns` block alongside the existing `chat` block:

```json
{
  "timeZone": "Asia/Kolkata",
  "dependencies": {},
  "runtimeVersion": "V8",
  "chat": {},
  "oauthScopes": [
    "https://www.googleapis.com/auth/gmail.readonly",
    "https://www.googleapis.com/auth/chat.bot"
  ],
  "addOns": {
    "common": {
      "name": "My Company Assistant",
      "logoUrl": "https://your-logo-url.png",
      "homepageTrigger": {
        "runFunction": "onAddOnHomepage"
      }
    },
    "gmail": {
      "contextualTriggers": [
        {
          "unconditional": {},
          "onTriggerFunction": "onGmailMessage"
        }
      ]
    }
  }
}
```

**Step 2:** Write separate handler functions for the add-on (CardService-based):

```javascript
// ============ CHAT APP HANDLERS ============

function onMessage(event) {
  var query = event.message.text;
  var result = processQuery(query);  // Shared logic!
  return { "text": result };
}

// ============ GMAIL ADD-ON HANDLERS ============

function onAddOnHomepage(e) {
  var userEmail = Session.getActiveUser().getEmail();
  var card = CardService.newCardBuilder()
    .setHeader(CardService.newCardHeader().setTitle("Company Assistant"))
    .addSection(CardService.newCardSection()
      .addWidget(CardService.newTextParagraph()
        .setText("Welcome, " + userEmail + "! Open an email to get started.")))
    .build();
  return [card];
}

function onGmailMessage(e) {
  var messageId = e.gmail.messageId;
  var message = GmailApp.getMessageById(messageId);
  var subject = message.getSubject();
  
  // Use the SAME shared logic as the Chat App
  var analysis = processQuery("Analyze email: " + subject);
  
  var card = CardService.newCardBuilder()
    .setHeader(CardService.newCardHeader().setTitle("Email Analysis"))
    .addSection(CardService.newCardSection()
      .addWidget(CardService.newTextParagraph().setText(analysis)))
    .build();
  return [card];
}

// ============ SHARED LOGIC (used by both) ============

function processQuery(query) {
  // Your RAG search, API calls, etc.
  // This function is called by BOTH onMessage (Chat) and onGmailMessage (Add-on)
  return "Result for: " + query;
}
```

### 11.3 Distribution is Independent

The Chat App and Gmail Add-on have **completely separate distribution channels**:

```
┌─────────────────────────────────┐    ┌───────────────────────────────────┐
│       CHAT APP USERS            │    │       GMAIL ADD-ON USERS          │
│                                 │    │                                   │
│  Controlled via:                │    │  Controlled via:                  │
│  GCP Console → Chat API →      │    │  Test Deployment → Admin Console  │
│  Configuration → Add users     │    │  → Install for employees          │
│                                 │    │                                   │
│  Users find the bot in          │    │  Users see the add-on icon in     │
│  Google Chat → search bot name  │    │  Gmail sidebar automatically     │
│                                 │    │                                   │
│  Adding someone to Chat does    │    │  Installing add-on does NOT       │
│  NOT give them the Gmail add-on │    │  give them the Chat bot           │
└─────────────────────────────────┘    └───────────────────────────────────┘
```

### 11.4 Service Accounts Work from Both

If your Chat App uses a **GCP service account** (e.g., to push proactive messages), the same service account works from add-on code too — because the GCP project is linked at the **project level**, not per deployment type.

### 11.5 Getting the User's Email in an Add-on

In a Chat App, you get the user from `event.user.email`. In an Add-on, use:

```javascript
var userEmail = Session.getActiveUser().getEmail();
// Returns the email of the user who is running the add-on
// e.g., "ritesh@company.com"
```

---

## 12. The Consent Screen — Complete Deep Dive

This is one of the most misunderstood parts. Let's break it down completely.

### 12.1 What is the Consent Screen?

The consent screen is a **popup/page shown by Google** that asks the user to grant permissions to the script. It looks like this:

```
┌──────────────────────────────────────────────────┐
│                                                  │
│            Sign in with Google                   │
│                                                  │
│  "Email Info Add-on" wants to:                   │
│                                                  │
│  ☑ Read your Gmail messages                      │
│  ☑ Know your locale/language                     │
│                                                  │
│  ┌────────────────┐  ┌──────────────────────┐    │
│  │    Cancel       │  │  Allow / Authorize   │    │
│  └────────────────┘  └──────────────────────┘    │
│                                                  │
│  This app is not verified by Google.             │
│  Only proceed if you trust the developer.        │
│                                                  │
└──────────────────────────────────────────────────┘
```

### 12.2 When Does the Consent Screen Appear?

```
Does the consent screen appear for ALL scripts?
│
├── Script running in the editor (you click "Run")
│   └── YES — first time only, then remembered
│
├── Container-bound script (menu/button in Sheet)
│   └── YES — first time only for each user
│
├── Web App
│   └── YES — first time a user opens the URL
│
├── Add-on
│   └── YES — when the user installs the add-on
│
├── Time-driven trigger (cron job)
│   └── YES — when the owner creates the trigger
│       (runs automatically after that, no more consent)
│
└── Simple trigger (onOpen, onEdit — no special permissions)
    └── NO — these run automatically without consent
        (but they have very limited permissions)
```

**Rule: The consent screen appears ONCE per user per script. After the user approves, Google remembers the grant and doesn't ask again (unless scopes change).**

### 12.3 How Does the Consent Screen Know Which Permissions to Show?

**THIS is the critical question.** The answer: **OAuth Scopes in the manifest.**

```
   Your appsscript.json                        Consent Screen
   ─────────────────────                        ──────────────

   "oauthScopes": [                        "This app wants to:"
     "https://www.googleapis.com              
      /auth/gmail.readonly",    ─────────→   ☑ Read your Gmail
                                             
     "https://www.googleapis.com              
      /auth/drive",             ─────────→   ☑ See, edit, create &
                                               delete all your Drive files
     "https://www.googleapis.com
      /auth/calendar",          ─────────→   ☑ See, edit, share &
                                               delete your calendar
     "https://www.googleapis.com
      /auth/spreadsheets",      ─────────→   ☑ See, edit, create &
                                               delete your spreadsheets
   ]
```

**There are two ways scopes are determined:**

#### Way 1: Automatic Detection (Default)

If you DON'T specify `oauthScopes` in `appsscript.json`, Google **automatically scans your code** and detects which services you use:

```javascript
GmailApp.getInboxThreads();
// ↳ Google detects this and adds: gmail.readonly or gmail.modify

SpreadsheetApp.openById("...");
// ↳ Google detects this and adds: spreadsheets

DriveApp.getFiles();
// ↳ Google detects this and adds: drive
```

**Problem with auto-detection:** Google often picks **broad permissions** (e.g., full drive access instead of read-only). This scares users.

#### Way 2: Manual Specification (Recommended)

You explicitly list the scopes in `appsscript.json`:

```json
"oauthScopes": [
  "https://www.googleapis.com/auth/gmail.readonly",
  "https://www.googleapis.com/auth/spreadsheets.readonly"
]
```

**Advantage:** You can pick **narrow/minimal scopes**. For example:
- `spreadsheets` = full access (read, write, delete)
- `spreadsheets.readonly` = only read access (much less scary to users)

### 12.4 Common OAuth Scopes Reference

| Scope | What it allows | Consent screen shows |
|---|---|---|
| `gmail.readonly` | Read emails | "Read your Gmail" |
| `gmail.send` | Send emails | "Send email on your behalf" |
| `gmail.modify` | Read + modify + delete emails | "Read, compose, send, and permanently delete your email" |
| `drive` | Full Drive access | "See, edit, create, and delete all your Drive files" |
| `drive.readonly` | Read-only Drive | "See and download all your Drive files" |
| `drive.file` | Only files the app created | "See, edit, create, and delete only the specific Drive files you use with this app" |
| `spreadsheets` | Full Sheets access | "See, edit, create, and delete your spreadsheets" |
| `spreadsheets.readonly` | Read-only Sheets | "See all your spreadsheets" |
| `calendar` | Full Calendar access | "See, edit, share, and permanently delete your calendars" |
| `calendar.readonly` | Read-only Calendar | "See your calendars" |
| `script.external_request` | Make HTTP calls to external APIs | "Connect to an external service" |

### 12.5 The Full Consent Flow — End to End

Here is the **complete flow** of what happens from the moment a user first interacts with your script:

```
Step 1: USER TRIGGERS THE SCRIPT
────────────────────────────────
User clicks the Web App URL / installs add-on / runs from menu
         │
         ▼
Step 2: GOOGLE CHECKS — Has this user approved this script before?
──────────────────────────────────────────────────────────────────
         │
         ├── YES → Skip to Step 6 (run the script directly)
         │
         └── NO → Continue to Step 3
                    │
                    ▼
Step 3: GOOGLE READS THE MANIFEST (appsscript.json)
────────────────────────────────────────────────────
Google reads the "oauthScopes" array from the manifest.
If no scopes are specified, Google auto-detects from the code.
         │
         ▼
Step 4: GOOGLE SHOWS THE CONSENT SCREEN
─────────────────────────────────────────
┌──────────────────────────────────────────────────────┐
│                                                      │
│  Google shows a page/popup:                          │
│                                                      │
│  "Email Info Add-on wants access to your             │
│   Google Account"                                    │
│                                                      │
│  This will allow "Email Info Add-on" to:             │
│  • Read your Gmail messages                          │  ← from gmail.readonly
│  • Know your locale                                  │  ← from script.locale
│                                                      │
│  [Cancel]                         [Allow]            │
│                                                      │
│  ⚠️ If the script is NOT verified:                   │
│  "This app isn't verified" warning appears.          │
│  User must click "Advanced" → "Go to [app name]"    │
│                                                      │
└──────────────────────────────────────────────────────┘
         │
         ├── User clicks "Cancel" → Nothing happens. Script does NOT run.
         │
         └── User clicks "Allow" → Continue to Step 5
                    │
                    ▼
Step 5: GOOGLE CREATES AN OAUTH TOKEN
──────────────────────────────────────
- Google creates an ACCESS TOKEN for the user
- This token contains:
  - WHO the user is (their Google account)
  - WHAT permissions they granted (the scopes)
  - WHEN it expires (usually 1 hour, auto-refreshed)
- Google stores this grant so it doesn't ask again
         │
         ▼
Step 6: SCRIPT EXECUTES ON GOOGLE'S SERVERS
────────────────────────────────────────────
- Google's server receives the request
- It runs your JavaScript code
- When your code calls GmailApp.getInboxThreads():
  1. The Apps Script runtime checks the OAuth token
  2. Does the token have gmail.readonly scope? YES → proceed
  3. Makes internal API call to Gmail with the token
  4. Gmail returns the data
  5. Your script processes it
         │
         ▼
Step 7: RESULT RETURNED TO USER
───────────────────────────────
- Web App: HTML page is sent back to the browser
- Add-on: Card UI is rendered in the sidebar
- Chat App: Message is sent to Chat
```

### 12.6 The "This app isn't verified" Warning

```
┌───────────────────────────────────────────────────────────────┐
│                                                               │
│  ⚠️ Google hasn't verified this app                          │
│                                                               │
│  The app is requesting access to sensitive info in your       │
│  Google Account. Until the developer verifies this app        │
│  with Google, you shouldn't use it.                           │
│                                                               │
│  [Back to safety]                                             │
│                                                               │
│  ▼ Advanced                                                   │
│  └── "Go to Email Info Add-on (unsafe)"  ← click this        │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

**When does this appear?**
- When your script uses **sensitive or restricted scopes** (like Gmail, Drive)
- AND the script has NOT gone through **Google's OAuth verification process**

**How to remove it:**
- For **company-internal** scripts: Your Google Workspace admin can mark the app as "trusted" in Admin Console → Security → API Controls → App Access Control
- For **public** scripts: Submit for Google's OAuth verification (takes weeks)

### 12.7 Where to Set Up the Consent Screen

The consent screen details come from **two places**:

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  SOURCE 1: appsscript.json (in your Apps Script project)     │
│  ────────────────────────────────────────────────────────     │
│  Controls:                                                   │
│  • Which permissions (oauthScopes) are requested             │
│  • The app name (from addOns.common.name)                    │
│                                                              │
│  SOURCE 2: Google Cloud Console (GCP Project)                │
│  ────────────────────────────────────────────────────────     │
│  Controls:                                                   │
│  • OAuth consent screen branding (logo, privacy policy URL)  │
│  • App description shown to users                            │
│  • Developer contact info                                    │
│  • Whether app is "Internal" (company only) or "External"    │
│  • Verified/unverified status                                │
│                                                              │
│  Location in GCP Console:                                    │
│  APIs & Services → OAuth consent screen                      │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**Steps to configure in GCP Console:**

1. Go to [console.cloud.google.com](https://console.cloud.google.com/)
2. Select your project
3. Navigate to: **APIs & Services → OAuth consent screen**
4. Choose user type:
   - **Internal** = only users in your Google Workspace org (no verification needed)
   - **External** = any Google user (needs verification for sensitive scopes)
5. Fill in:
   - App name
   - User support email
   - App logo
   - Authorized domains
   - Developer contact email
6. On the "Scopes" page, add the same scopes from your `appsscript.json`
7. Click **Save**

---

## 13. GCP Project — Auto-Created vs Manual & Sharing Scripts in a Company

### 13.1 When Do You Need a GCP Project?

Not all Apps Script projects need a manually created GCP project. Google auto-creates one behind the scenes in many cases.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                  GCP PROJECT REQUIREMENT                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Web App / Simple Script:                                               │
│  ├── GCP project auto-created (hidden, managed by Google)               │
│  ├── You NEVER need to touch GCP Console                                │
│  └── Just deploy and share the URL                                      │
│                                                                         │
│  Gmail/Sheets/Docs Add-on (company-internal):                           │
│  ├── GCP project auto-created for basic testing                         │
│  ├── Manual GCP needed ONLY for:                                        │
│  │   ├── Marketplace publishing (external distribution)                 │
│  │   ├── Admin install across the organization                          │
│  │   ├── OAuth consent screen customization                             │
│  │   └── OAuth verification (to remove "unverified" warning)            │
│  └── For test deployments used personally → auto-created GCP is fine    │
│                                                                         │
│  Chat App:                                                              │
│  ├── Manual GCP project ALWAYS required ❗                               │
│  ├── WHY? Because you must:                                             │
│  │   ├── Enable the Google Chat API (disabled by default)               │
│  │   ├── Configure connection settings (Deployment ID, bot name, etc.)  │
│  │   └── Register the bot (slash commands, permissions)                 │
│  └── There is NO auto-creation path for Chat Apps                       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

> **Why the difference?** Add-ons and Web Apps use standard OAuth flows that Google can handle automatically. Chat Apps require explicit API enablement and bot registration in GCP — there's no way to auto-configure this.

### 13.2 Option A: Share as Web App (Simplest)

```
Developer                                     Users
────────                                      ─────

1. Write code in script.google.com
2. Deploy → New Deployment → Web App
3. Set "Execute as" → Me or User
4. Set "Who has access" →
   "Anyone in [company]"
5. Click Deploy
6. Get URL ──────────────────────────────→  7. Click URL
                                            8. See consent screen (first time)
                                            9. Click "Allow"
                                            10. Use the app!
```

### 13.3 Option B: Share as Add-on (Within Company)

```
Developer                                     Google Admin
────────                                      ────────────

1. Write code with Card Service
2. Configure appsscript.json
3. Deploy → Test deployments                  
4. Get the Deployment ID                      
5. Create GCP project                         
6. Configure OAuth consent screen             
   (set as "Internal")                        
7. Share Deployment ID with Admin ───────→    8. Go to Admin Console
                                              9. Apps → Google Workspace
                                                 Marketplace apps
                                              10. Add → "Install from
                                                  test deployment"
                                              11. Enter Deployment ID
                                              12. Install for all users
                                                  or specific OUs
                                                        │
                                                        ▼
                                              All employees now see
                                              the add-on in Gmail/
                                              Sheets/Docs automatically
```

### 13.4 Option C: Publish to Google Workspace Marketplace (External)

```
┌────────────────────────────────────────────────────────────────┐
│                 Marketplace Publishing Flow                    │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  1. Create GCP Project                                        │
│  2. Enable required APIs (Gmail API, Drive API, etc.)         │
│  3. Configure OAuth consent screen → "External"               │
│  4. Write and test your add-on code                           │
│  5. Deploy → New Deployment → Add-on                          │
│  6. Go to: Google Workspace Marketplace SDK (in GCP)          │
│  7. Fill in: listing info, screenshots, pricing               │
│  8. Submit for Google Review                                  │
│     ├── Review takes days to weeks                            │
│     ├── Google checks for policy compliance                   │
│     └── May ask for OAuth verification too                    │
│  9. Once approved → Live on Marketplace                       │
│  10. Anyone can search and install                            │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### 13.5 Access Control Summary

| Who has access? | Meaning |
|---|---|
| Only myself | Only you can use it (testing) |
| Anyone within [your company domain] | All employees in your Google Workspace org |
| Anyone | Any Google user on the internet |
| Anyone (even anonymous) | People without a Google account (Web App only, limited) |

---

## 14. Common Company Pattern — Shared Sheet as Database

The most common pattern in companies:

```
┌──────────────────────────────────────────────────────────┐
│                                                          │
│   Owner creates a Google Sheet ("Company Database")      │
│   Owner shares it with all employees (Editor access)     │
│   Script is hardcoded to use THIS sheet's ID             │
│                                                          │
│   var SHEET_ID = "1BxiMVs0XRA5nFMdKvBd...";             │
│   var sheet = SpreadsheetApp.openById(SHEET_ID);         │
│                                                          │
└────────────────────────────┬─────────────────────────────┘
                             │
                             ▼
   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
   │  Employee A  │  │  Employee B  │  │  Employee C  │
   │  runs script │  │  runs script │  │  runs script │
   │      │       │  │      │       │  │      │       │
   │      ▼       │  │      ▼       │  │      ▼       │
   │  Reads/Writes│  │  Reads/Writes│  │  Reads/Writes│
   │  to the SAME │  │  to the SAME │  │  to the SAME │
   │  shared sheet│  │  shared sheet│  │  shared sheet│
   └──────────────┘  └──────────────┘  └──────────────┘
                             │
                             ▼
                 ┌───────────────────────┐
                 │  "Company Database"   │
                 │  Google Sheet         │
                 │  (in Owner's Drive,   │
                 │   shared with all)    │
                 └───────────────────────┘
```

This works regardless of "Execute as Me" or "Execute as User" — because the sheet ID is hardcoded and all users have been granted access to that specific sheet.

**Example code:**

```javascript
// Hardcoded sheet ID — this sheet is shared with all employees
var SHEET_ID = "1BxiMVs0XRA5nFMdKvBdIq...";

function addEntry() {
  var sheet = SpreadsheetApp.openById(SHEET_ID);
  // ↳ Opens the shared company sheet

  var activeSheet = sheet.getActiveSheet();
  // ↳ Gets the first/active tab in the spreadsheet

  activeSheet.appendRow([
    new Date(),                    // Column A: timestamp
    Session.getActiveUser().getEmail(),  // Column B: who added it
    "Some data here"               // Column C: the data
  ]);
  // ↳ Adds a new row at the bottom
  // ↳ Session.getActiveUser() returns the ACTUAL user running the script
  //   (works in both "Execute as" modes)
}
```

---

## 15. Ownership & Best Practices — What Happens When the Developer Leaves

This is a **critical concern** for company projects. If the developer who created the Apps Script project leaves the organization, **everything can break**.

### 15.1 What Breaks When the Owner's Account is Deactivated

```
┌─────────────────────────────────────────────────────────────────┐
│          DEVELOPER LEAVES → ACCOUNT DEACTIVATED                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❌ Apps Script project becomes inaccessible                    │
│     (owned by deactivated account)                              │
│                                                                 │
│  ❌ All deployments stop working                                │
│     (Web App URLs return errors)                                │
│                                                                 │
│  ❌ Triggers stop firing                                        │
│     (time-driven, onEdit, etc.)                                 │
│                                                                 │
│  ❌ Chat App stops responding                                   │
│     (GCP project may still exist, but script is dead)           │
│                                                                 │
│  ❌ Add-on stops working for all installed users                │
│                                                                 │
│  ⚠️ GCP project MAY survive                                    │
│     (if owned by the organization, not the individual)          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 15.2 Solutions — How to Prevent This

#### Solution 1: Use a Shared/Service Account as Owner (Best Practice ✅)

Create a shared Google account like `bot-admin@company.com` and use it as the **owner** of all critical Apps Script projects.

```
┌─────────────────────────────────────────────────────┐
│  bot-admin@company.com  ← OWNER of all projects    │
│                                                     │
│  ├── Apps Script Project: "Company Chat Bot"        │
│  ├── Apps Script Project: "HR Tool"                 │
│  └── Apps Script Project: "Sales Dashboard"         │
│                                                     │
│  Developers are added as EDITORS (not owners)       │
│  ├── ritesh@company.com (Editor)                    │
│  ├── priya@company.com (Editor)                     │
│  └── amit@company.com (Editor)                      │
│                                                     │
│  If any developer leaves → project continues        │
│  because bot-admin@company.com still exists          │
└─────────────────────────────────────────────────────┘
```

#### Solution 2: Transfer Ownership Before Leaving (Messy ⚠️)

The leaving developer can transfer ownership of the Apps Script project to another person. But this is messy:
- Deployment IDs change (all admin installs break)
- Triggers need to be re-created by the new owner
- GCP project linkage may need updating
- Easy to forget some projects

#### Solution 3: GCP Project Owned by Organization

The GCP project can be owned by the organization rather than an individual. This protects the GCP-side resources (APIs, service accounts, Chat API config), but does NOT protect the Apps Script code itself.

### 15.3 Best Practice Summary

| What | Best Practice |
|---|---|
| **Apps Script project owner** | Shared account (`bot-admin@company.com`) |
| **GCP project owner** | Organization-level or shared account |
| **Developers** | Added as Editors to both Apps Script and GCP |
| **Triggers** | Created by the shared account |
| **Deployments** | Done from the shared account |
| **Documentation** | Keep a list of all projects, their GCP links, and purposes |

> **Rule of thumb:** If a project is critical to the company, it should NEVER be owned by an individual employee's account. Always use a shared/service account.

---

## 16. Local Development with clasp

Instead of using the web-based editor, you can develop Apps Script locally in VS Code.

### Install clasp

```bash
npm install -g @google/clasp
```

### Login

```bash
clasp login
# Opens browser → sign in with your Google account
```

### Create a new project

```bash
clasp create --type standalone --title "My Company Tool"
# Creates a new Apps Script project and a local .clasp.json file
```

### Project structure

```
my-project/
├── .clasp.json          ← links to the remote Apps Script project
├── appsscript.json      ← manifest (scopes, triggers, etc.)
├── Code.js              ← your main code (renamed from .gs)
└── Utils.js             ← additional files
```

### Push code to Google

```bash
clasp push
# Uploads all local files to the Apps Script project
```

### Pull code from Google

```bash
clasp pull
# Downloads the latest code from the Apps Script project
```

### Open in browser

```bash
clasp open
# Opens the Apps Script editor in your browser
```

### Deploy

```bash
clasp deploy --description "v1.0"
# Creates a new deployment
```

### Workflow

```
VS Code (local)                    Google's Servers
──────────────                     ────────────────

Write code in .js files
        │
        ▼
  clasp push  ────────────────────→  Code uploaded
                                     to Apps Script
                                          │
  clasp deploy ───────────────────→  Deployment
                                     created
                                          │
                                     Users can now
                                     access the app
                                          │
  clasp pull  ←────────────────────  Pull latest
                                     changes
```

---

## 17. Quick Decision Guide

```
What do you need?
      │
      ├── Simple tool with a UI page?
      │         → Web App (doGet/doPost + HtmlService)
      │
      ├── Tool that works INSIDE Gmail/Sheets/Docs?
      │         → Google Workspace Add-on (CardService)
      │
      ├── Bot that responds in Google Chat?
      │         → Chat App (onMessage + GCP project)
      │
      ├── Automated background task (no user interaction)?
      │         → Time-driven trigger (like a cron job)
      │         → Project Settings → Triggers → Add Trigger
      │
      ├── All users should read/write the SAME data?
      │         → Shared Sheet + hardcoded Sheet ID
      │         → (most common company pattern)
      │
      ├── Company-only distribution?
      │         → Test Deployment + Admin install
      │         → OR Web App with "Anyone in org" access
      │
      └── Public distribution (for everyone)?
                → Google Workspace Marketplace
                → Requires GCP project + OAuth verification + Google review
```

---

## Summary — Key Takeaways

| # | Concept | Key Point |
|---|---|---|
| 1 | **Runtime** | Scripts run on Google's servers, NOT on the user's computer |
| 2 | **Execute as** | "Me" = owner's account for all users. "User" = each user's own account |
| 3 | **File location** | Created files go to root of Google Drive (of whoever script runs as) |
| 4 | **Add-ons** | Always run as the user. Use CardService for UI. Configure via manifest |
| 5 | **Consent screen** | Appears once per user. Shows permissions from oauthScopes in manifest |
| 6 | **Scopes** | Defined in `appsscript.json`. Control what the consent screen asks for |
| 7 | **Three deployment types** | Web App, Add-on, Chat App are completely different — different APIs, UIs, and distribution |
| 8 | **Test vs New Deployment** | Test = always latest code (for internal). New = frozen version (for URLs/Marketplace) |
| 9 | **Chat API requirement** | Google Chat API requires a Google Workspace account — does NOT work with personal Gmail |
| 10 | **Chat App config** | Uses **Deployment ID** (not Script ID) in GCP Chat API Configuration |
| 11 | **One project, many types** | A single project can be both a Chat App AND a Gmail Add-on simultaneously |
| 12 | **GCP auto-creation** | Web Apps/Add-ons get auto-created GCP. Chat Apps ALWAYS need manual GCP setup |
| 13 | **Verification** | Unverified apps show a warning. Company apps can be trusted by admin |
| 14 | **Company sharing** | Web App (URL) or Add-on (admin install) or Marketplace (public) |
| 15 | **Ownership** | Never let an individual employee own critical projects — use a shared account |
| 16 | **Shared data** | Use a shared Google Sheet with hardcoded ID — simplest company pattern |
| 17 | **Local dev** | Use `clasp` CLI to develop in VS Code with git version control |

---

> **Next steps:** Try creating a simple Web App first (easiest), then move to an Add-on (more polished). Use the shared Sheet pattern for company data.
