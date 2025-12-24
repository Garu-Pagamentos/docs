# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the **Garu API documentation site** built with [Mintlify](https://mintlify.com). Garu is a Brazilian payment gateway supporting PIX, Credit Card, and Boleto payments.

**All documentation content must be written in Brazilian Portuguese.**

## Commands

```bash
# Install Mintlify CLI (first time only)
npm i -g mint

# Start local development server
mint dev

# Update Mintlify CLI
mint update
```

Local preview runs at `http://localhost:3000`.

## Architecture

### Configuration
- `docs.json` - Main configuration file (navigation, colors, metadata, OpenAPI reference)
- `.mintignore` - Files/folders to exclude from documentation build

### Content Structure
- `index.mdx`, `quickstart.mdx` - Landing pages
- `guias/` - User guides (criando-produtos, producao)
- `api-reference/` - API documentation
  - `produtos/` - Product endpoints (criar, listar, atualizar, excluir, link-pagamento)
  - `webhooks.mdx`, `exemplos.mdx`, `troubleshooting.mdx`
- `docs/` - Internal reference files (ignored by Mintlify, contains styleguide and temp files)

### OpenAPI Integration
The API spec is fetched from `https://garu.com.br/api/swagger-json` (configured in docs.json).

## Brand Guidelines

Colors from Garu styleguide (`docs/GARU_STYLEGUIDE.md`):
- **Jade (primary)**: `#257264` - Trust, growth, stability
- **Equinócio (light)**: `#FAF6ED` - Modernity, backgrounds
- **Jade Dark**: `#005243` - Dark mode, contrast

## Adding New Pages

1. Create `.mdx` file in appropriate directory
2. Add page path to `docs.json` navigation
3. Use Mintlify components: `<Card>`, `<CodeGroup>`, `<Steps>`, `<Accordion>`, etc.
