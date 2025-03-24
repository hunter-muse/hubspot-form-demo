# HubSpot Forms API Integration Handoff for ASGE Dev Team

### Overview

This document outlines how to integrate a custom form with the HubSpot Forms API, including how to capture and submit the `hubspotutk` tracking cookie. This integration is confirmed to work and designed to preserve full attribution and session tracking inside HubSpot, even with a non-HubSpot (ASP.NET) form.

### Why This Approach Matters

Using the **HubSpot Forms API** ensures that:

- Form submissions are correctly tracked as contacts in HubSpot
- HubSpot can associate the form fill with prior page views, sessions, and traffic source (via `hubspotutk`)
- You have full control over how and when data is sent

The alternative (HubSpot's "collected forms" auto-detection) often fails to track attribution properly and can result in incomplete data.

---

### Key Components

#### 1. HubSpot Tracking Script

Include this on any page where the form appears:

```html
<script type="text/javascript" id="hs-script-loader" async defer src="//js.hs-scripts.com/YOUR_PORTAL_ID.js"></script> <!-- Replace YOUR_PORTAL_ID with your actual HubSpot Portal ID -->
```

This sets the `hubspotutk` cookie in the user's browser.

#### 2. Create a Form in HubSpot (or get the required variables in a pre-existing form)

Before integrating with the Forms API, you must create a form in HubSpot (if you haven't already):

1. Log in to HubSpot and go to **Marketing → Forms**
2. Click **Create form**
3. Choose **Standalone Form**
4. Add the fields that match your custom form (e.g., First Name, Last Name, Email, Subscription Preferences)
5. Publish the form
6. Click **Actions → Share → Embed Code**
7. Extract the **Portal ID** and **Form GUID** from the embed code:

```javascript
hbspt.forms.create({
  portalId: "YOUR_PORTAL_ID",
  formId: "YOUR_FORM_GUID"
});
```

Replace the placeholders in the code examples below with your actual Portal ID and Form GUID.

#### 3. Capture `hubspotutk` (client-side only)

Because this cookie is set client-side, it must be accessed with JavaScript:

```javascript
function getCookie(name) {
  const match = document.cookie.match(new RegExp('(^| )' + name + '=([^;]+)'));
  return match ? match[2] : null;
}
```

#### 4. Integration Pattern for ASP.NET

You have two options:

**Option A - Client-side Submission (Simplest):**
Use JavaScript to send the data (including the cookie) directly to HubSpot:

```javascript
document.querySelector('button').addEventListener('click', async function () {
  const form = document.getElementById('custom-form');
  const hutk = getCookie('hubspotutk');

  const data = {
    fields: [
      { name: 'firstname', value: form.firstname.value },
      { name: 'lastname', value: form.lastname.value },
      { name: 'email', value: form.email.value },
      { name: 'subscription_preferences', value: form.subscription_preferences.value }
    ],
    context: {
      hutk: hutk,
      pageUri: window.location.href,
      pageName: document.title
    }
  };

  const response = await fetch("https://api.hsforms.com/submissions/v3/integration/submit/YOUR_PORTAL_ID/YOUR_FORM_GUID", {
    method: 'POST',
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(data)
  });

  const result = await response.json();
  console.log('HubSpot response:', result);
  alert("Form submitted!");
});
```

**Option B - Server-side Submission from ASP.NET:**

1. Add a hidden input field:

```html
<input type="hidden" name="hutk" id="hutk" />
```

2. Populate it using JS:

```javascript
document.getElementById('hutk').value = getCookie('hubspotutk');
```

3. In your ASP.NET controller, read it from the form POST and submit to HubSpot using C#:

```csharp
var httpClient = new HttpClient();
var formData = new {
  fields = new[] {
    new { name = "firstname", value = Request["firstname"] },
    new { name = "lastname", value = Request["lastname"] },
    new { name = "email", value = Request["email"] },
    new { name = "subscription_preferences", value = Request["subscription_preferences"] }
  },
  context = new {
    hutk = Request["hutk"],
    pageUri = Request.Url.AbsoluteUri,
    pageName = "Your Page Name Here"
  }
};

var content = new StringContent(JsonConvert.SerializeObject(formData), Encoding.UTF8, "application/json");
var result = await httpClient.PostAsync("https://api.hsforms.com/submissions/v3/integration/submit/YOUR_PORTAL_ID/YOUR_FORM_GUID", content);
```

---

### Example Resources

- HubSpot Forms API Docs: [https://legacydocs.hubspot.com/docs/methods/forms/submit_form](https://legacydocs.hubspot.com/docs/methods/forms/submit_form)
- Tracking Cookie (`hubspotutk`): [https://knowledge.hubspot.com/reports/what-is-the-hubspotutk-cookie](https://knowledge.hubspot.com/reports/what-is-the-hubspotutk-cookie)
- HubSpot Developer Community: [https://developers.hubspot.com/](https://developers.hubspot.com/)

---

### Final Notes

- This integration is **live-tested and verified**
- Current spam flags are due to test domain (vercel.app) not being whitelisted — this will not happen on a production domain tied to your HubSpot account
- `hubspotutk` is critical to accurate attribution in HubSpot, and the Forms API allows us to capture it even with ASP.NET forms

Let us know if you'd like a working test version or additional implementation support!
