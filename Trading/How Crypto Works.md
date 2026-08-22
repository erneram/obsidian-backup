---
title: How Crypto Works
tags: [trading, crypto, blockchain, fundamentals]
---

# 🪙 Cómo Funciona el Crypto

## El problema que resuelve

Antes del crypto, para transferir dinero necesitabas un banco: una institución central que dice "Juan tiene $500, María tiene $300, Juan le transfiere $100 a María → ahora Juan tiene $400 y María $400".

El problema: **confías en el banco**. El banco puede bloquearte, congelarte la cuenta, equivocarse, o ser corrupto.

**Bitcoin resolvió esto en 2009**: ¿qué tal si ese registro de quién tiene qué no lo controla un banco, sino **miles de computadoras simultáneamente**? Si alguien intenta hackear el registro, tiene que hackear miles de copias al mismo tiempo. Matemáticamente imposible.

---

## El Blockchain

El blockchain es un **libro de contabilidad distribuido**. En lugar de un servidor central, hay miles de copias idénticas repartidas por el mundo.

Cada "página" del libro se llama **bloque**. Cada bloque contiene:
1. Un grupo de transacciones (ej: "Wallet A le envió 0.5 BTC a Wallet B").
2. Un sello de tiempo.
3. Un **hash** del bloque anterior (lo que los conecta en cadena).

### ¿Qué es un hash?

Es una huella digital matemática. Tomas cualquier texto y una función matemática lo convierte en un string único de letras y números:

```
"Hola" → a8f5f167f44f4964e6c998dee827110c
"Hola!" → dca6a4028cbca9e6fc44f80c11d6b0d5
```

Cambiar **una sola letra** da un hash completamente diferente. Por eso si alguien modifica una transacción, el hash cambia, el bloque siguiente ya no encaja, y toda la red lo rechaza.

---

## Los Mineros / Validadores

Alguien tiene que verificar que las transacciones son legítimas antes de añadirlas al bloque. Esos son los **mineros** (Bitcoin) o **validadores** (Ethereum y la mayoría de cryptos modernas).

### Proof of Work (Bitcoin)
Los mineros compiten resolviendo un problema matemático difícil. El primero en resolverlo gana el derecho a añadir el siguiente bloque y recibe una recompensa en BTC. Gasta electricidad real → hace que atacar la red sea carísimo.

### Proof of Stake (Ethereum, Solana, etc.)
En lugar de gastar electricidad, los validadores **depositan** (stakean) crypto como garantía. Si validan transacciones falsas, pierden su depósito. El incentivo a ser honesto es económico, no energético.

---

## Wallets y Llaves

Tu "wallet" no guarda crypto. La crypto siempre está en el blockchain. La wallet guarda tus **llaves**.

| Llave | Analogía | Función |
|-------|----------|---------|
| **Llave Pública** | Tu número de cuenta bancaria | La compartes para recibir fondos |
| **Llave Privada** | Tu PIN + contraseña del banco | La firmas para autorizar transacciones. **NUNCA la compartas.** |
| **Seed Phrase** | La llave maestra de tu bóveda | 12-24 palabras que generan tu llave privada. Si la pierdes, pierdes todo. |

### ¿Cómo funciona una transacción?

1. Tú firmas la transacción con tu **llave privada** (como firmar un cheque).
2. La red verifica que la firma es válida usando tu **llave pública** (como verificar tu firma en el banco).
3. La transacción se añade al bloque y se transmite a todos los nodos.
4. Confirmada en ~10 minutos para BTC, ~12 segundos para ETH.

---

## Exchanges: CEX vs DEX

### CEX (Centralized Exchange) — Binance, Coinbase, BingX
Es como un banco crypto. Tú depositas crypto, el exchange la guarda. Tú confías en ellos. Si el exchange quiebra (ej: FTX en 2022), pierdes tu crypto.

> "Not your keys, not your coins."

### DEX (Decentralized Exchange) — Uniswap, dYdX
Operas directamente desde tu wallet. Nadie custodia tu crypto. Más seguro pero más complejo.

---

## ¿Por qué tiene valor el crypto?

| Factor | Explicación |
|--------|-------------|
| **Escasez** | Bitcoin tiene un máximo de 21 millones de unidades. Nunca habrá más. |
| **Utilidad** | Ethereum se usa para ejecutar contratos inteligentes. |
| **Demanda** | Más personas queriendo comprar → precio sube. |
| **Narrativa** | Oro digital, reserva de valor, tecnología del futuro → afecta la demanda. |
| **Especulación** | La mayoría del movimiento de precio es especulativo, especialmente en altcoins. |

---

## Ejemplo completo de una transacción

1. Tú tienes 1 BTC en tu wallet (`bc1qxy2...`).
2. Quieres enviar 0.1 BTC a tu amigo (`bc1qab3...`).
3. Tu wallet firma la transacción con tu llave privada.
4. La transacción se transmite a la red de nodos.
5. Los mineros la validan y la incluyen en el siguiente bloque.
6. Después de 6 confirmaciones (~60 minutos), la transacción es irreversible.
7. Tu amigo ahora tiene 0.1 BTC y tú tienes 0.9 BTC (menos la fee de red).

---

## Relacionado
- [[Trading/Futures vs Spot|Futures vs Spot]] — cómo operar crypto en los mercados.
- [[Trading/Supply and Demand|Supply & Demand]] — cómo la oferta y demanda determina el precio del crypto.
- [[Trader/Trader|Trader]] — proyecto práctico de trading con BingX.

---

## ↩️ Volver
- [[Trading/Trading|📊 Trading Hub]]
