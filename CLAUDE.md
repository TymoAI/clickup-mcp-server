# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **Model Context Protocol (MCP) server** for ClickUp integration. It enables AI applications to interact with ClickUp workspaces through a standardized protocol, supporting task management, time tracking, document management, and workspace organization.

**Package:** `@taazkareem/clickup-mcp-server`
**Version:** 0.8.5
**License:** MIT
**Node Requirements:** >=18.0.0 <23.0.0

## Development Commands

### Building
```bash
# Full build (TypeScript compilation + make executable)
npm run build

# Watch mode for development
npm run dev

# Start the built server
npm start
```

The build process:
1. Compiles TypeScript to JavaScript (outputs to `build/`)
2. Sets executable permissions on `build/index.js`

### Testing
While test scripts are defined in package.json, the actual test files need to be implemented. The server can be tested using:
- MCP Inspector (for HTTP Streamable transport)
- Example SSE client in `examples/` directory
- Integration with Claude Desktop or other MCP clients

### Environment Setup
Required environment variables:
```bash
CLICKUP_API_KEY=your-api-key        # From ClickUp Settings > Apps
CLICKUP_TEAM_ID=your-team-id        # From workspace URL
```

Optional configuration:
```bash
DOCUMENT_SUPPORT=true               # Enable document management tools (default: false)
LOG_LEVEL=info                      # trace|debug|info|warn|error (default: error)
ENABLED_TOOLS=create_task,get_task  # Whitelist specific tools
DISABLED_TOOLS=delete_task          # Blacklist specific tools
ENABLE_SSE=true                     # Enable HTTP/SSE transport (default: false)
PORT=3231                           # HTTP server port (default: 3231)
ENABLE_SECURITY_FEATURES=true       # Enable security enhancements
ENABLE_HTTPS=true                   # Enable HTTPS/TLS
```

## Architecture

### Transport Layer
The server supports multiple transport mechanisms:

1. **STDIO Transport (Default)** - Standard MCP transport using stdin/stdout
   - Used by most MCP clients (Claude Desktop, etc.)
   - Entry point: `src/index.ts` → `startStdioServer()`

2. **HTTP Streamable Transport** - Modern HTTP-based MCP
   - MCP Inspector compatible
   - Endpoint: `http://127.0.0.1:3231/mcp`
   - Entry point: `src/sse_server.ts` → `startSSEServer()`

3. **SSE Transport (Legacy)** - Server-Sent Events for backwards compatibility
   - Endpoint: `http://127.0.0.1:3231/sse`
   - Used by n8n and custom integrations

All transports share the same unified server configuration (70% code reduction from previous architecture).

### Core Components

**Entry Point:** `src/index.ts`
- Process lifecycle management (uncaught exceptions, unhandled rejections)
- Transport selection based on `ENABLE_SSE` config
- Environment logging and startup orchestration

**Server Configuration:** `src/server.ts`
- Exports `server` instance and `configureServer()` function
- MCP request handlers: `ListTools`, `CallTool`, `ListResources`, `ListPrompts`, `GetPrompt`
- Tool filtering logic: `isToolEnabled()` (respects `ENABLED_TOOLS` > `DISABLED_TOOLS` precedence)
- Request routing to appropriate tool handlers via switch statement

**Configuration:** `src/config.ts`
- Parses environment variables and `--env` CLI arguments
- Validates required credentials (`CLICKUP_API_KEY`, `CLICKUP_TEAM_ID`)
- Log level enum and parsing (`LogLevel.TRACE` to `LogLevel.ERROR`)
- Security configuration (opt-in for backwards compatibility)

**Services:** `src/services/clickup/`
- Factory pattern: `createClickUpServices(config)` returns all service instances
- Services: `WorkspaceService`, `TaskService`, `ListService`, `FolderService`, `ClickUpTagService`, `TimeTrackingService`, `DocumentService`
- Shared singleton: `clickUpServices` in `src/services/shared.ts` (prevents duplicate initialization)
- Base class pattern: All services extend `BaseClickUpService` with error handling

### Tools Organization

Tools are organized by domain in `src/tools/`:

- **Task Tools** (`src/tools/task/`) - Most complex subsystem
  - `handlers.ts` - Main handler implementations with validation
  - `single-operations.ts` - Single task CRUD operations
  - `bulk-operations.ts` - Concurrent batch processing
  - `workspace-operations.ts` - Cross-workspace task queries
  - `time-tracking.ts` - Time entry management
  - `attachments.ts` - File attachment handling
  - `utilities.ts` - Shared helpers (name resolution, validation)

- **Workspace Tools** (`src/tools/workspace.ts`) - Hierarchy navigation
- **List Tools** (`src/tools/list.ts`) - List management
- **Folder Tools** (`src/tools/folder.ts`) - Folder operations
- **Tag Tools** (`src/tools/tag.ts`) - Tag CRUD and task tagging
- **Member Tools** (`src/tools/member.ts`) - User lookup and assignee resolution
- **Document Tools** (`src/tools/documents.ts`) - Document and page management (opt-in via `DOCUMENT_SUPPORT`)

Each tool exports:
1. Tool definition object (name, description, schema)
2. Handler function (validation + service call + response formatting)

### Key Architectural Patterns

1. **Unified Server Architecture** - All transports (STDIO, HTTP Streamable, SSE) use the same `configureServer()` logic, eliminating code duplication

