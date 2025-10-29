# SQL Server Support for DBeaver MCP Server

**Date:** 2025-10-29
**Author:** Nick Titus
**Status:** Approved for Implementation

## Purpose

Add SQL Server support to dbeaver-mcp-server for read-only analytics queries against AWS RDS SQL Server databases. The existing DBeaver connection works; the MCP server needs direct query execution capability.

## Context

**Current State:**
- MCP server supports SQLite and PostgreSQL
- SQL Server queries fail with "driver not supported" error
- DBeaver connection to AWS RDS SQL Server works correctly
- User needs read-only analytics access through Claude Code

**Requirements:**
- Execute SELECT queries against SQL Server
- Use existing DBeaver connection credentials
- Handle AWS RDS SSL requirements
- Maintain pattern established by PostgreSQL implementation

**Non-Goals:**
- Stored procedure support (not needed for analytics)
- Connection pooling across queries (single-query pattern works)
- Write operations beyond SELECT (read-only focus)

## Design

### Architecture

Follow the PostgreSQL implementation pattern. Add `mssql` npm package and implement `executeMSSQLQuery()` method in `DBeaverClient` class. Extract connection properties from DBeaver configuration, configure AWS RDS SSL defaults, execute query, return results in standard format.

### Component Changes

**1. Dependencies (package.json)**

Add two packages:
```json
"dependencies": {
  "mssql": "^10.0.0"
},
"devDependencies": {
  "@types/mssql": "^9.1.0"
}
```

**2. Import Statement (dbeaver-client.ts)**

Add at top with other imports:
```typescript
import sql from 'mssql';
```

**3. Driver Detection (executeWithNativeTool method)**

Add SQL Server detection to existing switch:
```typescript
else if (driver.includes('mssql') || driver.includes('sqlserver') || driver.includes('microsoft')) {
  return this.executeMSSQLQuery(connection, query);
}
```

Update error message to include SQL Server in supported list.

**4. Query Execution (new executeMSSQLQuery method)**

Add private method following PostgreSQL pattern:

```typescript
private async executeMSSQLQuery(connection: DBeaverConnection, query: string): Promise<QueryResult> {
  // Extract connection properties
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
    await pool.connect();

    if (this.debug) {
      console.log(`SQL Server connected: ${host}:${port}/${database}`);
    }

    const result = await pool.request().query(query);

    // Extract columns from metadata
    const columns: string[] = result.recordset.columns.map((col: any) => col.name);

    // Convert rows to array format
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
    // Provide clear error messages
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
      if (this.debug) {
        console.warn('Failed to close SQL Server connection pool:', closeError);
      }
    }
  }
}
```

### Configuration Details

**AWS RDS SSL Requirements:**
- `encrypt: true` - AWS requires encrypted connections
- `trustServerCertificate: true` - AWS manages certificates, not user-provided
- Connection timeout matches existing MCP server timeout config

**Connection Property Extraction:**
Mirror DBeaver's property names:
- `host` or `properties.host` → SQL Server `server`
- `port` or `properties.port` → defaults to 1433
- `database` or `properties.database`
- `user` or `properties.user`
- `password` from `properties.password` (DBeaver secure storage)

### Error Handling

Provide specific messages for common failures:

**Connection Timeout:**
```
SQL Server connection timed out connecting to [host]:[port].
Check network access and firewall rules.
```

**Authentication Failure:**
```
SQL Server authentication failed for user '[user]'.
Verify credentials in DBeaver.
```

**SSL Failure:**
```
SQL Server SSL connection failed. AWS RDS requires encrypt=true.
Original error: [error message]
```

**Generic Failure:**
```
SQL Server query failed: [error message]
```

## Testing Plan

**Pre-Implementation:**
1. Document existing DBeaver connection settings (host, port, database, user)
2. Verify DBeaver connection works with test query
3. Note any custom SSL settings in DBeaver

**Post-Implementation:**
1. Install dependencies: `npm install`
2. Build: `npm run build`
3. Verify command works: `dbeaver-mcp-server --help`
4. Test via Claude Code in tpv-data directory:
   - List connections (should show SQL Server)
   - Test connection (executes `SELECT @@VERSION`)
   - Run analytics query: `SELECT TOP 10 * FROM [table]`

**Success Criteria:**
- SQL Server queries execute successfully
- Results return in < 5 seconds (comparable to DBeaver)
- Error messages clearly identify problems
- SQLite and PostgreSQL support unaffected

**Rollback Plan:**
- Revert changes: `git checkout .`
- Previous working state preserved

## Implementation Steps

1. Install mssql packages: `npm install mssql @types/mssql --save`
2. Add import to `src/dbeaver-client.ts`
3. Add `executeMSSQLQuery()` method (75 lines)
4. Update `executeWithNativeTool()` driver detection (3 lines)
5. Build: `npm run build`
6. Test connection and queries
7. Commit with message: "feat: add SQL Server support for AWS RDS"

## Files Modified

- `package.json` - Add mssql dependencies (2 lines)
- `package-lock.json` - Auto-generated by npm install
- `src/dbeaver-client.ts` - Add import + method + detection (~80 lines total)

## Files Unchanged

- `src/utils.ts` - SQL Server queries already defined
- `src/config-parser.ts` - Works with any DBeaver connection
- `src/types.ts` - QueryResult interface already compatible
- `.mcp.json` - Already configured correctly

## Trade-offs

**Chosen: Robust Integration Approach**

Alternatives considered:

1. **Quick Integration** - Copy PostgreSQL exactly without AWS-specific handling
   - Ships in 30 minutes
   - Risk: SSL connection failures with cryptic errors
   - Rejected: Could burn hours debugging AWS SSL issues

2. **Future-Proof** - Full SQL Server feature set (pooling, stored procs, multiple result sets)
   - Ships in 2-3 hours
   - Benefit: Supports all SQL Server features
   - Rejected: Overkill for read-only analytics queries

3. **Robust Integration** - AWS RDS best practices with clear error messages (selected)
   - Ships in 1-2 hours
   - Handles AWS SSL correctly from start
   - Production-ready without iteration
   - Clean foundation for future expansion if needed

## Risks and Mitigations

**Risk: DBeaver connection properties differ from expectations**
- Mitigation: Extract exact properties from working DBeaver connection first
- Fallback: Debug with `debug: true` flag to see connection attempts

**Risk: AWS RDS has non-standard SSL requirements**
- Mitigation: Use `encrypt: true` and `trustServerCertificate: true` (AWS standard)
- Fallback: Check AWS RDS connection string documentation

**Risk: mssql package version incompatibility**
- Mitigation: Use latest stable version (10.0.0)
- Fallback: Pin to known-good version if issues arise

## Future Enhancements

If read-only limitation removed later:
- Add write query validation (existing validateQuery function handles this)
- Test INSERT/UPDATE/DELETE operations
- Add transaction support if needed

If stored procedures needed:
- Add `executeStoredProcedure()` method
- Handle output parameters and return values
- Update MCP tool definitions

Connection pooling not needed (single-query pattern works for analytics).
