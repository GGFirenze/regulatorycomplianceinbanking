# Chase UK Regulatory Compliance Demo
## User Guide: How the Dual-Project Amplitude Setup Works

---

## The Problem We're Solving

A bank has two teams with different data access needs:

| Team | What They Need |
|------|----------------|
| **Fraud Analysts** | Full PII (name, email, phone, etc.) to investigate fraud |
| **Everyone Else** | Only behavioral data (clicks, page views) - no PII |

**Regulatory requirement**: No tracking at all until the user consents.

---

## The Solution: Two Amplitude Projects

```
┌─────────────────────────────────────────────────────────────────┐
│                        USER LANDS ON PAGE                        │
│                                                                  │
│                    ┌──────────────────────┐                      │
│                    │  No SDKs Loaded      │                      │
│                    │  No Tracking         │                      │
│                    │  Form Data Local     │                      │
│                    └──────────────────────┘                      │
│                              │                                   │
│                              ▼                                   │
│                    ┌──────────────────────┐                      │
│                    │  User Fills Form     │                      │
│                    │  (still no tracking) │                      │
│                    └──────────────────────┘                      │
│                              │                                   │
│                              ▼                                   │
│                    ┌──────────────────────┐                      │
│                    │  User Clicks CONSENT │                      │
│                    └──────────────────────┘                      │
│                              │                                   │
│              ┌───────────────┴───────────────┐                   │
│              ▼                               ▼                   │
│   ┌─────────────────────┐       ┌─────────────────────┐          │
│   │    PROJECT A        │       │    PROJECT B        │          │
│   │   (Fraud Team)      │       │   (Everyone Else)   │          │
│   │                     │       │                     │          │
│   │ • Full PII sent     │       │ • Autocapture ON    │          │
│   │ • Autocapture OFF   │       │ • NO PII sent       │          │
│   │ • 1 event only      │       │ • All future events │          │
│   └─────────────────────┘       └─────────────────────┘          │
│              │                               │                   │
│              └───────────────┬───────────────┘                   │
│                              ▼                                   │
│                    ┌──────────────────────┐                      │
│                    │  Same Hashed User ID │                      │
│                    │  (for linking later) │                      │
│                    └──────────────────────┘                      │
└─────────────────────────────────────────────────────────────────┘
```

---

## Key Functions Explained

### 1. Form Submission Handler

```javascript
form.addEventListener('submit', async function(e) {
    e.preventDefault();  // Stop normal form submission
    
    // Collect form data (still local, not sent anywhere)
    const formData = {
        firstName: document.getElementById('firstName').value,
        email: document.getElementById('email').value,
        // ... other fields
    };
    
    // Only if consent checkbox is checked...
    if (consentCheckbox.checked) {
        // NOW we initialize Amplitude
        await initializeAmplitudeProjects(formData);
    }
});
```

**What this does**: Waits for the user to submit the form. Collects all data locally first. Only initializes Amplitude if consent is given.

---

### 2. Hash Email Function (Privacy)

```javascript
async function hashEmail(email) {
    const encoder = new TextEncoder();
    const data = encoder.encode(email.toLowerCase().trim());
    const hashBuffer = await crypto.subtle.digest('SHA-256', data);
    const hashArray = Array.from(new Uint8Array(hashBuffer));
    const hashHex = hashArray.map(b => b.toString(16).padStart(2, '0')).join('');
    return hashHex.substring(0, 16);  // First 16 characters
}
```

**What this does**: 
- Takes email like `john.smith@example.com`
- Converts to hash like `619d75b91fba3d7a`
- This becomes the User ID for both projects
- **Why?** Can't reverse-engineer the email from the hash, but same email always = same hash

---

### 3. Manual SDK Loading (Critical for GDPR)

```javascript
function loadManualSDK(id) {
    return new Promise((resolve, reject) => {
        // Load Analytics SDK (does NOT auto-initialize)
        const analyticsScript = document.createElement('script');
        analyticsScript.src = 'https://cdn.amplitude.com/libs/analytics-browser-2.11.2-min.js.gz';
        
        analyticsScript.onload = () => {
            // Then load Session Replay plugin
            const replayScript = document.createElement('script');
            replayScript.src = 'https://cdn.amplitude.com/libs/plugin-session-replay-browser-1.8.1-min.js.gz';
            replayScript.onload = () => resolve();
            document.head.appendChild(replayScript);
        };
        
        document.head.appendChild(analyticsScript);
    });
}
```

**What this does**:
- Loads SDK from `cdn.amplitude.com/libs/` (manual method)
- **NOT** from `cdn.amplitude.com/script/` (script loader - auto-initializes!)
- SDK is downloaded but **dormant** until we call `init()`

**Why manual loading?**

| Method | URL Pattern | Behavior |
|--------|-------------|----------|
| Script Loader | `cdn.amplitude.com/script/{API_KEY}.js` | Auto-initializes immediately ❌ |
| Manual Loading | `cdn.amplitude.com/libs/analytics-browser-X.X.X-min.js.gz` | Waits for `init()` call ✅ |

---

### 4. The Main Initialization Function

