# Azure Database for PostgreSQL Flexible Server

# Connecting with Microsoft Entra Authentication Using pgAdmin and VS Code

## Overview

This document explains how to connect to **Azure Database for PostgreSQL Flexible Server** using **Microsoft Entra Authentication** from client tools such as **pgAdmin** and **Visual Studio Code (PostgreSQL Extension)**.

The authentication approach differs slightly depending on whether:

1. An individual **Entra User** is configured as an administrator.
2. An **Entra Group** is configured as an administrator.

Understanding this distinction is critical, especially when connecting through tools such as pgAdmin that require manual token-based authentication.

---

## Prerequisites

Before proceeding, ensure:

- Microsoft Entra Authentication is enabled for the PostgreSQL Flexible Server.
- Your user account is a member of the configured Entra administrator account or group.
- Azure PowerShell (`Az` module) or Azure CLI is installed.
- You have network connectivity to the PostgreSQL server.
- pgAdmin or VS Code PostgreSQL extension is installed.

---

## Authentication Scenarios

### Scenario 1: Entra User Configured as PostgreSQL Administrator

**Administrator configured in Azure**

```text
abc@abc.com
```

#### Login Behavior

```text
Username : abc@abc.com
Password : Access Token generated for abc@abc.com
```

#### Authentication Flow

```text
User (abc@abc.com)
        |
        v
Generate Access Token
        |
        v
pgAdmin Login
Username = abc@abc.com
Password = Access Token
        |
        v
Azure PostgreSQL
```

#### Steps

##### Step 1: Sign in to Azure

```powershell
Connect-AzAccount
```

##### Step 2: Verify Active Account

```powershell
Get-AzContext
```

##### Step 3: Generate Access Token

```powershell
$tokenSecure = (Get-AzAccessToken -ResourceUrl "https://ossrdbms-aad.database.windows.net").Token

$BSTR = [System.Runtime.InteropServices.Marshal]::SecureStringToBSTR($tokenSecure)

$token = [System.Runtime.InteropServices.Marshal]::PtrToStringAuto($BSTR)

$token
```

##### Step 4: Configure pgAdmin

```text
Username : abc@abc.com
Password : <Access Token>
```

The connection should be established successfully.

---

### Scenario 2: Entra Group Configured as PostgreSQL Administrator

**Administrator configured in Azure**

```text
PG-DBA-Admins
```

**User**

```text
abc@abc.com
```

**Member Of**

```text
PG-DBA-Admins
```

#### Important Difference

When an **Entra Group** is configured as PostgreSQL Administrator:

- The access token is still generated using the individual user account.
- The username supplied during login must be the **Entra Group Name**, not the individual user account.

##### Correct Login Values

```text
Username : PG-DBA-Admins
Password : Access Token generated for abc@abc.com
```

#### Authentication Flow

```text
User (abc@abc.com)
        |
        v
Generate Access Token
        |
        v
pgAdmin Login
Username = PG-DBA-Admins
Password = User Access Token
        |
        v
Azure validates:
 - User token
 - User membership in group
        |
        v
Access Granted
```

#### Common Mistake

Using:

```text
Username : abc@abc.com
Password : <Access Token>
```

When the PostgreSQL administrator is configured as:

```text
PG-DBA-Admins
```

This can result in authentication failure.

#### Correct Configuration

```text
Username : PG-DBA-Admins
Password : <Access Token generated for abc@abc.com>
```

---

## Generating Access Token Using Azure PowerShell

### Full Procedure

```powershell
# Authenticate to Azure
Connect-AzAccount

# Verify active context
Get-AzContext

# Obtain PostgreSQL access token
$tokenSecure = (Get-AzAccessToken -ResourceUrl "https://ossrdbms-aad.database.windows.net").Token

# Convert SecureString to plain text
$BSTR = [System.Runtime.InteropServices.Marshal]::SecureStringToBSTR($tokenSecure)

$token = [System.Runtime.InteropServices.Marshal]::PtrToStringAuto($BSTR)

# Display token
$token
```

Copy the generated token and use it as the password in pgAdmin.

---

## Generating Access Token Using Azure CLI

### Authenticate

```bash
az login
```

### Generate Token

```bash
az account get-access-token \
  --resource-type oss-rdbms \
  --query accessToken \
  -o tsv
```

Alternatively:

```bash
az account get-access-token \
  --resource https://ossrdbms-aad.database.windows.net \
  --query accessToken \
  -o tsv
```

Copy the token output and use it as the password in pgAdmin.

---

## Connecting Using pgAdmin

### General Tab

```text
Name : Azure PostgreSQL
```

### Connection Tab

```text
Host Name : <server-name>.postgres.database.azure.com

Port      : 5432

Database  : postgres
           (or target database)

Username  :
  - Entra User Name
    OR
  - Entra Group Name

Password  :
  Access Token
```

### SSL Tab

```text
SSL Mode : Require
```

---

## Connecting Using Visual Studio Code

The PostgreSQL extension for VS Code provides a more user-friendly authentication experience.

### Steps

1. Install the PostgreSQL extension.
2. Create a new PostgreSQL connection.
3. Select:

```text
Authentication Type = Microsoft Entra ID
```

4. A browser authentication window will appear.
5. Sign in using your Entra account.
6. Complete MFA if prompted.
7. The extension automatically retrieves and manages the access token.

### Key Advantages

- No manual token generation required.
- No need to paste access tokens into the password field.
- Supports interactive browser-based sign-in.
- Handles token refresh automatically.

---

## Troubleshooting

### Password Authentication Failed

Verify the following:

- User is configured as PostgreSQL administrator, or
- User is a member of the configured Entra administrator group.
- The correct username is being used.
- A fresh access token is supplied.

---

### Access Token Expired

Access tokens have a limited lifetime.

Generate a new token and reconnect.

---

### Login Works for Direct User but Fails for Group

Verify:

- User is a member of the Entra group.
- Entra group is configured as PostgreSQL administrator.
- Group name is specified in the Username field.
- Access token belongs to a valid member of the group.

---

## Quick Reference

### Case 1: Entra User as Administrator

```text
Administrator = abc@abc.com

Username = abc@abc.com
Password = User Access Token
```

### Case 2: Entra Group as Administrator

```text
Administrator = PG-DBA-Admins

Username = PG-DBA-Admins
Password = Access Token generated for abc@abc.com
```

---

## Key Takeaway

When an **individual Entra user** is configured as the PostgreSQL administrator:

```text
Username = User UPN
Password = User Access Token
```

When an **Entra Group** is configured as the PostgreSQL administrator:

```text
Username = Entra Group Name
Password = Access Token generated for the individual user
```

This distinction is the most common reason for Microsoft Entra authentication failures in pgAdmin and other PostgreSQL client utilities.