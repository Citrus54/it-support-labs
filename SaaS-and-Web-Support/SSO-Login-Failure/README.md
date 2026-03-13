# SSO Login Failure (SAML Reply URL / ACS Mismatch)

## Ticket
**Issue:** User cannot sign in through SSO to the web application.

**Error:** SSO login fails after authentication and does not return properly to the application.

---

## Environment
- **Identity Provider (IdP):** Microsoft Entra ID
- **Service Provider (SP):** Microsoft Entra SAML Toolkit
- **Test User:** Maya
- **SSO Protocol:** SAML 2.0
- **Working SP Initiated Login URL:**  
  `https://samltoolkit.azurewebsites.net/SAML/Login/21195`
- **Working Assertion Consumer Service (ACS) URL:**  
  `https://samltoolkit.azurewebsites.net/SAML/Consume/21195`

---

## Skills Demonstrated
- SSO troubleshooting
- SAML authentication flow validation
- Reply URL / ACS troubleshooting
- Microsoft Entra Enterprise Application configuration
- User assignment verification
- Root cause isolation
- Incident documentation

---

## What SSO Is Doing in This Lab
Single Sign-On allows a user to authenticate with an identity provider and then gain access to a separate application without entering credentials again inside that app.

In this lab:

1. The user starts at the application.
2. The application redirects the user to Microsoft Entra ID.
3. The user authenticates with Entra.
4. Entra sends a SAML response back to the application’s **Assertion Consumer Service (ACS) URL**.
5. The application validates the assertion and creates a signed-in session.

If the ACS / Reply URL is wrong, the user may authenticate successfully with Microsoft but still fail to sign into the target app.

That is the exact failure being simulated here.

---

## Working State Verification

### 1. Confirm User Assignment
In Microsoft Entra:

**Enterprise applications → Microsoft Entra SAML Toolkit → Users and groups**

Verify that **Maya** is assigned to the application.

Expected result:
- Maya appears in the assigned users list

---

### 2. Confirm Basic SAML Configuration
Go to:

**Enterprise applications → Microsoft Entra SAML Toolkit → Single sign-on → SAML → Basic SAML Configuration**

Expected values:

**Identifier (Entity ID)**  
`https://samltoolkit.azurewebsites.net`

**Reply URL (ACS URL)**  
`https://samltoolkit.azurewebsites.net/SAML/Consume/21195`

**Sign-on URL**  
`https://samltoolkit.azurewebsites.net/SAML/Login/21195`

---

### 3. Confirm Live Login Works
Open a private/incognito browser window and browse to:

`https://samltoolkit.azurewebsites.net/SAML/Login/21195`

Expected flow:
- browser redirects to Microsoft sign-in
- user signs in as Maya
- browser returns to the SAML Toolkit application
- toolkit page loads successfully while logged in

This confirms working SSO.

---

## Breaking the Environment

### Break Scenario
Misconfigure the **Reply URL / ACS URL** in Entra.

Go to:

**Enterprise applications → Microsoft Entra SAML Toolkit → Single sign-on → Basic SAML Configuration → Edit**

Change:

**Reply URL**
from:
`https://samltoolkit.azurewebsites.net/SAML/Consume/21195`

to:
`https://samltoolkit.azurewebsites.net/SAML/Consume/99999`

Save the change.

The new path does not match the actual ACS endpoint expected by the service provider.

---

## What This Simulates
This simulates a real-world SSO misconfiguration where the identity provider is sending the SAML assertion to the wrong location.

Typical real-world results include:
- SAML response posted to the wrong endpoint
- application cannot consume the assertion
- SSO fails after successful Microsoft authentication
- login loop, redirect failure, or app error after sign-in


---

## Symptoms After Break

### Test the Broken State
Open a new private/incognito browser window and browse to:

`https://samltoolkit.azurewebsites.net/SAML/Login/21195`

### Expected Result
- user is redirected to Microsoft sign-in
- user successfully authenticates with Microsoft Entra ID
- return to the application fails or does not create a valid app session

Possible observed symptoms:
- failed redirect back to the application
- application error page
- login loop
- no valid session created after sign-in
- sign-in appears successful in Microsoft but not in the target app

### Important Distinction
Authentication to Microsoft Entra may still succeed.

The failure happens **after authentication**, during the handoff back to the service provider.


---

## Root Cause
The **Reply URL (ACS URL)** configured in Microsoft Entra does not match the actual ACS endpoint expected by the SAML Toolkit application.

Entra sends the SAML assertion to the configured Reply URL.  
If that URL is incorrect, the service provider cannot process the assertion and complete the login.

Result:
- identity provider login succeeds
- service provider login fails

---

## Resolution
Restore the correct ACS / Reply URL.

Go to:

**Enterprise applications → Microsoft Entra SAML Toolkit → Single sign-on → Basic SAML Configuration**

Set:

**Reply URL**  
`https://samltoolkit.azurewebsites.net/SAML/Consume/21195`

Save the change.

---

## Verification After Fix
Open a new private/incognito browser window and browse to:

`https://samltoolkit.azurewebsites.net/SAML/Login/21195`

Expected result:
- redirected to Microsoft sign-in
- sign in as Maya
- returned successfully to the SAML Toolkit application
- logged-in session is created successfully

This confirms the SSO flow has been restored.

---

## Evidence Collected

### Working State Screenshots
- Maya assigned to the application
- Working Basic SAML Configuration
- Successful launch from SP Initiated Login URL
- Successful post-login toolkit page

### Broken State Screenshots
- Reply URL changed to incorrect ACS path
- Microsoft sign-in page during failed test
- Failed return to application after authentication

### Fixed State Screenshots
- Reply URL restored
- Successful post-fix sign-in result

---

## Commands / Tools Used
This lab was browser and admin portal based rather than command-line based.

Tools used:
- Microsoft Entra admin center
- Enterprise Applications
- Single sign-on (SAML)
- Browser private/incognito session
- Live sign-in testing

---

## Key Concepts

### Identity Provider (IdP)
The system that authenticates the user.  
In this lab: **Microsoft Entra ID**

### Service Provider (SP)
The target application the user is trying to access.  
In this lab: **Microsoft Entra SAML Toolkit**

### SAML Assertion
The authentication response Entra sends back to the service provider after the user signs in.

### Entity ID / Identifier
Identifies the service provider to the identity provider.

### Reply URL / ACS URL
The endpoint where the identity provider sends the SAML assertion after authentication.

### Sign-on URL
The URL where the SSO flow begins from the service provider side.

### Real Support Lesson
If the user can authenticate to Microsoft but still cannot access the app, the issue may be:
- Reply URL / ACS mismatch
- Entity ID mismatch
- User not assigned to application
- Claims mismatch
- Certificate mismatch
