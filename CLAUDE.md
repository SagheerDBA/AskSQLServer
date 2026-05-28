# AskSQLServer
# CLAUDE.md -- Installer and Operating Brain

---

## What This Is

AskSQLServer lets you query any SQL Server database in plain English.
No T-SQL required. Works with any SQL Server database.

This file drives everything:
- Type `Setup AskSQLServer` to install and configure the tool
- After setup, open the created project folder in VS Code to start querying

---

## TRIGGER WORDS

| What user types | What Claude does |
|---|---|
| `Setup AskSQLServer` | Run the guided setup wizard |
| `Refresh Schema` | Rebuild schema from the live database (run from project folder) |
| `Query: <question>` or natural question | Translate to T-SQL, run it, return results (run from project folder) |

---

## SETUP WIZARD

When user types `Setup AskSQLServer`, run this sequence exactly.
Ask one question at a time. Wait for the answer before asking the next.

### Questions (in order)

**Q1 -- Project root**
```
Where do you want to create the AskSQLServer project?
Enter a folder path (e.g. C:\Projects\) and Claude will create AskSQLServer\ inside it.
```

**Q2 -- SQL Server instance**
```
What SQL Server instance do you want to connect to?
Example: MYSERVER or MYSERVER\SQLEXPRESS
```

**Q3 -- Database name**
```
What is the database name?
```

**Q4 -- Authentication**
```
Windows Authentication or SQL Authentication?
Type W for Windows (recommended) or S for SQL.
```

**Q5 -- SQL username (only if user chose S)**
```
What is the SQL login username?
```

**Q6 -- HTML output folder**
```
Where should HTML reports be saved?
Default is C:\Temp\AskSQLServer -- use this? Y or N
```
If N:
```
Enter the full path for HTML reports:
```

---

### Setup Actions (after all questions answered)

**Action 1 -- Check Invoke-Sqlcmd is available**

```powershell
if (-not (Get-Command Invoke-Sqlcmd -ErrorAction SilentlyContinue)) {
    Write-Host "Invoke-Sqlcmd not found." -ForegroundColor Red
}
```

If not found, stop and tell the user:
```
Invoke-Sqlcmd is not available. Install it by running:
  Install-Module -Name SqlServer -Scope CurrentUser
Or install SSMS which includes it automatically.
Setup cannot continue until Invoke-Sqlcmd is available.
```

**Action 2 -- Check if project already exists**

```powershell
Test-Path '[Q1 path]\AskSQLServer'
```

If it exists, ask:
```
A project already exists at [path]\AskSQLServer\. Overwrite it? Y or N
```
If N, stop.

**Action 3 -- Create folder structure**

```powershell
New-Item -ItemType Directory -Force -Path '[Q1]\AskSQLServer\config'
New-Item -ItemType Directory -Force -Path '[Q1]\AskSQLServer\scripts'
New-Item -ItemType Directory -Force -Path '[Q1]\AskSQLServer\schema'
New-Item -ItemType Directory -Force -Path '[Q1]\AskSQLServer\output'
```

**Action 4 -- Write all project files**

Write each file listed in the FILE CONTENTS section below to the project folder.

**Action 5 -- Write config\config.ps1**

```powershell
$SqlInstance     = '[Q2 value]'
$DatabaseName    = '[Q3 value]'
$SqlVersion      = 'auto'
$OutputPath      = '[Q6 value]'
$DefaultRowLimit = 1000
$AuthType        = 'Windows'   # or 'SQL'
$SqlUsername     = ''          # filled only for SQL auth
```

**Action 6 -- Save SQL credential (SQL auth only)**

If user chose SQL auth:
```powershell
$Cred = Get-Credential -UserName '[Q5 value]' -Message 'Enter SQL Server password for AskSQLServer'
$Cred | Export-Clixml -Path '[Q1]\AskSQLServer\config\sql-credential.xml'
```
Tell user: "A credential prompt will appear. Enter your SQL Server password."
The credential is DPAPI-encrypted. It never appears as plain text in any file.

**Action 7 -- Test connection and detect SQL Server version**

```powershell
Invoke-Sqlcmd -ServerInstance '[Q2]' -Database '[Q3]' -Query 'SELECT DB_NAME() AS ConnectedDB, @@VERSION AS SqlVersion' -ErrorAction Stop
```
For SQL auth, add: `-Username '[Q5]' -Password $Cred.GetNetworkCredential().Password`