```javascript
async function initializeAmplitudeProjects(formData) {
    // STEP 1: Generate User ID FIRST (before any SDK init)
    const hashedEmail = await hashEmail(formData.email);
    console.log('🔑 Generated User ID:', hashedEmail);
    
    // STEP 2: Load the SDK (once, reused for both projects)
    await loadManualSDK('amplitude-sdk');
    
    // STEP 3: Create separate instance for Project A
    window.amplitudeA = window.amplitude.createInstance();
    
    // STEP 4: Initialize Project A WITH userId in config
    window.amplitudeA.init(PROJECT_A_API_KEY, hashedEmail, {
        autocapture: {
            attribution: false,
            pageViews: false,
            sessions: false,
            formInteractions: false,
            fileDownloads: false,
            elementInteractions: false  // ALL OFF
        }
    });
    
    // STEP 5: Send PII event to Project A
    window.amplitudeA.track('Account Application Submitted', {
        first_name: formData.firstName,
        email: formData.email,
        phone: formData.phone,
        // ... full PII
    });
    
    // STEP 6: Create separate instance for Project B
    window.amplitudeB = window.amplitude.createInstance();
    
    // STEP 7: Initialize Project B WITH same userId
    window.amplitudeB.init(PROJECT_B_API_KEY, hashedEmail, {
        autocapture: true  // ALL ON
    });
    
    // STEP 8: Set non-PII properties only
    window.amplitudeB.identify(new window.amplitude.Identify()
        .set('employment_status', formData.employmentStatus)
        .set('income_bracket', formData.annualIncome)
        // NO email, name, phone, etc.
    );
}
```

---

## The Critical Fix: Why User ID Must Be in `init()`

### The Bug (Before Fix)

```javascript
// ❌ WRONG - causes "Anonymous User"
window.amplitude.init(API_KEY, { autocapture: true });
window.amplitude.setUserId(hashedEmail);  // Too late! Autocapture already fired
```

**What happened**: Autocapture events fired immediately after `init()`, BEFORE `setUserId()` could execute.

### The Fix (After)

```javascript
// ✅ CORRECT - User ID from first event
window.amplitude.init(API_KEY, hashedEmail, { autocapture: true });
//                           ↑
//                    User ID as 2nd parameter
```

**What happens now**: User ID is set BEFORE autocapture starts, so all events have the correct User ID.

---

## Data Flow Summary

```
┌─────────────────────────────────────────────────────────────────┐
│                          CONSENT CLICK                           │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│  1. Hash email → "619d75b91fba3d7a"                              │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│  2. Load SDK (manual, no auto-init)                              │
└─────────────────────────────────────────────────────────────────┘
                                │
                ┌───────────────┴───────────────┐
                ▼                               ▼
┌───────────────────────────┐   ┌───────────────────────────┐
│  PROJECT A                │   │  PROJECT B                │
│                           │   │                           │
│  createInstance()         │   │  createInstance()         │
│  init(API_A, userId, {    │   │  init(API_B, userId, {    │
│    autocapture: OFF       │   │    autocapture: ON        │
│  })                       │   │  })                       │
│                           │   │                           │
│  track('Application       │   │  identify({               │
│    Submitted', {          │   │    employment: '...',     │
│    name: 'John Smith',    │   │    income: '...'          │
│    email: 'john@...',     │   │    // NO PII              │
│    phone: '+44...',       │   │  })                       │
│    // FULL PII            │   │                           │
│  })                       │   │  // Autocapture handles   │
│                           │   │  // all future events     │
└───────────────────────────┘   └───────────────────────────┘
                │                               │
                ▼                               ▼
┌───────────────────────────┐   ┌───────────────────────────┐
│  Fraud Analysts See:      │   │  Everyone Else Sees:      │
│                           │   │                           │
│  User: 619d75b91fba3d7a   │   │  User: 619d75b91fba3d7a   │
│  Name: John Smith         │   │  Employment: Employed     │
│  Email: john@example.com  │   │  Income: £50k-75k         │
│  Phone: +44 7700 123456   │   │  [Click] Boosted Savings  │
│  Event: App Submitted     │   │  [Click] 1% Cashback      │
└───────────────────────────┘   └───────────────────────────┘
                │                               │
                └───────────────┬───────────────┘
                                ▼
                ┌───────────────────────────┐
                │  Same User ID = Can link  │
                │  behavior to identity     │
                │  (with proper auth)       │
                └───────────────────────────┘
```

---

## Key Takeaways

1. **No tracking before consent** - SDK loads ONLY after user clicks consent
2. **Manual SDK loading** - Use `cdn.amplitude.com/libs/` not `cdn.amplitude.com/script/`
3. **User ID in init()** - Pass as 2nd parameter to ensure it's set from first event
4. **createInstance()** - Creates separate Amplitude instances for each project
5. **Same User ID** - Hashed email links both projects for authorized cross-referencing
6. **PII isolation** - Only Project A gets personal data, Project B gets behavior only

---

## Testing Checklist

- [ ] Fill form and submit without consent → Nothing tracked
- [ ] Fill form, check consent, submit → Both projects active
- [ ] Check Project A → Has full PII + "Account Application Submitted" event
- [ ] Check Project B → Has autocapture events + NO PII
- [ ] Compare User IDs → Same hash in both projects
- [ ] Click feature cards post-consent → Events appear in Project B with correct User ID

---

## File Location

`/Users/giuliano.giannini/Desktop/chase-compliance-demo.html`

## Test Server

```bash
cd ~/Desktop
python3 -m http.server 8888
# Open: http://localhost:8888/chase-compliance-demo.html
```

---

*Created: December 12, 2025*
*Author: Giuliano Giannini*

