# mcp-www

**DNS-based MCP service discovery over UDP.**

## Problem

Agents need to discover MCP servers, but current approaches lean on centralized registries or hardcoded configurations. This creates single points of failure, adds deployment overhead, and forces agents into walled gardens. There should be a way to discover MCP services using infrastructure that already exists everywhere: DNS.

## How It Works

**mcp-www** is itself a standard MCP server. An agent connects to it the same way it connects to any other MCP server — no new client code, no special SDK, no registry signup.

Once connected, the agent calls the `browse_domain` tool with a domain name. mcp-www performs a standard **UDP DNS TXT lookup** for `_mcp.{domain}`, parses the semicolon-delimited record, and returns structured metadata about the MCP servers published by that domain.

```
Agent  →  mcp-www (MCP server)  →  UDP DNS query for _mcp.example.com TXT
                                    ←  "v=mcp1; src=https://mcp.example.com; ..."
       ←  Structured JSON response
```

No HTTP registry in the loop. The DNS infrastructure **is** the registry.

## Key Design Points

- **Uses UDP DNS (port 53) for lookups** — the lightest possible network primitive. No TCP handshake, no TLS negotiation, no HTTP overhead. A single UDP packet out, a single packet back.
- **The DNS infrastructure IS the registry** — no additional servers to deploy, no uptime to maintain, no accounts to create. If you can publish a TXT record, you can advertise your MCP server.
- **mcp-www is a standard MCP server** — any MCP-compliant agent can use it with zero new client code. It's just another server in your agent's config.
- **Supports the `_mcp` TXT record convention** — records follow a semicolon-delimited format:
  ```
  v=mcp1; src=https://mcp.example.com; auth=oauth2
  ```
- **Works with split-horizon DNS** — enterprise and private networks can publish internal `_mcp` records visible only inside their network, enabling private service discovery without exposing anything to the public internet.

## Tools Exposed

### `browse_domain`

Lookup `_mcp.{domain}` TXT records and return a parsed list of discovered MCP servers.

```json
{
  "tool": "browse_domain",
  "arguments": {
    "domain": "example.com"
  }
}
```

Returns structured server metadata: server URL, protocol version, auth requirements, and any additional fields published in the TXT record.

### `browse_server`

Given a discovered server URL, connect to it and retrieve its `tools/list` manifest. Lets the agent inspect what a discovered server actually offers before deciding to connect.

```json
{
  "tool": "browse_server",
  "arguments": {
    "url": "https://mcp.example.com"
  }
}
```

### `browse_multi`

Batch lookup across multiple domains in a single call. Useful for scanning a list of known domains or performing broad discovery.

```json
{
  "tool": "browse_multi",
  "arguments": {
    "domains": ["example.com", "acme.org", "internal.corp"]
  }
}
```

### `call_remote_tool`

Call a tool on a remote MCP server. Use `browse_server` first to discover available tools, then use this to execute them. Handles the JSON-RPC initialize handshake and `tools/call` request.

```json
{
  "tool": "call_remote_tool",
  "arguments": {
    "url": "https://mcp.example.com",
    "tool": "list_articles",
    "arguments": { "limit": 5 }
  }
}
```

## Try It

**[korm.co](https://korm.co)** publishes a live `_mcp` TXT record. You can discover and interact with it end-to-end:

```
browse_domain("korm.co")        → discovers MCP server at https://mcp.korm.co
browse_server("https://mcp.korm.co")  → lists available tools (list_articles, get_article, get_author_info)
call_remote_tool("https://mcp.korm.co", "list_articles")  → returns blog articles
```

## Status

**Working.** The server implements DNS-based discovery, server inspection, and remote tool calling over the Streamable HTTP transport.

Feedback, criticism, and alternative approaches are welcome — open an issue or start a discussion.

## Related

- [Model Context Protocol Specification](https://spec.modelcontextprotocol.io)
- [MCP Discussion #2334](https://github.com/modelcontextprotocol/specification/discussions/2334)
- [MCP PR #2127](https://github.com/modelcontextprotocol/specification/pull/2127)
- [MCP SEP #1959](https://github.com/nicobailon/mcp-seps/blob/main/SEP/1959/README.md)

## License

MIT
