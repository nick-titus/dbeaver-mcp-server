# SQL Server Support Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add SQL Server query execution support to dbeaver-mcp-server for AWS RDS databases with read-only analytics queries.

**Architecture:** Follow PostgreSQL implementation pattern. Use `mssql` npm package for connection and query execution. Extract connection properties from DBeaver config, configure AWS RDS SSL defaults, execute queries, return results in standard QueryResult format.

**Tech Stack:** TypeScript, mssql npm package (10.0.0), Node.js 18+

---

## Prerequisites Verification

Before starting implementation, verify:
1. DBeaver has working SQL Server connection
2. Connection credentials saved in DBeaver
3. Design document exists at `docs/plans/2025-10-29-sql-server-support-design.md`

---

## Task 1: Install SQL Server Driver Dependencies

**Files:**
- Modify: `package.json`
- Modify: `package-lock.json` (auto-generated)

**Context:** The mssql package provides Microsoft SQL Server connectivity for Node.js. Version 10.0.0 is the latest stable release with TypeScript support.

**Step 1: Install mssql runtime package**

Run:
```bash
npm install mssql@^10.0.0 --save
```

Expected output:
```
added 1 package, and audited 187 packages in 2s
```

**Step 2: Install mssql TypeScript types**

Run:
```bash
npm install @types/mssql@^9.1.0 --save-dev
```

Expected output:
```
added 1 package, and audited 188 packages in 1s
```

**Step 3: Verify dependencies added**

Run:
```bash
grep mssql package.json
```

Expected output showing both lines:
```json
"mssql": "^10.0.0"
```
and in devDependencies:
```json
"@types/mssql": "^9.1.0"
```

**Step 4: Commit dependency changes**

```bash
git add package.json package-lock.json
git commit -m "build: add mssql driver dependencies

Install mssql@10.0.0 for SQL Server connectivity.
Install @types/mssql@9.1.0 for TypeScript support.

Supports AWS RDS SQL Server connections with SSL.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 2: Add Import Statement for SQL Server Driver

**Files:**
- Modify: `src/dbeaver-client.ts:6`

**Context:** Import statement must be placed with other database driver imports near top of file. Current imports include `pg` for PostgreSQL at line 6.

**Step 1: Locate existing imports**

Run:
```bash
head -20 src/dbeaver-client.ts | grep -n "import"
```

Expected: Shows imports from lines 1-6, including `import { Client } from 'pg';`

**Step 2: Add mssql import after pg import**

Edit `src/dbeaver-client.ts` line 6, add after the `pg` import:

```typescript
import { Client } from 'pg';
import sql from 'mssql';
```

Complete import section should look like:
```typescript
import { spawn, ChildProcess } from 'child_process';
import fs from 'fs';
import path from 'path';
import os from 'os';
import csv from 'csv-parser';
import { Client } from 'pg';
import sql from 'mssql';
import {
  DBeaverConnection,
  QueryResult,
  SchemaInfo,
  ExportOptions,
  ConnectionTest,
  DatabaseStats
} from './types.js';
```

**Step 3: Verify TypeScript compilation**

Run:
```bash
npm run build
```

Expected output:
```
> dbeaver-mcp-server@1.2.3 build
> tsc && echo '#!/usr/bin/env node' > dist/index.js.tmp...
```

No TypeScript errors should appear.

**Step 4: Commit import addition**

```bash
git add src/dbeaver-client.ts
git commit -m "feat(sql-server): add mssql driver import

Import sql from mssql package for SQL Server connectivity.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 3: Implement executeMSSQLQuery Method

**Files:**
- Modify: `src/dbeaver-client.ts:224` (insert before `executeDBeaver` method)

**Context:** This method extracts connection properties from DBeaver config, configures AWS RDS-specific SSL options, executes the query, and returns results in the standard QueryResult format. It follows the exact pattern established by `executePostgreSQLQuery` at lines 170-224.

**Step 1: Locate insertion point**

Run:
```bash
grep -n "private async executeDBeaver" src/dbeaver-client.ts
```

Expected: Shows method starting around line 226

We'll insert the new method just before this line.

**Step 2: Write executeMSSQLQuery method**

Insert at line 224 (before `executeDBeaver`):