- If FAILS: show the exact error. Tell user to check instance name, database name, firewall, and permissions. Do not continue until fixed.
- If SUCCEEDS: extract the SQL Server version year from @@VERSION (e.g. 2019) and update $SqlVersion in config.ps1. Confirm: "Connected successfully to [instance] -- [database]. SQL Server [version] detected."

**Action 8 -- Build schema**

```powershell
& '[Q1]\AskSQLServer\scripts\Initialize-Schema.ps1'
```

**Action 9 -- Create output folder**

```powershell
if (-not (Test-Path '[Q6 value]')) {
    New-Item -ItemType Directory -Path '[Q6 value]' | Out-Null
}
```

**Action 10 -- Confirm complete**

```
Setup complete.

Instance  : [Q2]
Database  : [Q3]
SQL Server: [detected version]
Schema    : [Q1]\AskSQLServer\schema\[DatabaseName]-schema.md
Reports   : [Q6]

To start querying:
  1. In VS Code: File > Open Folder > [Q1]\AskSQLServer\
  2. Open Claude Code chat
  3. Ask your first question. Example:
     Query: show me all tables and their row counts
```

---

## FILE CONTENTS

Write these files exactly as shown during setup Action 4.

---

### File: CLAUDE.md (project-level)
Path: `[Q1]\AskSQLServer\CLAUDE.md`

```
# AskSQLServer -- Project
# CLAUDE.md -- Operating Instructions

## Trigger Words

| What user types | What Claude does |
|---|---|
| Refresh Schema | Rebuild schema from the live database |
| Query: <question> or natural question | Translate to T-SQL, run, return results |

## How to Query

When the user asks a plain-English question:

1. Load config\config.ps1
2. Glob schema\ folder -- load the first .md file found
   If no schema file exists, tell user to run Refresh Schema first.
3. Generate T-SQL using the schema
4. Run: .\scripts\Run-Query.ps1 -Query "TSQL" -Description "short desc" -Format Table
   For HTML output (user says "as HTML"):
   Run: .\scripts\Run-Query.ps1 -Query "TSQL" -Description "short desc" -Format HTML
5. Show results. Always include the T-SQL used.

## T-SQL Rules

- Use TOP $DefaultRowLimit unless user asks for all rows
- Always list columns explicitly -- never SELECT *
- Use square brackets around names with spaces
- Use INFORMATION_SCHEMA for table/column metadata questions
- Use sys.indexes, sys.index_columns for index questions
- Use sys.objects for procedures, views, functions
- Match column and table names exactly as in schema
- Add ORDER BY when it helps readability
- If table or column not in schema, say so clearly -- do not guess

## Schema Refresh

When user types Refresh Schema:
1. Run: .\scripts\Initialize-Schema.ps1
2. Confirm: Schema refreshed. Found X tables, Y columns.
3. Tell user to reload Claude Code so the new schema is picked up.

## Safety Rules

- NEVER execute DROP, DELETE, TRUNCATE, ALTER, UPDATE, INSERT
- NEVER store or display passwords
- NEVER use SQL auth unless configured during setup
- If asked to modify data: refuse clearly
  "I only run read-only SELECT queries."
- Only query tables and columns in the schema file

## HTML Report Format

Header  : navy gradient (#1a3a5c to #2d5986), teal bottom border (#0891b2), white title + meta
Bar     : teal (#0891b2), white text, row count + database badge
Table   : gradient navy headers, alternating white/#f0f7ff rows, hover highlight (#fffbeb), NULLs as empty
T-SQL   : dark header label, dark code block (#1a202c) with light text
Footer  : centered, light grey, AskSQLServer | date
Fully self-contained HTML, plain ASCII only, no external dependencies.
Save to OutputPath from config. Open in browser after saving.
```

---

### File: scripts\Initialize-Schema.ps1
Path: `[Q1]\AskSQLServer\scripts\Initialize-Schema.ps1`

