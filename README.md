# Strands SDK

Monorepo for the Strands Agents SDK — Python, TypeScript, WASM bridge, and developer tooling.

## Packages

| Directory | Description | Registry |
|-----------|-------------|----------|
| [`strands-py/`](./strands-py/) | Python SDK (1.x) | [PyPI: `strands-agents`](https://pypi.org/project/strands-agents/) |
| [`strands-ts/`](./strands-ts/) | TypeScript SDK | [npm: `@strands-agents/sdk`](https://www.npmjs.com/package/@strands-agents/sdk) |
| [`strands-mcp/`](./strands-mcp/) | MCP server | [PyPI: `strands-agents-mcp-server`](https://pypi.org/project/strands-agents-mcp-server/) |
| [`strands-wasm/`](./strands-wasm/) | WASM bridge | — |
| [`strands-py-wasm/`](./strands-py-wasm/) | Python 2.x projection (WASM-based) | — |
| [`strandly/`](./strandly/) | Dev CLI tooling | — |
| [`site/`](./site/) | Documentation site (Astro) | — |
| [`designs/`](./designs/) | Design proposals | — |
| [`wit/`](./wit/) | WIT interface definitions | — |

## Development

### Python SDK

```bash
cd strands-py
pip install hatch
hatch test tests --cover
hatch fmt --linter --check
```

### TypeScript SDK

```bash
npm ci
npm run build
npm run test:all:coverage
```

## Release Process

Releases use namespaced tags to trigger publishing:

- **Python SDK**: Push tag `python/v<version>` (e.g., `python/v1.41.0`)
- **TypeScript SDK**: Push tag `typescript/v<version>` (e.g., `typescript/v1.3.0`)
- **MCP Server**: Push tag `mcp/v<version>` (e.g., `mcp/v0.3.0`)

## Repository Structure

This repo was formed by consolidating:
- [`strands-agents/sdk-python`](https://github.com/strands-agents/sdk-python) → `strands-py/`
- [`strands-agents/sdk-typescript`](https://github.com/strands-agents/sdk-typescript) → `strands-ts/`, `strands-wasm/`, `strands-py-wasm/`, `strandly/`, `wit/`
- [`strands-agents/docs`](https://github.com/strands-agents/docs) → `site/`, `designs/`
- [`strands-agents/mcp-server`](https://github.com/strands-agents/mcp-server) → `strands-mcp/`

Full git history from all repositories is preserved.

## License

This project is dual-licensed under the Apache License 2.0 and MIT License.
