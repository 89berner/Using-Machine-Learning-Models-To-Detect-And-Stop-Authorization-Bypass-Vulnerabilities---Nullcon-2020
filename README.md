# Rage Against The IDOR's: Using Machine Learning Models To Detect And Stop Authorization Bypass Vulnerabilities - Nullcon 2020

[![Nullcon 2020](https://img.shields.io/badge/Nullcon-2020-blue.svg)](https://nullcon.net/website/goa-2020/speakers/juan-berner.php)

**Speaker:** Juan Berner ([@89berner](https://twitter.com/89berner))
**Video Link:** [Watch the talk on YouTube](https://www.youtube.com/watch?v=KrSAqH8ZTB8) 

## High-Level Summary

This talk explores the persistent threat of Insecure Direct Object References (IDORs) and other authorization bypass vulnerabilities. Juan Berner discusses why traditional prevention methods are often insufficient and how Machine Learning (ML) models can be leveraged to detect and, crucially, stop these attacks by learning the expected behavior of applications and identifying deviations that indicate a potential breach. The presentation covers various ML approaches, data signal collection, anonymization techniques for sensitive data, and a practical demonstration of an ML-powered detection and blocking system.

## The Problem: IDORs and Authorization Bypasses

IDORs are a common and high-risk vulnerability where an attacker can access unauthorized data by manipulating direct object references (e.g., user IDs, document IDs) in requests.

*   **Prevalence:** IDORs are frequently reported and can lead to significant data breaches (evidenced by bug bounty reports).
*   **Low Exploitation Barrier:** Often require minimal technical skill to exploit.
*   **Beyond Basic IDORs:** The talk also covers:
    *   **Privilege Escalation:**
        *   Horizontal (accessing data of other users at the same privilege level - classic IDOR).
        *   Vertical (gaining higher privileges, e.g., accessing admin functions).
    *   **Avoiding Controls:** Bypassing security mechanisms like 2FA to perform unauthorized actions.
*   **Root Causes in Code:**
    *   Checks being ignored (e.g., due to an `ignore_auth` parameter).
    *   Checks being skipped due to flawed logic.
    *   Checks not existing at all (most dangerous).

## Limitations of Prevention & Traditional Detection

While prevention is key (random identifiers, framework-level access controls, SAST), it's not always enough due to complex codebases and human error.

Traditional detection methods often fall short:
*   **Enumeration/Rate Limiting:** Detects *attempts* but not necessarily successful exploitation, leading to many false positives.
*   **Hardcoded Rules:** Brittle and difficult to maintain with evolving applications.
*   **Alert Fatigue:** Security teams get overwhelmed by alerts that don't signify actual breaches.

The goal is to move from detecting the *possibility* of exploitation to detecting *actual* exploitation.

## The Machine Learning Approach

The core idea is to treat authorization as a classification problem. ML models can learn the normal context and behavior of requests and their corresponding authorization outcomes.

### 1. Core Idea:
*   **Training Phase:**
    `Request + Context + Actual Authorization Result -> [Train] -> Model`
*   **Prediction Phase:**
    `Request + Context -> [Model] -> Predicted Auth Result`
    `Compare (Predicted Auth Result, Actual Auth Result)`
    `If Mismatch -> Alert/Block`

### 2. Signal Collection (Context is Key):
To build effective models, rich context beyond basic access logs is needed. This involves:
*   **Enriched Request Logs:** Capturing detailed information about each request.
    *   **Authorization Results:** Explicit outcomes of authorization checks (e.g., `is_user_logged_in: success`, `is_username_current_user: failed`).
    *   **Backend Signals / Database Activity:** Number of DB operations (selects, inserts, updates, deletes), tables/columns accessed.
    *   **Response Information:** Response size, status code, (anonymized) content.
    *   **Performance Information:** Wall clock time.
    *   **Authentication Information:** User roles, permissions.
*   **Centralized Authorization Logic:** Ideally, having a centralized library, sidecar, or agent that performs authorization checks and logs this rich context. Challenges include standardizing complex business logic.

### 3. Handling Different Scenarios & Model Types:

*   **Scenario 1: Backend Signals Differ:**
    *   If a failed authorization results in fewer DB operations than a successful one, models like Random Forests or Gradient Boosting Machines (GBMs) can learn this pattern.
    *   **Limitation:** If data is pre-loaded (fetched *before* the auth check), backend signals might look identical for success and failure, making the model a "coin flip."

*   **Scenario 2: Response Content Differs (Backend Signals are Homogeneous):**
    *   When backend signals are unhelpful, the model needs to learn from the server response.
    *   **Anonymization is Crucial:**
        *   Cannot log raw PII/PCI data.
        *   Content might be compressed.
        *   Responses can vary due to legitimate reasons (e.g., translations, user customizations).
    *   **Techniques:**
        *   **Context Triggered Piecewise Hashing (CTPH / SSDEEP):** A fuzzy hashing algorithm. Small changes in input lead to small, predictable changes in the hash. Useful for comparing similarity of responses after stripping sensitive content (e.g., hashing HTML structure). Can be visualized as an "image" by mapping hash characters to colors.
        *   **Partial Hashing / Bag of Hashes:** Hash individual components of the response (e.g., JSON keys, HTML tags) and create a "bag of hashes" (similar to bag-of-words in NLP).
    *   **Model Types Explored:**
        *   Linear/Logistic Regression, SVMs: Did not perform well.
        *   Clustering (K-Means with custom kernel): Showed some promise.
        *   Convolutional Neural Networks (CNNs): Promising for the "image-like" SSDEEP hash representations.
        *   Gradient Boosting Machines (GBMs): Effective with "bag of hashes."

*   **Scenario 3: Learning Patterns of Access (No Explicit Auth Check Logged):**
    *   Even if an authorization check is missing entirely, the model can learn that certain data access patterns (e.g., querying a "message_for_friends" column) *should* be preceded by specific authorization checks.
    *   This involves building models per "authorization path" (the sequence of auth checks and their results leading to a data access).
    *   If a request accesses sensitive data without traversing an expected authorization path, it's an anomaly.

### 4. Building and Training the Models:
*   **Model Granularity:** Either a single large model or, more effectively, one model per "authorization path."
*   **Training Period:** Must be representative of normal traffic (e.g., consider user activity across different times/regions).
*   **Lifecycle:**
    1.  Start in **Alert** mode.
    2.  Transition to **Block** mode once confident.
    3.  Continuously monitor for **Anomalies** (deviations from learned normal behavior).
*   **Model Updates:** Retrain or update models when new application logic/versions are introduced (online training is a possibility).

### 5. Detecting & Blocking Attacks:
*   **Out-of-Band Detection:** Perform detection asynchronously to avoid adding latency to user requests.
*   **Avoiding Alert Fatigue:** Focus on confirmed exploitation rather than just attempts.
*   **Blocking Implementation:**
    *   **Inline:** The model makes a decision before the response is sent to the user (can add latency).
    *   **Hybrid:** A mix, potentially with faster checks inline and more complex ones out-of-band.
*   **Blocking Write Operations:** The same principles can be applied to detect and block unauthorized write operations by analyzing the context and expected authorization flow for writes.

## Demo Overview

The demo showcased a simple forum application with intentionally introduced IDOR vulnerabilities.
1.  **Initial State (Detection Mode):** An attacker (logged-in user) successfully accesses another user's profile by manipulating the username in the URL (IDOR). The system logs this, and the ML model (trained on 10k users, 100k interactions) identifies the discrepancy:
    *   The actual authorization check (`is_username_current_user`) failed.
    *   However, the backend signals (DB queries) and response content were consistent with a *successful* data retrieval.
    *   The model flags this as an anomaly.
2.  **Blocking Mode Activated:** The system is switched to block mode.
3.  **Attack Re-attempted:** The attacker tries the same IDOR. This time, the system, based on the model's prediction of what *should* have happened (a failed auth should not return full user data), blocks the request and returns a generic "blocked" message instead of the sensitive data.

## Key Takeaways

*   Authorization bypass vulnerabilities (especially IDORs) are a significant and ongoing threat.
*   Prevention alone is not sufficient; robust detection and blocking are necessary.
*   Machine Learning offers a scalable way to learn application behavior and detect anomalies indicative of authorization bypass.
*   Rich context (backend signals, response characteristics, authorization flow) is vital for training effective ML models.
*   Anonymization techniques (like SSDEEP and partial hashing) are essential when dealing with sensitive response data.
*   Different ML models (Random Forests, GBMs, CNNs) can be applied depending on the nature of the signals (structured backend data vs. unstructured/anonymized response data).
*   The system can learn not only to compare predicted vs. actual auth results but also to identify when an expected authorization check is missing entirely for a given data access pattern.

## About the Speaker

Juan Berner is a Principal Security Engineer @ Booking.com.
*   **Twitter:** [@89berner](https://twitter.com/89berner)
*   **Blog:** [medium.com/@89berner](https://medium.com/@89berner)