```typescript
  private async executeMSSQLQuery(connection: DBeaverConnection, query: string): Promise<QueryResult> {
    // Extract connection properties with fallbacks
    const host = connection.host || connection.properties?.host || 'localhost';
    const port = connection.port || (connection.properties?.port ? parseInt(connection.properties.port) : 1433);
    const database = connection.database || connection.properties?.database;
    const user = connection.user || connection.properties?.user;
    const password = connection.properties?.password;

    // Validate required properties
    if (!database) {
      throw new Error('SQL Server database name is required');
    }
    if (!user) {
      throw new Error('SQL Server username is required');
    }
    if (!password) {
      throw new Error('SQL Server password is required. Ensure credentials are saved in DBeaver.');
    }

    // Configure connection for AWS RDS
    const config: sql.config = {
      server: host,
      port: port,
      database: database,
      user: user,
      password: password,
      options: {
        encrypt: true,  // AWS RDS requires encryption
        trustServerCertificate: true,  // Trust AWS-managed certificates
        enableArithAbort: true,  // Required by mssql package
        connectionTimeout: this.timeout,
        requestTimeout: this.timeout
      }
    };

    const pool = new sql.ConnectionPool(config);

    try {
      // Establish connection
      await pool.connect();

      if (this.debug) {
        console.log(`SQL Server connected: ${host}:${port}/${database}`);
      }

      // Execute query
      const result = await pool.request().query(query);

      // Extract column names from recordset metadata
      const columns: string[] = result.recordset.columns.map((col: any) => col.name);

      // Convert rows to array format matching QueryResult interface
      const rows: any[][] = result.recordset.map((row: any) =>
        columns.map(col => row[col])
      );

      return {
        columns,
        rows,
        rowCount: result.rowsAffected[0] || rows.length,
        executionTime: 0
      };
    } catch (error) {
      // Enhanced error messages for common issues
      if (error instanceof Error) {
        if (error.message.includes('timeout')) {
          throw new Error(`SQL Server connection timed out connecting to ${host}:${port}. Check network access and firewall rules.`);
        } else if (error.message.includes('Login failed')) {
          throw new Error(`SQL Server authentication failed for user '${user}'. Verify credentials in DBeaver.`);
        } else if (error.message.includes('SSL') || error.message.includes('certificate')) {
          throw new Error(`SQL Server SSL connection failed. AWS RDS requires encrypt=true. Original error: ${error.message}`);
        }
      }
      throw new Error(`SQL Server query failed: ${error instanceof Error ? error.message : String(error)}`);
    } finally {
      try {
        await pool.close();
      } catch (closeError) {
        // Log but don't throw - query may have succeeded
        if (this.debug) {
          console.warn('Failed to close SQL Server connection pool:', closeError);
        }
      }
    }
  }
```

**Step 3: Verify TypeScript compilation**

Run:
```bash
npm run build
```

Expected: Clean compilation with no errors

If you see type errors:
- Check `sql.config` type matches mssql package
- Verify `sql.ConnectionPool` is correctly imported
- Ensure `QueryResult` interface is compatible

**Step 4: Commit method implementation**

```bash
git add src/dbeaver-client.ts
git commit -m "feat(sql-server): implement executeMSSQLQuery method

Add SQL Server query execution following PostgreSQL pattern:
- Extract connection properties from DBeaver config
- Configure AWS RDS SSL options (encrypt, trustServerCertificate)
- Execute query with timeout handling
- Convert recordset to QueryResult format
- Enhanced error messages for connection/auth/SSL failures

Follows design: docs/plans/2025-10-29-sql-server-support-design.md

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 4: Update Driver Detection Logic

**Files:**
- Modify: `src/dbeaver-client.ts:113-124` (executeWithNativeTool method)

**Context:** The `executeWithNativeTool` method routes queries to driver-specific implementations. Currently supports SQLite and PostgreSQL. Need to add SQL Server detection before the fallback error.

**Step 1: Locate executeWithNativeTool method**

Run:
```bash
grep -n "private async executeWithNativeTool" src/dbeaver-client.ts
```

Expected: Method starts around line 113

**Step 2: Review current implementation**

Run:
```bash
sed -n '113,124p' src/dbeaver-client.ts
```

Expected output showing current logic:
```typescript
  private async executeWithNativeTool(connection: DBeaverConnection, query: string): Promise<QueryResult> {
    const driver = connection.driver.toLowerCase();

    if (driver.includes('sqlite')) {
      return this.executeSQLiteQuery(connection, query);
    } else if (driver.includes('postgres')) {
      return this.executePostgreSQLQuery(connection, query);
    } else {
      throw new Error(`Database driver "${driver}" is not yet supported...`);
    }
  }