```powershell
<#
.SYNOPSIS
    Reads the target database schema and saves it as a local markdown file.
.DESCRIPTION
    Connects to the SQL Server instance defined in config\config.ps1,
    reads all tables, views, and columns from INFORMATION_SCHEMA, and saves
    the schema to schema\[DatabaseName]-schema.md for use by Claude Code.
    Read-only -- does not modify the database.
.NOTES
    Script File  : Initialize-Schema.ps1
    Version      : v1.0
    Build        : 001
    Status       : Production
    PowerShell   : 5.1 only
    Requires     : Invoke-Sqlcmd
    Author       : AskSQLServer
    Last Updated : 2026-05-28
#>
#Requires -Version 5.1
Set-StrictMode -Version Latest
$ErrorActionPreference = 'Stop'

$ConfigPath = Join-Path $PSScriptRoot '..\config\config.ps1'
if (-not (Test-Path $ConfigPath)) {
    Write-Error "config\config.ps1 not found. Run Setup AskSQLServer first."
    exit 1
}
. $ConfigPath

$SchemaDir  = Join-Path $PSScriptRoot '..\schema'
$SchemaFile = Join-Path $SchemaDir "${DatabaseName}-schema.md"

if (-not (Test-Path $SchemaDir)) { New-Item -ItemType Directory -Path $SchemaDir | Out-Null }

Write-Host "Connecting to ${SqlInstance} -- ${DatabaseName}" -ForegroundColor Cyan

$SqlParams = @{ ServerInstance = $SqlInstance; Database = $DatabaseName; ErrorAction = 'Stop' }

if ($AuthType -eq 'SQL') {
    $CredPath = Join-Path $PSScriptRoot '..\config\sql-credential.xml'
    if (Test-Path $CredPath) {
        $Cred = Import-Clixml -Path $CredPath
        $SqlParams['Username'] = $Cred.UserName
        $SqlParams['Password'] = $Cred.GetNetworkCredential().Password
    } else {
        $Cred = Get-Credential -UserName $SqlUsername -Message "SQL password for ${SqlInstance}"
        $SqlParams['Username'] = $Cred.UserName
        $SqlParams['Password'] = $Cred.GetNetworkCredential().Password
    }
}

$Tables  = Invoke-Sqlcmd @SqlParams -Query "SELECT TABLE_SCHEMA, TABLE_NAME, TABLE_TYPE FROM INFORMATION_SCHEMA.TABLES ORDER BY TABLE_SCHEMA, TABLE_NAME"
$Columns = Invoke-Sqlcmd @SqlParams -Query "SELECT TABLE_SCHEMA, TABLE_NAME, COLUMN_NAME, DATA_TYPE, CHARACTER_MAXIMUM_LENGTH, IS_NULLABLE, ORDINAL_POSITION FROM INFORMATION_SCHEMA.COLUMNS ORDER BY TABLE_SCHEMA, TABLE_NAME, ORDINAL_POSITION"

$ColumnMap = @{}
foreach ($Col in $Columns) {
    $Key = "$($Col.TABLE_SCHEMA).$($Col.TABLE_NAME)"
    if (-not $ColumnMap.ContainsKey($Key)) { $ColumnMap[$Key] = [System.Collections.Generic.List[object]]::new() }
    $ColumnMap[$Key].Add($Col)
}

$Lines = [System.Collections.Generic.List[string]]::new()
$Lines.Add("# Schema: ${DatabaseName}")
$Lines.Add("# Instance: ${SqlInstance}")
$Lines.Add("# Generated: $(Get-Date -Format 'yyyy-MM-dd HH:mm')")
$Lines.Add("")
$Lines.Add("---")
$Lines.Add("")
$Lines.Add("## Tables and Views")
$Lines.Add("")

foreach ($Table in $Tables) {
    $Schema = $Table.TABLE_SCHEMA; $TableName = $Table.TABLE_NAME; $Key = "${Schema}.${TableName}"
    $TypeLabel = if ($Table.TABLE_TYPE -eq 'VIEW') { ' (VIEW)' } else { '' }
    $Lines.Add("### [${Schema}].[${TableName}]${TypeLabel}")
    $Lines.Add("")
    if ($ColumnMap.ContainsKey($Key)) {
        $Lines.Add("| Column | Data Type | Nullable |")
        $Lines.Add("|--------|-----------|----------|")
        foreach ($Col in $ColumnMap[$Key]) {
            $dt = $Col.DATA_TYPE
            if ($Col.CHARACTER_MAXIMUM_LENGTH -ne [DBNull]::Value -and $Col.CHARACTER_MAXIMUM_LENGTH -gt 0) { $dt = "${dt}($($Col.CHARACTER_MAXIMUM_LENGTH))" }
            elseif ($Col.CHARACTER_MAXIMUM_LENGTH -eq -1) { $dt = "${dt}(MAX)" }
            $Lines.Add("| $($Col.COLUMN_NAME) | ${dt} | $(if ($Col.IS_NULLABLE -eq 'YES') {'YES'} else {'NO'}) |")
        }
    } else { $Lines.Add("(no columns found)") }
    $Lines.Add("")
}
$Lines.Add("---")
$Lines.Add("*End of schema*")
$Lines | Out-File -FilePath $SchemaFile -Encoding UTF8

Write-Host "Schema saved: ${SchemaFile}" -ForegroundColor Green
Write-Host "Tables/Views: $($Tables.Count) | Columns: $($Columns.Count)" -ForegroundColor Green
```

