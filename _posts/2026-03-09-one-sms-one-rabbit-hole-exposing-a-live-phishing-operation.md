---
title: "One SMS, One Rabbit Hole: Exposing a Live Phishing Operation with 5561 Victims"
date: 2026-03-09 17:00:00 +0100
categories: [Research, Phishing]
tags: [smishing, phishing, telegram-bot, fraud, osint, threat-analysis, phishing-kit, fraud-as-a-service]
image:
  path: /assets/img/2026-03-09-one-sms-one-rabbit-hole-exposing-a-live-phishing-operation/cover.jpg
  alt: Phishing SMS Analysis
---

# Case Study: Analysis of a Phishing Infrastructure Targeting Stolen Smartphones

---

## Introduction

A few months ago my smartphone was stolen.

Shortly after the incident, one of the emergency contacts associated with the device recovery feature received an SMS containing a link to what appeared to be a legitimate Apple account recovery page.

At first glance, this looked like a classic **targeted phishing attempt**: a fake page designed to trick the victim into entering credentials, OTP codes, and device unlock information.

However, after digging into the infrastructure behind that link, it became clear that this was **not just a simple phishing page**.

The website was part of a **larger phishing ecosystem** — multiple components working together to collect and process victim data at scale.

This write-up documents the investigation and technical analysis of that infrastructure.

For ethical and legal reasons:

- All **URLs and endpoints have been censored**
- **Tokens and credentials have been removed**
- **Victim information has been anonymized**

The goal of this report is **to raise awareness and document how phishing infrastructures operate**, not to enable attacks.

---

## Analysis Methodology

The investigation was structured into several progressive stages, starting from the initial phishing entry point and expanding outward as new components were discovered.

| # | Phase | Description |
|---|-------|-------------|
| 1 | **Initial Attack Vector** | Analysis of the phishing SMS and the fake Apple recovery page used to target the victim. |
| 2 | **Infrastructure Reconnaissance** | Traffic interception via Burp Suite to identify backend endpoints and application behavior. |
| 3 | **Endpoint Discovery** | Fuzzing and enumeration of publicly accessible files and directories exposed by the server. |
| 4 | **Exposed File Analysis** | Investigation of a debug file leaking session data, internal variables, and a Telegram bot token. |
| 5 | **Telegram Bot Analysis** | Token validation, bot interaction via the Telegram API, and real-time observation of the victim data exfiltration pipeline. |
| 6 | **File Manager Exposure** | Discovery and access to an exposed Tiny File Manager instance running with default credentials, revealing the full server structure. |
| 7 | **Administrative Panel Discovery** | Identification and access to the phishing campaign management dashboard using credentials intercepted from the Telegram bot. |
| 8 | **Campaign Intelligence** | Analysis of the panel's data, including operator activity, infrastructure scale, and total victim count. |

All sensitive information has been properly anonymized or removed throughout each section.

---

## 1. Initial Attack Vector

The investigation started after receiving a suspicious SMS spoofed as coming from Apple.

![Spoofed SMS](/assets/img/2026-03-09-one-sms-one-rabbit-hole-exposing-a-live-phishing-operation/0.png){: width="400" }
*Figure 1 — Spoofed SMS.*

The message contained a link pointing to a domain that clearly didn't belong to Apple — a textbook attempt to impersonate the account recovery process.

Under normal circumstances this would have been just another generic phishing message. But one thing stood out: my number had never been targeted before. So where did they get it?

**Where did the attackers obtain my phone number?**

Earlier that same day my smartphone had been stolen. After realizing this, I enabled **Lost Mode** and set up a recovery message with a contact number — which is exactly what ended up in the attacker's hands.

![Phishing landing page](/assets/img/2026-03-09-one-sms-one-rabbit-hole-exposing-a-live-phishing-operation/1.png)
*Figure 2 — Phishing landing page.*

Inspecting the phishing page confirmed it: the interface referenced the **exact iPhone model that had been stolen**, meaning the attackers were pulling device information from the recovery process to craft a convincing, targeted lure.

This wasn't a random campaign. It was **specifically built to target victims of stolen smartphones**.

---

## 2. Infrastructure Reconnaissance

To understand what was running behind the phishing page, traffic was intercepted with **Burp Suite** while simulating a victim interaction — specifically, submitting the OTP that the page requested.

After submitting, an AJAX request fired toward:

`/help/input/`

The intercepted request looked like this:

`type=passcode&id=138880&pass=123456`

Breaking down the parameters:

- `type=passcode` — the type of data being submitted. During testing, multiple form types appeared: OTP fields and email/password inputs for iCloud credentials.
- `id=138880` — a victim identifier correlating the submission to a specific record in the backend.
- `pass=123456` — the OTP entered on the phishing page.

![OTP submission request](/assets/img/2026-03-09-one-sms-one-rabbit-hole-exposing-a-live-phishing-operation/2.png)
*Figure 3 — OTP submission request.*

