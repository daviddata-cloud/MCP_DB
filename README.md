# DB MCP (HR CSV → SQLite) — Open Source Reference

This folder contains a **fully open-source** Model Context Protocol (MCP) server implementation that:

- Loads an HR “people” CSV file
- Reads **3 lines of metadata** at the top of the CSV (comment lines starting with `#`)
- Imports the CSV into an **in-memory SQLite** database
- Exposes **read-only MCP tools** over **stdio** (newline-delimited JSON-RPC 2.0)

No Claude Desktop setup is required. A small Python client is included for testing.

## Files

- `db_mcp_server.py` — MCP server (stdio)
- `db_mcp_client.py` — simple MCP stdio client for testing
- `data/hr_people.csv` — sample HR CSV with 3-line metadata header

## Run the server

```bash
python db_mcp_server.py
```

Optionally pass a custom CSV path:

```bash
python db_mcp_server.py /path/to/your/hr_people.csv
```

Or set an environment variable:

```bash
HR_CSV_PATH=/path/to/your/hr_people.csv python db_mcp_server.py
```

## Test with the included client (recommended)

```bash
python db_mcp_client.py
```

You should see:

- `initialize` handshake
- `tools/list`
- a sample SQL query result
- an interactive prompt to run more `SELECT` queries

## Tools exposed

- `hr_metadata` — returns the 3-line metadata header as a JSON object
- `hr_schema` — returns the SQLite schema for table `employees`
- `hr_query` — execute **read-only** `SELECT`/`WITH` SQL queries
- `hr_find_people` — structured search without writing SQL

## CSV metadata format (first 3 lines)

Example:

```text
# dataset: HR People
# description: Synthetic employee roster for MCP demo (no real PII)
# primary_key: employee_id
employee_id,first_name,last_name,...
```

Metadata lines are parsed as `key: value`. If a line is not `key: value`, it is stored as `meta_line_1`, `meta_line_2`, etc.

## Notes for sharing

- Everything here is standard-library Python (SQLite + CSV).
- The demo data is synthetic (no real PII).
- The server writes **only JSON-RPC** to stdout. Logs go to stderr (safe for stdio MCP).
  
## Samples
```
-Terminal 1

C:\Users\davidzhang\Downloads\ml\ml\db_mcp_1>python mcp_server.py data/hr_people.csv
[db_mcp_server] Ready. Loaded data/hr_people.csv. Tools: 4


-Terminal 2
 C:\Users\davidzhang\Downloads\ml\ml\db_mcp_1>python client.py --csv ./data/hr_people.csv
[db_mcp_server] Ready. Loaded ./data/hr_people.csv. Tools: 4
[db_mcp_server] Internal error:
Traceback (most recent call last):
  File "C:\Users\davidzhang\Downloads\ml\ml\db_mcp_1\db_mcp_server.py", line 537, in main
    server.handle(msg)
  File "C:\Users\davidzhang\Downloads\ml\ml\db_mcp_1\db_mcp_server.py", line 493, in handle
    self.handle_initialize(id_value, params)
  File "C:\Users\davidzhang\Downloads\ml\ml\db_mcp_1\db_mcp_server.py", line 438, in handle_initialize
Initialize response:
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32603,
    "message": "Internal error"
  },
  "id": 1
}
    _send(resp)

Tools:
  File "C:\Users\davidzhang\Downloads\ml\ml\db_mcp_1\db_mcp_server.py", line 45, in _send
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "tools": [
      {
        "name": "hr_metadata",
        "title": "HR dataset metadata",
        "description": "Return the 3-line metadata header read from the HR CSV file.",
        "inputSchema": {
          "type": "object",
          "additionalProperties": false
        },
        "outputSchema": {
          "type": "object"
        }
      },
      {
        "name": "hr_schema",
        "title": "HR table schema",
        "description": "Return SQLite schema information for the employees table.",
        "inputSchema": {
          "type": "object",
          "additionalProperties": false
        },
        "outputSchema": {
          "type": "object"
        }
      },
      {
        "name": "hr_query",
        "title": "Run a read-only SQL query",
        "description": "Execute a read-only SQL query (SELECT/WITH only) against the in-memory SQLite database.\nTable name: employees\nExample: SELECT department, COUNT(*) AS n FROM employees GROUP BY department",
        "inputSchema": {
          "type": "object",
          "properties": {
            "sql": {
              "type": "string",
              "description": "A SELECT/WITH SQL query to run."
            },
            "limit": {
              "type": "integer",
              "minimum": 1,
              "maximum": 500,
              "description": "Optional row limit (wraps the query)."
            }
          },
          "required": [
            "sql"
          ],
          "additionalProperties": false
        },
        "outputSchema": {
          "type": "object",
          "properties": {
            "rowCount": {
              "type": "integer"
            },
            "rows": {
              "type": "array",
              "items": {
                "type": "object"
              }
            }
          },
          "required": [
            "rowCount",
            "rows"
          ]
        }
      },
      {
        "name": "hr_find_people",
        "title": "Find employees (structured filters)",
        "description": "Find employees by common HR filters without writing SQL.",
        "inputSchema": {
          "type": "object",
          "properties": {
            "name_contains": {
              "type": "string",
              "description": "Substring match against first or last name (case-insensitive)."
            },
            "department": {
              "type": "string"
            },
            "title": {
              "type": "string"
            },
            "location": {
              "type": "string"
            },
            "min_salary": {
              "type": "number"
            },
            "max_salary": {
              "type": "number"
            },
            "hired_after": {
              "type": "string",
              "description": "YYYY-MM-DD"
            },
            "hired_before": {
              "type": "string",
              "description": "YYYY-MM-DD"
            },
            "limit": {
              "type": "integer",
              "minimum": 1,
              "maximum": 200,
              "default": 25
            }
          },
          "additionalProperties": false
        },
        "outputSchema": {
          "type": "object",
          "properties": {
            "rowCount": {
              "type": "integer"
            },
            "rows": {
              "type": "array",
              "items": {
                "type": "object"
              }
            },
            "appliedFilters": {
              "type": "object"
            }
          },
          "required": [
            "rowCount",
            "rows",
            "appliedFilters"
          ]
        }
      }
  }
}