---

### File: scripts\Run-Query.ps1
Path: `[Q1]\AskSQLServer\scripts\Run-Query.ps1`

```powershell
<#
.SYNOPSIS
    Executes a T-SQL query and returns results as a table or HTML report.
.NOTES
    Script File  : Run-Query.ps1
    Version      : v1.0
    Build        : 001
    Status       : Production
    PowerShell   : 5.1 only
    Requires     : Invoke-Sqlcmd
    Author       : AskSQLServer
    Last Updated : 2026-05-28
#>
#Requires -Version 5.1
param(
    [Parameter(Mandatory=$true)][string]$Query,
    [Parameter(Mandatory=$false)][string]$Description = 'Query Result',
    [Parameter(Mandatory=$false)][ValidateSet('Table','HTML')][string]$Format = 'Table'
)
Set-StrictMode -Version Latest
$ErrorActionPreference = 'Stop'
Add-Type -AssemblyName System.Web

$ConfigPath = Join-Path $PSScriptRoot '..\config\config.ps1'
if (-not (Test-Path $ConfigPath)) { Write-Error "config\config.ps1 not found."; exit 1 }
. $ConfigPath

$SqlParams = @{ ServerInstance = $SqlInstance; Database = $DatabaseName; Query = $Query; ErrorAction = 'Stop' }
if ($AuthType -eq 'SQL') {
    $CredPath = Join-Path $PSScriptRoot '..\config\sql-credential.xml'
    $Cred = if (Test-Path $CredPath) { Import-Clixml -Path $CredPath } else { Get-Credential -UserName $SqlUsername -Message "SQL password for ${SqlInstance}" }
    $SqlParams['Username'] = $Cred.UserName
    $SqlParams['Password'] = $Cred.GetNetworkCredential().Password
}

Write-Host "Querying ${SqlInstance} -- ${DatabaseName}..." -ForegroundColor Cyan
$Results = Invoke-Sqlcmd @SqlParams
if ($null -eq $Results -or $Results.Count -eq 0) { Write-Host "No rows returned." -ForegroundColor Yellow; exit 0 }
$RowCount = $Results.Count

if ($Format -eq 'Table') { Write-Host ""; $Results | Format-Table -AutoSize; Write-Host "Rows: ${RowCount}" -ForegroundColor Cyan; exit 0 }

if (-not (Test-Path $OutputPath)) { New-Item -ItemType Directory -Path $OutputPath | Out-Null }
$Timestamp = Get-Date -Format 'yyyy-MM-dd HH:mm'
$DateStamp = Get-Date -Format 'yyyy-MM-dd'
$HtmlFile  = Join-Path $OutputPath (($Description -replace '[^A-Za-z0-9_-]','_') + '.html')
$ExcludeCols = @('RowError','RowState','Table','ItemArray','HasErrors')
$ColNames = $Results[0].Table.Columns | Where-Object { $ExcludeCols -notcontains $_.ColumnName } | Select-Object -ExpandProperty ColumnName
$HeaderCells  = ($ColNames | ForEach-Object { "<th>$_</th>" }) -join ''
$DataRowsHtml = ($Results | ForEach-Object { $Row=$_; "<tr>$(($ColNames | ForEach-Object { $v=$Row[$_]; if($v -is [DBNull] -or $null -eq $v){'<td></td>'}else{"<td>$([System.Web.HttpUtility]::HtmlEncode($v.ToString()))</td>"} }) -join '')</tr>" }) -join "`n"
$QEnc = [System.Web.HttpUtility]::HtmlEncode($Query)