The server response also leaked references to the **CakePHP framework** and internal application paths — not directly exploitable here, but useful for the next phase: knowing the framework helps narrow down likely file structures and routes to fuzz.

---

## 3. Endpoint Discovery

With a clearer picture of the backend structure, the next step was to fuzz for exposed files and directories not referenced by the phishing page itself.

![Fuzzing results](/assets/img/2026-03-09-one-sms-one-rabbit-hole-exposing-a-live-phishing-operation/3.png)
*Figure 4 — Fuzzing results revealing additional server files.*

One result stood out immediately:

`test.php`

A file with a generic name like this is almost always a development artifact — something left behind during testing and never cleaned up before going live.

---

## 4. Analysis of `test.php`

Opening `test.php` revealed a raw **debug dump of the current application session** — almost certainly a `var_dump` left in the code and never removed.

![Debug output from test.php](/assets/img/2026-03-09-one-sms-one-rabbit-hole-exposing-a-live-phishing-operation/4.png)
*Figure 5 — Debug output exposed by `test.php`.*

The dump contained several internal session variables describing the active phishing campaign: `phished phone model`, `campaign creator ip`, and various backend references.

But one thing stood out immediately: a **Telegram Bot token** sitting in plain text inside the session data.

This token meant the infrastructure was using a Telegram bot as a **real-time exfiltration channel** — OTP codes, credentials, device details, all forwarded directly to an attacker-controlled chat. And now that token was exposed.

---

## 5. Telegram Bot Analysis

The discovery of a **Telegram bot token** within the exposed session
data represented a critical pivot point during the investigation.
Telegram bots are commonly abused in phishing infrastructures as
lightweight **exfiltration channels**, allowing attackers to receive
stolen data in real time without maintaining complex backend logging
systems.

Before proceeding with further analysis, the first step was to verify
whether the exposed token was still valid.

---

### Token Validation

The token was tested against the official **Telegram Bot API** using the
`getMe` endpoint.

    https://api.telegram.org/bot<TOKEN>/getMe

This endpoint returns basic metadata about the bot associated with the
provided token and is commonly used to verify the validity of bot
credentials.

![Telegram Bot API getMe response](/assets/img/2026-03-09-one-sms-one-rabbit-hole-exposing-a-live-phishing-operation/5.png)
*Figure 6 — Telegram Bot API `getMe` response (redacted).*

The API returned a valid response containing bot metadata, including:
- **Bot ID**
- **@username**
- **first_name**

All sensitive identifiers have been **redacted** for obvious reasons.

The successful response confirmed that:

- the token was **valid**
- the bot was **actively registered**
- the phishing infrastructure was **likely still operational**

---

### Why a Bot Token is Interesting

A Telegram bot token provides direct access to the **Telegram Bot API**,
which exposes several endpoints allowing interaction with the bot.

One of the most relevant endpoints for analysis is:

    /getUpdates

This endpoint retrieves **update objects**, which represent events
generated by bot interactions.

Each update may contain structured data such as:

- messages sent to the bot
- commands
- chat identifiers
- message content
- metadata associated with the interaction

In phishing infrastructures, bots are often used to **forward stolen
victim data directly to an attacker-controlled chat**.

This means that retrieving updates from the bot may expose the **entire
logging flow of the phishing campaign**.

---

### Intercepting Bot Updates with Telegraph

To analyze this behavior, the token was imported into **Telegraph**, a
third‑party Telegram client fork that allows extended interaction with
Telegram APIs.

Unlike the official Telegram client — which authenticates users
through phone numbers — Telegraph supports authentication using **bot
tokens**.

This effectively allows the bot to be treated as a **client session**,
capable of interacting with Telegram's infrastructure in a similar way
to a regular user.

Once authenticated, Telegraph continuously polls the Telegram API using
mechanisms equivalent to:

    https://api.telegram.org/bot<TOKEN>/getUpdates

The returned **JSON update objects** are then parsed and rendered inside
the client interface.

This process allowed observing the updates received by the bot in real
time, effectively positioning the analysis within the bot's **message
processing flow**.

---

### Observing the Logging Pipeline

During this phase, multiple updates were observed containing structured
messages generated by the phishing application.

These messages included victim‑submitted information such as:

- authentication codes (OTP)
- device information
- campaign identifiers
- other sensitive information related to the phishing session

This confirmed that the attackers were using the Telegram bot as a
**real‑time data exfiltration channel**, where information submitted
through the phishing pages was automatically forwarded to a Telegram
chat controlled by the operators.

![Victim data logged by the phishing bot](/assets/img/2026-03-09-one-sms-one-rabbit-hole-exposing-a-live-phishing-operation/6.png)
*Figure 7 — Victim data logged by the phishing bot (redacted).*

As shown in the evidence above, the intercepted updates contained **multiple categories of sensitive operational data**.

Among the observed information were:

- several **OTP submission attempts** from victims interacting with the phishing page
- evidence of **multiple operators** managing different phishing campaigns
- and, unexpectedly, what appeared to be **login credentials for an internal campaign management panel**, likely used by the attackers to administer the phishing infrastructure.

