---
title: Trader
tags: [hub, trader, crypto, bingx]
---

# 📈 Trader – Hub

Proyecto de trading de criptomonedas usando Claude Code con el servidor MCP de CCXT conectado al exchange BingX para futuros perpetuos.

---

## 🗂️ Contenido

- [[Trader/CLAUDE|CLAUDE.md]] — instrucciones del proyecto para Claude Code: setup MCP, credenciales, convenciones de trading.

---

## ⚙️ Stack

| Componente | Tecnología |
|-----------|-----------|
| **Claude Code** | CLI de Anthropic con MCP |
| **MCP Server** | `@lazydino/ccxt-mcp` |
| **Exchange** | BingX |
| **Tipo de cuenta** | `bingx_futures` (perpetual swap) |
| **Símbolo formato** | `BTC/USDT:USDT` |

---

## 🔐 Archivos sensibles (gitignored)

- `mcp.json` — configuración del servidor MCP para Claude Code.
- `ccxt-accounts.json` — credenciales API de BingX. **No commitear nunca.**

---

## 🔗 Relacionado

- [[Tools/Claude Code|Claude Code]] — herramienta base del proyecto.
- [[Projects/Projects|Projects]] — otros proyectos del vault.

---

## ↩️ Volver al HOME
- [[HOME|🏠 HOME del Vault]]