@"
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>$Description</title>
<style>
  *{box-sizing:border-box}
  body{font-family:Segoe UI,Arial,sans-serif;font-size:13px;margin:0;padding:0;background:#eef2f7}
  .wrap{max-width:1200px;margin:0 auto}
  .hdr{background:linear-gradient(135deg,#1a3a5c 0%,#2d5986 100%);color:#fff;padding:20px 28px;border-bottom:4px solid #0891b2}
  .hdr h1{margin:0 0 6px;font-size:18px;font-weight:600;letter-spacing:0.3px}
  .hdr .meta{font-size:11px;opacity:0.8}
  .bar{background:#0891b2;color:#fff;padding:8px 28px;font-size:12px;font-weight:500}
  .badge{display:inline-block;background:rgba(255,255,255,0.25);color:#fff;font-size:10px;font-weight:600;padding:2px 8px;border-radius:12px;margin-left:8px}
  .body{padding:24px 28px}
  .card{background:#fff;border-radius:6px;box-shadow:0 2px 8px rgba(0,0,0,0.08);overflow:hidden;margin-bottom:24px}
  table{border-collapse:collapse;width:100%;font-size:12px}
  thead{background:linear-gradient(to right,#1a3a5c,#2d5986)}
  th{color:#fff;padding:10px 12px;text-align:left;font-weight:600;font-size:11px;text-transform:uppercase;letter-spacing:0.5px;border-right:1px solid rgba(255,255,255,0.1)}
  th:last-child{border-right:none}
  td{padding:8px 12px;border-bottom:1px solid #edf2f7;color:#2d3748}
  tr:nth-child(even) td{background:#f0f7ff}
  tr:hover td{background:#fffbeb}
  .tsql-wrap{background:#fff;border-radius:6px;box-shadow:0 2px 8px rgba(0,0,0,0.08);overflow:hidden}
  .tsql-label{background:#2d3748;color:#a0aec0;font-size:10px;font-weight:700;letter-spacing:1px;text-transform:uppercase;padding:8px 16px}
  pre{background:#1a202c;color:#e2e8f0;padding:16px;font-family:Consolas,Courier New,monospace;font-size:11px;white-space:pre-wrap;word-wrap:break-word;margin:0;line-height:1.6}
  .ftr{text-align:center;font-size:10px;color:#a0aec0;padding:16px}
</style>
</head>
<body>
<div class="wrap">
  <div class="hdr">
    <h1>$Description</h1>
    <div class="meta">Instance: $SqlInstance | Database: $DatabaseName | Run: $Timestamp</div>
  </div>
  <div class="bar">$RowCount rows returned <span class="badge">$DatabaseName</span></div>
  <div class="body">
    <div class="card">
      <table>
        <thead><tr>$HeaderCells</tr></thead>
        <tbody>
$DataRowsHtml
        </tbody>
      </table>
    </div>
    <div class="tsql-wrap">
      <div class="tsql-label">T-SQL Generated</div>
      <pre>$QEnc</pre>
    </div>
  </div>
  <div class="ftr">AskSQLServer | $DateStamp</div>
</div>
</body>
</html>
"@ | Out-File -FilePath $HtmlFile -Encoding UTF8

Write-Host "HTML saved: ${HtmlFile}" -ForegroundColor Green
Write-Host "Rows: ${RowCount}" -ForegroundColor Cyan
Start-Process $HtmlFile
```

---

### File: config\config.ps1.example
Path: `[Q1]\AskSQLServer\config\config.ps1.example`

```powershell
# AskSQLServer configuration -- reference copy
# The real config.ps1 is created automatically by Setup AskSQLServer.
$SqlInstance     = 'YOUR_INSTANCE_HERE'
$DatabaseName    = 'YOUR_DATABASE_HERE'
$SqlVersion      = 'auto'
$OutputPath      = 'C:\Temp\AskSQLServer'
$DefaultRowLimit = 1000
$AuthType        = 'Windows'
$SqlUsername     = ''
```

---

### File: .gitignore
Path: `[Q1]\AskSQLServer\.gitignore`

```
config/config.ps1
config/sql-credential.xml
schema/*.md
output/*.html
*.tmp
*.log
```

---

## QUERYING (after setup, from project folder)

See the project-level CLAUDE.md created during setup for full query instructions.
Quick reference:

- `Query: <question>` -- ask anything in plain English
- `Refresh Schema` -- rebuild schema after database changes
- Add "as HTML" to any question to get an HTML report