This last element proved particularly interesting, as it suggested the existence of a **dedicated backend interface used to manage phishing campaigns**, which will be further analyzed in the following sections.

---

## 6. File Manager Exposure & Misconfiguration

Going back to the files discovered during fuzzing, a few other things were worth examining more closely.

---

### Directory Listing Exposure

The server had no `.htaccess` restrictions on directory indexing, making it possible to **browse multiple directories directly** and enumerate their contents without any authentication.

![Directory listing exposed](/assets/img/2026-03-09-one-sms-one-rabbit-hole-exposing-a-live-phishing-operation/7.png)
*Figure 8 — Directory listing exposed due to missing `.htaccess` restrictions.*

---

### Tiny File Manager — Running on Default Credentials

Among the exposed files was an instance of **Tiny File Manager** — a lightweight, single-file PHP web file manager often used for quick server administration. It's also a common find in poorly secured phishing infrastructures.

![Tiny File Manager discovered](/assets/img/2026-03-09-one-sms-one-rabbit-hole-exposing-a-live-phishing-operation/8.png)
*Figure 9 — Tiny File Manager instance discovered within the phishing infrastructure.*

Tiny File Manager ships with publicly documented default credentials that most deployments never change. Testing them here worked immediately.

![Tiny File Manager login interface](/assets/img/2026-03-09-one-sms-one-rabbit-hole-exposing-a-live-phishing-operation/9.png)
*Figure 10 — Tiny File Manager login interface.*

![Tiny File Manager dashboard](/assets/img/2026-03-09-one-sms-one-rabbit-hole-exposing-a-live-phishing-operation/10.png)
*Figure 11 — Successful access to the Tiny File Manager dashboard.*

This gave full visibility over the server file structure: phishing page templates, backend scripts, configuration files, campaign resources — everything. The full internal layout of the kit was now accessible.

---

## 7. Administrative Panel Discovery & Campaign Intelligence

Reviewing the configuration files and backend scripts inside the file manager revealed several integrations used to run the campaigns:

- **SMS delivery APIs** and **Twilio** for automated outbound messaging
- Internal config parameters for the phishing backend
- Database references for campaign storage

More importantly, one of the config files contained a direct reference to a **dedicated campaign management panel**.

---

### Getting In

Accessing the endpoint brought up a login page for a web-based administration dashboard.

![Phishing campaign admin panel](/assets/img/2026-03-09-one-sms-one-rabbit-hole-exposing-a-live-phishing-operation/11.png)
*Figure 12 — Phishing campaign administration panel.*

The credentials intercepted earlier through the Telegram bot worked on the first try.

![Successful authentication to the campaign dashboard](/assets/img/2026-03-09-one-sms-one-rabbit-hole-exposing-a-live-phishing-operation/12.png)
*Figure 13 — Successful authentication to the campaign dashboard.*

The dashboard exposed multiple sections for managing phishing operations. One of the tabs contained a table with all phishing attempts logged under the operator's account.

![Operator-level phishing activity log](/assets/img/2026-03-09-one-sms-one-rabbit-hole-exposing-a-live-phishing-operation/13.png)
*Figure 14 — Logged phishing activity within the campaign dashboard.*

---

### Administrator Account Activity

After monitoring the Telegram bot for several more hours, a second credential set appeared — this one belonging to a **higher-privileged account**, likely the main operator running the infrastructure.

Logging in with those credentials revealed the full picture.

![Admin-level dashboard showing total victim count](/assets/img/2026-03-09-one-sms-one-rabbit-hole-exposing-a-live-phishing-operation/14.jpg)
*Figure 15 — Admin-level dashboard showing total victim count.*

The dashboard reported a total of:

**5,561 successfully targeted users**.

---

### Final Remarks

This entire investigation started from a single SMS.

One link. One fake page. Something that most people would either ignore or fall for without a second thought.

But behind that link sat a **fully operational phishing-as-a-service infrastructure**: automated SMS delivery, real-time Telegram exfiltration, a web-based campaign management panel, multiple active operators — and **5,561 confirmed victims**.

The goal of this analysis was purely demonstrative: to show what actually lives behind a phishing link when you start pulling the thread. Not just a fake login page, but an entire ecosystem engineered to scale fraud against real people.

What made this infrastructure unravel wasn't sophisticated exploitation. It was a cascade of basic, avoidable mistakes — a debug file left in production, a file manager running on default credentials, bot tokens leaking through unprotected session data. **Operational security failures, not technical limitations**, were what exposed the entire operation.

This is a recurring pattern in phishing infrastructure: the technical components are often well-built, but the operators misconfigure them. And those misconfigurations are all it takes.

> Everything documented in this report was analyzed strictly for **research and awareness purposes**. All sensitive information — URLs, credentials, victim data, bot tokens — has been anonymized or redacted. No systems were modified or disrupted during the investigation.

All relevant technical indicators and collected evidence have been **responsibly reported to the appropriate law enforcement authorities** to support further investigation and potential disruption of the operation.
