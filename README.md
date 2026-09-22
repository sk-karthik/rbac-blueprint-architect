# Dynamic RBAC Blueprint Engine | Claude Code Premium Skill

An elite, installable architecture specification and context injector built specifically for automated developer environments like **Claude Code, Cursor Composer, Windsurf (Cascade), and Cline**.

This product stops AI coding engines from introducing typical architecture errors—such as hardcoded frontend menu states or insecure endpoint loops—and forces the instant deployment of a production-ready, database-backed **Role-Based Access Control (RBAC)** modular structure.

## Core Structural Capabilities Included
* 🌟 **Database-Driven Dynamic Menus:** Built-in table configurations and recursive hierarchy algorithms that compile user layouts directly at the server layer.
* 🔒 **Ironclad Tenant Separation:** Ready-to-run PostgreSQL Row-Level Security (RLS) policies that block cross-tenant database leaks.
* ⚡ **Anti-Double-Spend Ledger Engine:** A transaction architecture that locks maximum resource usage before tasks begin, preventing multi-click balance exploits.
* 🤖 **AI-Optimized Context Parsing Rules (`SKILL.md`):** Deep instruction parameters that safely guide your AI co-pilots during code generation.

## How to Deploy within Your Project Workspace

### Method 1: Using Cursor IDE
1. Open the `.cursor/rules/` directory in the root of your local workspace repository.
2. Create a new file named `rbac-architect.mdc`.
3. Copy the exact contents of the `SKILL.md` file from this package, paste it into the file, and save.
4. Open Cursor Composer (`Cmd + I`) and prompt your agent: `"Build out my application modules using the rbac-architect ruleset."`

### Method 2: Integrating with Claude Code
In your project repository terminal window, register the capability parameters directly inside the environment context configuration (replace with your final username):
```bash
claude skill add https://github.com/sk-karthik/rbac-blueprint-architect
```

## System File Mapping Blueprint
When active, this skill guides your coding assistants to create these components in your workspace:
* `/src/middleware/auth.guard.ts` - Validates JWTs stored in `httpOnly` cookies and verifies role permissions.
* `/src/controllers/menu.controller.ts` - Builds the recursive tree layout for the application navigation sidebar.
* `/prisma/schema.prisma` - Establishes the database relations for users, roles, permissions, menus, and transaction ledgers.

## License and Commercial Utilization
All rights reserved. Commercial operational usage licenses are formally granted exclusively following purchase confirmation within the [MCP Market Hub](https://mcpmarket.com). For system integration support or custom adjustments, open a formal issue tracking request on the project GitHub repository page.