```

**Step 3: Add SQL Server detection**

Modify the method to add SQL Server check before the error:

```typescript
  private async executeWithNativeTool(connection: DBeaverConnection, query: string): Promise<QueryResult> {
    const driver = connection.driver.toLowerCase();

    if (driver.includes('sqlite')) {
      return this.executeSQLiteQuery(connection, query);
    } else if (driver.includes('postgres')) {
      return this.executePostgreSQLQuery(connection, query);
    } else if (driver.includes('mssql') || driver.includes('sqlserver') || driver.includes('microsoft')) {
      return this.executeMSSQLQuery(connection, query);
    } else {
      throw new Error(`Database driver "${driver}" is not yet supported for direct query execution. Supported drivers: SQLite, PostgreSQL, SQL Server. Please use DBeaver GUI for this connection type.`);
    }
  }
```

**Key changes:**
- Added `else if` for SQL Server detection (checks mssql, sqlserver, microsoft)
- Calls `executeMSSQLQuery` method
- Updated error message to list "SQL Server" as supported

**Step 4: Verify TypeScript compilation**

Run:
```bash
npm run build
```

Expected: Clean compilation

**Step 5: Commit driver detection update**

```bash
git add src/dbeaver-client.ts
git commit -m "feat(sql-server): add SQL Server driver detection

Update executeWithNativeTool to route SQL Server queries:
- Detect mssql, sqlserver, or microsoft in driver name
- Route to executeMSSQLQuery method
- Update error message to list SQL Server as supported

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 5: Build and Verify Binary

**Files:**
- Build: `dist/` (output directory)

**Context:** Final build verification before testing with actual SQL Server connection. Ensure binary is executable and help text displays correctly.

**Step 1: Clean build**

Run:
```bash
npm run clean
```

Expected:
```
> rimraf dist
```

**Step 2: Full build**

Run:
```bash
npm run build
```

Expected: Clean compilation with no errors or warnings

**Step 3: Verify dist output**

Run:
```bash
ls -lh dist/index.js
```

Expected: Executable file with size ~50-100KB

**Step 4: Test help command**

Run from worktree:
```bash
node dist/index.js --help
```

Expected output showing SQL Server is mentioned:
```
DBeaver MCP Server v1.2.3

Usage: dbeaver-mcp-server [options]

Options:
  -h, --help     Show this help message
  --version      Show version information
  --debug        Enable debug logging

Features:
  - Universal database support via DBeaver connections
  - Read and write operations with safety checks
  ...
```

**Step 5: Commit final build verification**

```bash
git add dist/
git commit -m "build: compile SQL Server support

Final build with SQL Server functionality.
All TypeScript compilation successful.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 6: Test with Actual SQL Server Connection

**Files:**
- No file changes (testing only)

**Context:** Manual integration testing with real AWS RDS SQL Server via Claude Code. Tests connection, authentication, and query execution. This validates the implementation works end-to-end.

**Step 1: Update global npm link**

Run from worktree:
```bash
npm link
```

Expected: Creates/updates symlink to new build with SQL Server support

**Step 2: Verify global command**

Run:
```bash
/Users/ntitus/.nvm/versions/node/v22.14.0/bin/dbeaver-mcp-server --version
```

Expected: Shows version 1.2.3

**Step 3: Test via Claude Code**

Navigate to tpv-data directory and start Claude Code session:

```bash
cd /Users/ntitus/Dev/trainingpeaks/tpv-data
```

In Claude Code, test these commands:

**Test 3a: List connections**
Ask: "List all my database connections"
Expected: Shows SQL Server connection in list

**Test 3b: Test connection**
Ask: "Test my SQL Server connection"
Expected: Success with server version (e.g., "Microsoft SQL Server 2019...")

**Test 3c: Execute query**
Ask: "Run this query on SQL Server: SELECT @@VERSION"
Expected: Returns version string

**Test 3d: Analytics query**
Ask: "Run this query on SQL Server: SELECT TOP 10 * FROM [your_table_name]"
Expected: Returns first 10 rows with column names

**Step 4: Document test results**

Create test results file:
```bash
cat > docs/testing/sql-server-verification.md << 'EOF'
# SQL Server Support - Test Results

