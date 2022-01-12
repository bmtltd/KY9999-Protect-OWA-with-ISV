# How to use IBM Security Verify to protect your on-premise Outlook Web Access portal

This how-to guide walks you through how to protect your Outlook Web Access (OWA) portal using IBM Security Verify (ISV).

## How it works

At the time of writting this, IBM Security Verify did not support WS-Fed, except if using WS-Fed with Office 365. As such, we need to use ADFS as a proxy so that it converts IBM Security Verify's SAML authentication over to WS-Fed via a `Claims Provider Trust`. The result is the following sign-in process:

``` mermaid
sequenceDiagram
  OWA->>ADFS: Auto-redirect
  ADFS->>ISV: Auto-redirect
  ISV-->>ADFS: After credentials and MFA
  ADFS-->>OWA: Jolly good. You're signed in.
```
Some IDaaS platforms get around this by having you install an agent on your Exchange server. What's nice about our approach is that it is less invasive. All we're doing is configuring Outlook Web Access to use ADFS, then configuring ISV as a `Claims Provider Trust` so that ISV handles the authentication, not ADFS. 