2. **Tool Filtering** - Precedence: `ENABLED_TOOLS` (whitelist) > `DISABLED_TOOLS` (blacklist) > all enabled (default)

3. **Name-based Entity Resolution** - All entities (tasks, lists, folders, spaces) support lookup by name OR ID with smart disambiguation

4. **Shared Services** - Single instance of ClickUp services across the application via `src/services/shared.ts`

5. **Error Handling** - JSON-RPC error codes:
   - `-32601`: Method not found (unknown tool or disabled tool)
   - `-32602`: Invalid params (validation error)
   - `-32000`: Generic server error

6. **Logging** - Structured logging via `Logger` class in `src/logger.ts` with configurable levels

### Tool Count
Currently **46 tools** across 7 categories:
- Workspace: 1 tool
- Task: 21 tools (including bulk operations and time tracking)
- List: 5 tools
- Folder: 4 tools
- Tag: 3 tools
- Member: 3 tools
- Document: 7 tools (optional, enabled via `DOCUMENT_SUPPORT=true`)

## Important Implementation Notes

### Adding New Tools
1. Create tool definition and handler in appropriate `src/tools/*.ts` file
2. Export from `src/tools/index.ts`
3. Import in `src/server.ts`
4. Add to `ListToolsRequestSchema` handler array
5. Add case to `CallToolRequestSchema` switch statement
6. Update tool count in logging statement

### Natural Language Date Parsing
Tasks support natural language dates for `dueDate` and `startDate`:
- Absolute: "now", "today", "tomorrow", "next Monday"
- Time-specific: "tomorrow at 9am", "next Friday at 2pm"
- Ranges: "start of today", "end of week"

### Global Task Lookup
Tasks can be found by name across the entire workspace without specifying a list:
- `get_task` with only `taskName` searches all lists
- Smart disambiguation shows context (list/folder/space) when multiple matches exist
- Prioritizes most recently updated task

### Custom Task IDs
Server automatically detects and handles custom task IDs (e.g., `DEV-1234`, `PROJ-456`)

### Security Features
All security features are **opt-in** and **disabled by default**:
- HTTPS/TLS encryption
- Origin validation
- Rate limiting
- Security headers

See `docs/security-features.md` for detailed configuration.

### Sponsor Message
The server includes an optional sponsor message in tool responses (controlled by `ENABLE_SPONSOR_MESSAGE` env var, default: true).

## File Structure
```
src/
├── index.ts              # Entry point, transport selection
├── server.ts             # MCP server configuration and request routing
├── config.ts             # Environment/CLI configuration parsing
├── logger.ts             # Structured logging with levels
├── sse_server.ts         # HTTP/SSE transport server
├── services/
│   ├── clickup/          # ClickUp API service layer
│   │   ├── index.ts      # Service factory and exports
│   │   ├── base.ts       # BaseClickUpService class
│   │   ├── workspace.ts  # WorkspaceService
│   │   ├── task/         # TaskService (split into multiple files)
│   │   ├── list.ts       # ListService
│   │   ├── folder.ts     # FolderService
│   │   ├── tag.ts        # ClickUpTagService
│   │   ├── time.ts       # TimeTrackingService
│   │   └── document.ts   # DocumentService
│   └── shared.ts         # Shared singleton services
├── tools/                # MCP tool definitions and handlers
│   ├── index.ts          # Tool exports
│   ├── workspace.ts      # Workspace hierarchy tool
│   ├── task/             # Task management tools (subdivided)
│   ├── list.ts           # List management tools
│   ├── folder.ts         # Folder management tools
│   ├── tag.ts            # Tag management tools
│   ├── member.ts         # Member management tools
│   └── documents.ts      # Document management tools
├── middleware/
│   └── security.ts       # Security middleware (HTTPS, rate limiting, etc.)
└── utils/                # Shared utilities

build/                    # TypeScript compilation output (gitignored)
docs/                     # User guides and documentation
examples/                 # Example clients (SSE client, etc.)
scripts/                  # Utility scripts (SSL cert generation, etc.)
```

## Common Development Tasks

### Running with Different Transports
```bash
# STDIO (default)
npm start

# HTTP Streamable + SSE
ENABLE_SSE=true PORT=3231 npm start

# With security features
ENABLE_SECURITY_FEATURES=true ENABLE_HTTPS=true npm start
```

### Debugging
Enable detailed logging:
```bash
LOG_LEVEL=debug npm start
```

### Tool Filtering for Testing
```bash
# Only enable specific tools
ENABLED_TOOLS=create_task,get_task,update_task npm start

# Disable destructive operations
DISABLED_TOOLS=delete_task,delete_bulk_tasks npm start
```

## Publishing
Package is published to npm with executable binary:
```bash
npm run prepare  # Runs build automatically
npm publish
```

The `bin` field in package.json makes `clickup-mcp-server` available as a CLI command after installation.

## Integration Examples

### Claude Desktop
Add to `claude_desktop_config.json`:
```json
{
  "mcpServers": {
    "ClickUp": {
      "command": "npx",
      "args": ["-y", "@taazkareem/clickup-mcp-server@latest"],
      "env": {
        "CLICKUP_API_KEY": "your-api-key",
        "CLICKUP_TEAM_ID": "your-team-id"
      }
    }
  }
}
```

### n8n Workflow Automation
Start with SSE enabled, then use MCP AI Tool node with SSE transport pointing to `http://localhost:3231`.

### MCP Inspector
Use HTTP Streamable endpoint: `http://127.0.0.1:3231/mcp`