**Date:** 2025-10-29
**Tester:** Nick Titus
**Environment:** AWS RDS SQL Server

## Test Results

### Test 1: List Connections
- Command: "List all my database connections"
- Result: [PASS/FAIL]
- Notes: [SQL Server connection appeared in list]

### Test 2: Connection Test
- Command: "Test my SQL Server connection"
- Result: [PASS/FAIL]
- Server Version: [version string]

### Test 3: Version Query
- Command: "SELECT @@VERSION"
- Result: [PASS/FAIL]
- Output: [version details]

### Test 4: Data Query
- Command: "SELECT TOP 10 * FROM [table]"
- Result: [PASS/FAIL]
- Rows Returned: [count]
- Columns: [list]

## Issues Found

[None / List any issues]

## Conclusion

SQL Server support [working as expected / needs fixes]
EOF
```

Edit the file with your actual test results.

**Step 5: Commit test results**

```bash
git add docs/testing/
git commit -m "test: document SQL Server integration verification

Manual integration testing with AWS RDS SQL Server:
- Connection listing verified
- Authentication successful
- Query execution working
- Analytics queries returning results

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 7: Finalize and Prepare for Merge

**Files:**
- Review all changes
- Update changelog if exists

**Context:** Final verification before merging to main. Review all commits, ensure code quality, prepare for integration.

**Step 1: Review commit history**

Run:
```bash
git log --oneline main..HEAD
```

Expected: Shows 6-7 commits from this feature branch

**Step 2: View total diff**

Run:
```bash
git diff main...HEAD --stat
```

Expected output:
```
 .gitignore                                    |   3 +
 docs/plans/2025-10-29-sql-server-support-design.md | 300 ++++++++++
 docs/plans/2025-10-29-sql-server-support.md       | 450 +++++++++++++
 package-lock.json                                 |  [changes]
 package.json                                      |   4 +-
 src/dbeaver-client.ts                            |  85 ++-
 6 files changed, ~850 insertions(+)
```

**Step 3: Verify no debug code left**

Run:
```bash
grep -n "console.log\|debugger\|TODO\|FIXME" src/dbeaver-client.ts
```

Expected: Only the intentional debug logging in executeMSSQLQuery (under `if (this.debug)`)

**Step 4: Final build and test**

Run:
```bash
npm run build
node dist/index.js --help
```

Expected: Clean build, help displays correctly

**Step 5: Push feature branch**

```bash
git push origin feature/sql-server-support
```

**Step 6: Document completion**

You're ready to either:
1. Merge to main locally
2. Create PR on GitHub for review
3. Use **@superpowers:finishing-a-development-branch** skill for guided completion

---

## Post-Implementation Notes

**For future reference:**

1. **If connection fails:**
   - Check DBeaver connection properties (Edit Connection → Connection Details)
   - Verify host, port, database name match exactly
   - Ensure password is saved in DBeaver
   - Check network access to AWS RDS (security groups, VPN)

2. **If SSL errors occur:**
   - AWS RDS requires `encrypt: true`
   - Uses `trustServerCertificate: true` for AWS-managed certs
   - No custom certificate files needed

3. **Adding more drivers:**
   - Follow this same pattern
   - Install driver package
   - Add import
   - Implement `execute[Driver]Query` method
   - Update `executeWithNativeTool` detection
   - Update utils.ts queries (already done for most drivers)

4. **Performance optimization:**
   - Connection pooling not needed (single-query pattern)
   - Timeout configured via `this.timeout` from MCP config
   - Query optimization happens in SQL Server, not in MCP

---

## Success Criteria

✅ mssql package installed and types available
✅ Import statement added without TypeScript errors
✅ executeMSSQLQuery method compiles and follows pattern
✅ Driver detection routes SQL Server correctly
✅ Binary builds without errors
✅ SQL Server connection succeeds via Claude Code
✅ Queries execute and return results
✅ Error messages are clear and helpful
✅ No regression on SQLite or PostgreSQL support

**Implementation complete when all criteria met.**
