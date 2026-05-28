# AskSQLServer

Query any SQL Server database in plain English. No T-SQL required.

Powered by [Claude Code](https://claude.ai/code).

---

## Requirements

- Windows with PowerShell 5.1
- SQL Server instance accessible from your machine
- [Claude Code](https://claude.ai/code) installed (VS Code extension or CLI)
- `Invoke-Sqlcmd` -- comes with SSMS or the SqlServer PowerShell module

---

## Install

**Step 1 -- Clone this repo**

```
git clone https://github.com/SagheerDBA/AskSQLServer.git C:\Tools\AskSQLServer
```

You can clone to any folder you like. `C:\Tools\AskSQLServer` is just an example.

**Step 2 -- Open the cloned folder in VS Code**

```
File > Open Folder > C:\Tools\AskSQLServer
```

Claude Code will read `CLAUDE.md` automatically.

**Step 3 -- Type this in the Claude Code chat**

```
Setup AskSQLServer
```

Claude will ask where you want the working project to live, then walk you through
instance name, database, and authentication. It tests the connection, builds the
schema, and tells you when everything is ready.

**Step 4 -- Open the project folder Claude created**

```
File > Open Folder > [path you chose during setup]\AskSQLServer
```

You are ready. The cloned repo folder is no longer needed.

---

## Usage

Ask anything in plain English in the Claude Code chat:

```
Query: show me the 10 biggest tables by row count

Query: which stored procedures were modified in the last 7 days?

Query: how many rows does the Orders table have?

Query: show me all customers with no orders, as HTML
```

Adding "as HTML" saves a formatted report to your output folder and opens it in the browser.

---

## Schema Refresh

When your database schema changes, type this in Claude Code:

```
Refresh Schema
```

---

## Safety

Read-only. Claude will not execute DROP, DELETE, TRUNCATE, ALTER, or any statement
that modifies data or structure.

Windows Authentication by default. SQL Authentication supported -- password is
DPAPI-encrypted and stored locally, never in plain text.

---

## License

MIT
