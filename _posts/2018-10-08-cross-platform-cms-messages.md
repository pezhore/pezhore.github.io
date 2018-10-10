---
layout: single
title:  "Cross-Platform Certificate Encrypted Strings"
date:   2018-04-16
categories: ['Encryption']
tag: ['Automation']
---

Password vaulting solves a very real problem: how do we leverage automation and avoid hardcoding passwords in source? We can use products like Hashicorp's Vault to provide RBAC, password expirations, and other advanced functionality that can plug into automation but this introduces a new problem - how do we protect that "first secret" that allows the newly built server to access and authenticate with the vault? We could set this password in plaintext/source - but that's the problem that vaulting attempts to solve. 

One potential solution is to leverage CMS messages. The [Cryptographic Message Syntax](https://en.wikipedia.org/wiki/Cryptographic_Message_Syntax) is a standardized method for signing, encrypting, decrypting, and validating any form of digital data using certificate-based key pairs.

A certificate must have extensions and usage that specifies document encryption in order decrypt/encrypt CMS messages. On Windows, this certificate resides in a certificate store on the local system - this certificate store is protected with a combination of ACL and the Windows Data Protection API. On Linux, the private key can be protected via folder and file permissions.

If the existing VM automation relies upon vRA, vRO, binary artifact repository and source repository - one potential usage for CMS messages to solve the "First Secret" problem is as follows:

* A new VM either has the certificate burned in to the template, or the certificate can be stored in vRO and injected into VMs at deployment time.
* The CMS message containing the "first secret" can be made available in source or binary repositories.
* At the opportune time, the VM will retrieve the CMS message, decrypt with the certificate and use this credential to auth with the desired password vaulting software.
* After the VM successfully authenticates and before the VM is handed off, vRO can forcibly remove the certificate from the VM to reduce exposure.

## Create the Certificate (Windows / PowerShell)

* Manually create certificate with an inf file and certreq:
  * Create certreq.inf file for cert request (example below). Most things can be changed, but **`szOID_ENHANCED_KEY_USAGE`**, **`szOID_DOCUMENT_ENCRYPTION`**, **`KeyUsage`**, and the **`[Extensions]`** fields should be left as is.
    ```
    [Version]
    Signature = "$Windows NT$"
    
    [Strings]
    szOID_ENHANCED_KEY_USAGE = "2.5.29.37"
    szOID_DOCUMENT_ENCRYPTION = "1.3.6.1.4.1.311.80.1"
    
    [NewRequest]
    Subject = "cn=certenc@mydomain.com"
    MachineKeySet = false
    KeyLength = 8192
    KeySpec = AT_KEYEXCHANGE
    HashAlgorithm = Sha1
    Exportable = true
    RequestType = Cert
    KeyUsage = "CERT_KEY_ENCIPHERMENT_KEY_USAGE | CERT_DATA_ENCIPHERMENT_KEY_USAGE"
    ValidityPeriod = "Years"
    ValidityPeriodUnits = "1"
    
    [Extensions]
    %szOID_ENHANCED_KEY_USAGE% = "{text}%szOID_DOCUMENT_ENCRYPTION%"
    ```
  * Create a new private/public key pair:
    
    `C:\> Certreq -new c:\path\to\certreq.inf output.cert`
* Alternatively, create it with powershell:
    ```language-powershell
    C:\> New-SelfSignedCertificate -Subject "CN=certenc@mydomain.com" -CertStoreLocation "Cert:\CurrentUser\My" -KeyUsage KeyEncipherment,DataEncipherment, KeyAgreement -Type DocumentEncryptionCert
    ```

## Export Certificate to Linux

* Export the certificate and private key pair with PowerShell
  ```language-powershell
  C:\> $MyPassword = ConvertTo-SecureString -String "exportpassword" -Force -AsPlainText
  C:\> (Get-ChildItem -path Cert:\CurrentUser\My\).Where({$_.Subject -eq "CN=certenc@mydomain.com"}) | Export-PfxCertificate -FilePath c:\temp\mypfx.pfx -Password $MyPassword
  ```
* On the Linux server, convert the pfx and remove the password
  * Convert the entire pfx to a pem file
    ```
    ~/> openssl pkcs12 -in mypfx.pfx -out all.pem
    Enter Import Password: [exportpassword]
    MAC verified OK
    Enter PEM pass phrase: [exportpassword]
    Verifying - Enter PEM pass phrase: [exportpassword]
    ```
  * Export key into rsa format
    ```
    ~/> openssl rsa -in all.pem -out rsa.key
    Enter pass phrase for all.pem: [exportpassword]
    writing RSA key
    ```
  * This will result in two files: `all.pem` and `rsa.key`.

## Encryption!

* Encrypt your secret:
  * Windows
    * Command:
      ```language-powershell
      C:\> Protect-CmsMessage -Content "SecretPassword1!" -To *certenc@mydomain.com
      -----BEGIN CMS-----
      MIIBuQYJKoZIhvcNAQcDoIIBqjCCAaYCAQAxggFRMIIBTQIBADA1MCExHzAdBgNVBAMMFmNlcnRl
      bmNAbWFzdGVyY2FyZC5jb20CEDtwj/MAVhipS6fXV2mKPRgwDQYJKoZIhvcNAQEHMAAEggEAqdy6
      BJ3F2zkCw0s1wwRXyfZ6LO5C/DldszJi9o5jodwh75pmbCkd+sOfU2XwIVSfbzYdgEuwpKqpoDFJ
      ykbuskBI9zqCXEDrWLSksRr9UfBdM4lToZR58IIhKhe7d6PU1YCvXcGM2zF2MzE1lPJHl8q9O24c
      EHxvz1OHaVcLnhZ+bxOfOhTbidiEr5DwTrL69gChWCAf4Yv6EUzacEsO08+A0kvWRBiwmtBSnprl
      6mJW4pgPZN6qgxSBJafswv5AS3X+9/bIyNoyNIwD3o5ZcEJ5CtZa2YGW6P1n1EMyOSeDndC8sxj1
      VWydE3aJ6aZwcI0P6Ifv38wjUNV/Wbs/HzBMBgkqhkiG9w0BBwEwHQYJYIZIAWUDBAEqBBAPgGSD
      tSj98L7jJLdlRpTxgCB6BfCAEwR8hQx9CyfbpfRTqgEFLsxHn76IgRWUvA5e8A==
      -----END CMS-----
      ```
    * The output can be captured to a plaintext file by adding the Out-File cmdlet: 
      ```language-powershell
      C:\> Protect-CmsMessage -Content "SecretPassword1!" -To *certenc@mydomain.com | Out-File -Path C:\Temp\encrypted.enc
      ```
  * Linux
    * Command:
      ```
      ~/> openssl cms -encrypt -from demo@localhost -to certenc@mydomain.com -subject "encrypted" -out encrypted.enc -outform PEM all.pem
      SecretPassword1!
      [ctrl]-[d]
      ```
    * This results in a mail.msg file like below
      ```
      ~/> cat encrypted.enc
      -----BEGIN CMS-----
      MIIBqAYJKoZIhvcNAQcDoIIBmTCCAZUCAQAxggFRMIIBTQIBADA1MCExHzAdBgNV
      BAMMFmNlcnRlbmNAbWFzdGVyY2FyZC5jb20CEDtwj/MAVhipS6fXV2mKPRgwDQYJ
      KoZIhvcNAQEBBQAEggEABPSASeNou8YBTjaEe3s+buvZwJuw7pGpLh3Hu2bsve+b
      +gTzoZuPAeLPR1LhAQVJiBXxSJUyGNemviehSgndOKYFWhojQnnncB/i/nin5ijk
      uBzuXgcFBEJDvFqtD8IsXGkLdwa2D7nuELpqd5rZXGE9GjMSFVtkK2gsczKE1Rza
      URVNGcWt94QgPQvx4xk1svp7wRgkBIOi+1nmXQ5r1kdS70gz6Z1pV9xanu5KhPyb
      fn8EwyzYoNiEsheFaSzstJCUOYiQckXjY/i/aSezrYM9PDPefCZqIIsTLQlAbi+c
      qhN5jPs4utMVpFDnT5oHG8ObsJQjGFKmpjbKRiWv8DA7BgkqhkiG9w0BBwEwFAYI
      KoZIhvcNAwcECEjEr1W2znlsgBgwF9+IDrrHEqStVkMkCMrO/PnpcM5dLIY=
      -----END CMS-----
      ```

## Decryption!

* Decrypt your secret
  * Windows
    * Obtain or create a file with the CMS message content
      ```language-powershell
      C:\> $Content = @"
      >> -----BEGIN CMS-----
      >> MIIBqAYJKoZIhvcNAQcDoIIBmTCCAZUCAQAxggFRMIIBTQIBADA1MCExHzAdBgNV
      >> BAMMFmNlcnRlbmNAbWFzdGVyY2FyZC5jb20CEDtwj/MAVhipS6fXV2mKPRgwDQYJ
      >> KoZIhvcNAQEBBQAEggEABPSASeNou8YBTjaEe3s+buvZwJuw7pGpLh3Hu2bsve+b
      >> +gTzoZuPAeLPR1LhAQVJiBXxSJUyGNemviehSgndOKYFWhojQnnncB/i/nin5ijk
      >> uBzuXgcFBEJDvFqtD8IsXGkLdwa2D7nuELpqd5rZXGE9GjMSFVtkK2gsczKE1Rza
      >> URVNGcWt94QgPQvx4xk1svp7wRgkBIOi+1nmXQ5r1kdS70gz6Z1pV9xanu5KhPyb
      >> fn8EwyzYoNiEsheFaSzstJCUOYiQckXjY/i/aSezrYM9PDPefCZqIIsTLQlAbi+c
      >> qhN5jPs4utMVpFDnT5oHG8ObsJQjGFKmpjbKRiWv8DA7BgkqhkiG9w0BBwEwFAYI
      >> KoZIhvcNAwcECEjEr1W2znlsgBgwF9+IDrrHEqStVkMkCMrO/PnpcM5dLIY=
      >> -----END CMS-----
      >> "@
      ```
    * Decrypt with the Unprotect-CmsMessage cmdlet
      ``language-powershell
      C:\> Unprotect-CmsMessage -Content $content
      SecretPassword1!
      ```
  * Linux
    * Obtain or create a file with the CMS message content
      ```
      ~/> echo "
      -----BEGIN CMS-----
      MIIBuQYJKoZIhvcNAQcDoIIBqjCCAaYCAQAxggFRMIIBTQIBADA1MCExHzAdBgNVBAMMFmNlcnRl
      bmNAbWFzdGVyY2FyZC5jb20CEDtwj/MAVhipS6fXV2mKPRgwDQYJKoZIhvcNAQEHMAAEggEAqdy6
      BJ3F2zkCw0s1wwRXyfZ6LO5C/DldszJi9o5jodwh75pmbCkd+sOfU2XwIVSfbzYdgEuwpKqpoDFJ
      ykbuskBI9zqCXEDrWLSksRr9UfBdM4lToZR58IIhKhe7d6PU1YCvXcGM2zF2MzE1lPJHl8q9O24c
      EHxvz1OHaVcLnhZ+bxOfOhTbidiEr5DwTrL69gChWCAf4Yv6EUzacEsO08+A0kvWRBiwmtBSnprl
      6mJW4pgPZN6qgxSBJafswv5AS3X+9/bIyNoyNIwD3o5ZcEJ5CtZa2YGW6P1n1EMyOSeDndC8sxj1
      VWydE3aJ6aZwcI0P6Ifv38wjUNV/Wbs/HzBMBgkqhkiG9w0BBwEwHQYJYIZIAWUDBAEqBBAPgGSD
      tSj98L7jJLdlRpTxgCB6BfCAEwR8hQx9CyfbpfRTqgEFLsxHn76IgRWUvA5e8A==
      -----END CMS-----
      " > encrypted.enc
      ```
    * Decrypt with the openssl cms executable
      ```
      ~/>  openssl cms -decrypt -in encrypted.enc -recip all.pem -inkey rsa.key -inform PEM
      SecretPassword1!%
      ```
    * Note that openssl seems to add a '`%`' to the end of the string.
* This string can now be used in scripts/credentials/other operations

