# Get Started: Configure OWA, ADFS and IBM Security Verify

ADFS plays an important role in this integration because it proxies the SAML authentication in IBM Security Verify and converts it to the OWA-compatible WS-Fed. 

## Section #1: Configure OWA to use ADFS

For a complete guide, including how to provision ADFS [click here](https://docs.microsoft.com/en-us/exchange/clients/outlook-on-the-web/ad-fs-claims-based-auth?view=exchserver-2019) for an excellent Microsoft article that walks you through everything. 

If you already have ADFS deployed, you just need to create a `Relying Party Trust`. [Click here](https://docs.microsoft.com/en-us/exchange/clients/outlook-on-the-web/ad-fs-claims-based-auth?view=exchserver-2019#step-4-create-a-relying-party-trust-and-custom-claim-rules-in-ad-fs-for-outlook-on-the-web-and-the-eac) to be taken directly to do those steps.

!!! note

    The Web Application Proxy server is not required for this tutorial.


## Section #2: Create an Application in ISV

Next, we need to create an Application in IBM Security Verify.

1.  Go to the IBM Security Verify administration console.
2.  Go to **Applications** > **Applications**.
3.  Click the **Add application** button.
4.  Create a new application with the following settings:
    
    - **Sign-on method:** 
    
        SAML2.0

    - **Assertion consumer service URL (HTTP POST):** 
      ```
      https://{adfs_fqdn}/adfs/ls/idpinitiatedsignon
      ```
    - **Target URL:** 
      ```
      https://{owa_fqdn}/owa
      ```
    - **Name ID format:**

        Unspecified

    - **Name identifier:**

        Email

    - **Attribute Mappings** > Check "Send all known user attributes in the SAML Assertion".


5.  Assign the application to all users (assuming that's what you want).


## Section #3: Create a Claims Provider Trust in ADFS

We need to create a Claims Provider Trust in ADFS, which is essentially just an additional identity provider for ADFS. By default, ADFS wants to use Active Directory as its preferred identity provider. We need to configure ADFS to always default to using our newly-created Claims Provider Trust whenever a sign-in request for Outlook Web Access is received.

On the ADFS server:

1.  Open the `AD FS Management` application.
2.  Under `AD FS` > `Claims Provider Trust`, click `Add Claims Provider Trust`.
3.  Using the metadata file from the Application you created in ISV (in the prior section), create a new Claims Provider Trust named `ISVaaIDP`.
4.  Under `Edit Claim Rules` > `Add Rule` add the following rule:
    
    -   **Claim rule template:**

        Send Claims using Custom Rule

    -   **Claim rule name:**
        ```
        sAMAccountName to temp
        ```
    
    -   **Custom rule:**
        ```
        c:[Type == "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/nameidentifier", Properties["http://schemas.xmlsoap.org/ws/2005/05/identity/claimproperties/format"] == "urn:oasis:names:tc:SAML:1.1:nameid-format:unspecified"] 
        => issue(store = "Active Directory", types = ("claims:temp/attribute1"), query = "(&(objectCategory=person)(objectClass=user)(|(userPrincipalName={0})(mail={0})));sAMAccountName;contoso\ADFS_SERVICE_ACCOUNT", param = c.Value);
        ```
        Note: `contoso` should be replaced with your actual domain name. And `ADFS_SERVICE_ACCOUNT` should be replaced with the AD service account ADFS is running under.

5.  Under `Edit Claim Rules` > `Add Rule` add the following 2nd rule:
    
    -   **Claim rule template:**

        Send Claims using Custom Rule

    -   **Claim rule name:**
        ```
        temp to WindowsAccountName
        ```
    
    -   **Custom rule:**
        ```
        c:[Type == "claims:temp/attribute1"] 
        => issue(Type = "http://schemas.microsoft.com/ws/2008/06/identity/claims/windowsaccountname", Issuer = "AD AUTHORITY", OriginalIssuer = "https://ylu2.verify.ibm.com/saml/sps/saml20ip/saml20", Value = "contoso\" + c.Value);
        ```
        Note: `contoso` should be replaced with your actual domain name.


6.  Run the following PowerShell command to tell ADFS always to authenticate using ISV whenever people try to sign in to Outlook Web Access:
    ```powershell
    Set-AdfsRelyingPartyTrust -TargetName "Outlook on the web" -ClaimsProviderName @("ISVaaIDP")
    ```
7.  You are done!