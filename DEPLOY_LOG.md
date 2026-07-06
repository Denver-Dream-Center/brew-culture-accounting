# Deploy Log

- 2026-07-06: Repository transferred from stephenddc to Denver-Dream-Center org (GitHub Team plan) to enable private-repo Pages hosting.
- Resolved: org-level OAuth app access restrictions were blocking connector access (new orgs default to "Access restricted"). Removed restrictions and reconnected the Claude GitHub MCP Connector.
- Note: Claude's GitHub connector still cannot write to this repo directly (authorized but never installed with repo access) — this file was committed manually as a workaround.
