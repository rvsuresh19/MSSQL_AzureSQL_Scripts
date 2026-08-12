# Revoking dbcreator Without Removing db_owner

| Field | Value |
|---------|---------|
| Document Type | SOP / Quick Reference |
| Audience | Junior DBA / SQL Operations Team |
| Change Type | SQL Server Access Modification |
| Version | 1.0 |

> **Important Control**
>
> `dbcreator` is a server-level role. `db_owner` is a database-level role.
> Remove only the access specified in the approved request.
> Do not alter database role membership unless it is explicitly approved.

---

# 1. Purpose

To provide a clear, repeatable procedure for removing the `dbcreator` server role from a SQL Server login without accidentally removing database-level permissions such as `db_owner`.

---

# 2. Scope

This SOP applies to SQL Server instances where a login must be removed from the `dbcreator` server role while retaining existing database access. It is intended for operational DBA activities performed via an approved change or access-removal ticket.

---

# 3. Role Difference: Server-Level vs Database-Level

| Role | Level | Meaning / Impact |
|------|------|------|
| **dbcreator** | Server-level role | Allows the login to create, alter, drop, and restore databases at the SQL Server instance level. |
| **db_owner** | Database-level role | Provides full control within a specific database only. It is managed inside each database. |

---

# 4. Procedure

## 4.1 Pre-Change Validation

Before changing access, capture evidence of current server role and database role membership. Attach the results to the ticket as before-change evidence.

### A. Verify Current Server Role Membership for the Login

```sql
SELECT
    sp.name AS LoginName,
    sr.name AS ServerRole
FROM sys.server_role_members rm
JOIN sys.server_principals sr
    ON rm.role_principal_id = sr.principal_id
JOIN sys.server_principals sp
    ON rm.member_principal_id = sp.principal_id
WHERE sp.name = 'DOMAIN\UserName';
```

### B. Verify Current Database Role Membership in Each Impacted Database

```sql
USE [DatabaseName];
GO

SELECT
    DB_NAME() AS DatabaseName,
    dp.name AS UserName,
    dr.name AS DatabaseRole
FROM sys.database_role_members drm
JOIN sys.database_principals dr
    ON drm.role_principal_id = dr.principal_id
JOIN sys.database_principals dp
    ON drm.member_principal_id = dp.principal_id
WHERE dp.name = 'DOMAIN\UserName';
```

---

## 4.2 Correct Change: Remove Only dbcreator

Run the following command at the instance level. Replace `DOMAIN\UserName` with the actual login name.

```sql
ALTER SERVER ROLE [dbcreator]
DROP MEMBER [DOMAIN\UserName];
GO
```

For older SQL Server versions where `ALTER SERVER ROLE` is not supported:

```sql
EXEC sp_dropsrvrolemember
    @loginame = 'DOMAIN\UserName',
    @rolename = 'dbcreator';
GO
```

---

## 4.3 Commands That Must NOT Be Run Unless Approved

> **Do not remove database roles as part of dbcreator removal**
>
> The following statements remove database-level access and must not be executed unless the approved request specifically asks for `db_owner` removal from that database.

```sql
-- Do NOT run for dbcreator removal
ALTER ROLE [db_owner]
DROP MEMBER [DOMAIN\UserName];
GO

-- Legacy syntax (also do NOT run unless approved)
EXEC sp_droprolemember
    'db_owner',
    'DOMAIN\UserName';
GO
```

---

## 4.4 Post-Change Validation

Confirm that `dbcreator` has been removed and database-level access remains unchanged.

### A. Confirm dbcreator is Removed

```sql
SELECT
    sp.name AS LoginName,
    sr.name AS ServerRole
FROM sys.server_role_members rm
JOIN sys.server_principals sr
    ON rm.role_principal_id = sr.principal_id
JOIN sys.server_principals sp
    ON rm.member_principal_id = sp.principal_id
WHERE sp.name = 'DOMAIN\UserName'
  AND sr.name = 'dbcreator';
```

**Expected Result:** No rows returned.

### B. Confirm db_owner Remains Unchanged

```sql
USE [DatabaseName];
GO

SELECT
    DB_NAME() AS DatabaseName,
    dp.name AS UserName,
    dr.name AS DatabaseRole
FROM sys.database_role_members drm
JOIN sys.database_principals dr
    ON drm.role_principal_id = dr.principal_id
JOIN sys.database_principals dp
    ON drm.member_principal_id = dp.principal_id
WHERE dp.name = 'DOMAIN\UserName'
  AND dr.name = 'db_owner';
```

**Expected Result:** `db_owner` membership should still be present if it existed before the change and was not part of the approved removal request.

---

# 5. Recovery If db_owner Was Accidentally Removed

If database-level `db_owner` access was removed by mistake and the access is still approved, restore it in the impacted database only after confirming with the ticket/change owner.

```sql
USE [DatabaseName];
GO

ALTER ROLE [db_owner]
ADD MEMBER [DOMAIN\UserName];
GO
```

### Validate Restored Membership

```sql
USE [DatabaseName];
GO

SELECT
    DB_NAME() AS DatabaseName,
    dp.name AS UserName,
    dr.name AS DatabaseRole
FROM sys.database_role_members drm
JOIN sys.database_principals dr
    ON drm.role_principal_id = dr.principal_id
JOIN sys.database_principals dp
    ON drm.member_principal_id = dp.principal_id
WHERE dp.name = 'DOMAIN\UserName'
  AND dr.name = 'db_owner';
```

---

# 6. DBA Checklist

- Confirm the approved request specifies removal of `dbcreator` only.
- Capture current server role membership before the change.
- Capture current database role membership before the change.
- Execute only:

  ```sql
  ALTER SERVER ROLE [dbcreator] DROP MEMBER
  ```

- Do not run:

  ```sql
  ALTER ROLE [db_owner] DROP MEMBER
  ```

  unless specifically approved.

- Validate `dbcreator` removal after the change.
- Validate existing `db_owner` access remains unchanged.
- Attach before/after evidence to the ticket.
- Obtain peer review for privileged access changes where required.

---

# 7. Lesson Learned

> **Key Takeaway**
>
> Server-level and database-level permissions must be handled independently.
> Always verify the exact role requested for removal, record before-change evidence, and execute only the permission change that is approved.

---

*SOP - Revoking dbcreator Without Removing db_owner | Internal DBA Operations*