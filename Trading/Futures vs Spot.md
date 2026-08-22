---
title: Futures vs Spot
tags: [trading, futures, spot, crypto, leverage]
---

# ⚡ Futures vs Spot

## La diferencia fundamental

| | **Spot** | **Futures** |
|--|----------|-------------|
| **¿Qué compras?** | El activo real | Un contrato (promesa) |
| **¿Posees el crypto?** | Sí | No |
| **¿Puedes perder más de lo que pusiste?** | No | Sí (si usas apalancamiento) |
| **¿Puedes ganar si el precio baja?** | No (solo si lo shorteas primero) | Sí (short directo) |
| **Riesgo máximo** | Perder el 100% de lo invertido | Liquidación total |

---

## Spot: Lo más simple

Compras BTC en el spot market a $30,000. BTC baja a $15,000. Perdiste 50%.

Pero sigues teniendo BTC. Puedes esperar. No te liquidan. No te cobran funding rate. El tiempo corre a tu favor si crees en el activo.

**Casos de uso del spot:**
- Inversión de largo plazo (HODLing).
- Quieres poseer el activo de verdad.
- Baja tolerancia al riesgo.

---

## Futures: Contratos, no activos

En futuros, no compras BTC. Firmas un **contrato** que dice: "si BTC sube, gano; si baja, pierdo" (o al revés si vas short).

Existen dos tipos:
- **Futures con fecha de vencimiento**: el contrato expira en una fecha (ej: cada trimestre en CME).
- **Perpetual Futures (perpetual swaps)**: No expiran nunca. Son los más usados en crypto (Binance, BingX). El mecanismo que los ancla al precio real se llama **funding rate**.

---

## El Apalancamiento (Leverage)

El apalancamiento te permite controlar una posición más grande que tu capital real.

### Ejemplo con 10x de leverage:

Tienes $1,000 y usas 10x de leverage.

Controlas una posición de **$10,000** en BTC.

| Escenario | Movimiento precio | Ganancia/Pérdida sobre $10k | Tu P&L real sobre $1k |
|-----------|------------------|----------------------------|----------------------|
| BTC sube 5% | +$500 | +$500 | **+50%** |
| BTC baja 5% | −$500 | −$500 | **−50%** |
| BTC baja 10% | −$1,000 | −$1,000 | **−100% → LIQUIDADO** |

> Con 10x leverage, un movimiento del 10% en contra **te liquida completamente**.

---

## La Liquidación

Cuando tu pérdida llega al margen que pusiste, el exchange cierra tu posición automáticamente para no dejarte en deuda.

**Precio de liquidación con 10x leverage en long:**
```
Precio entrada = $30,000
Leverage = 10x
Margen = $1,000 (para controlar $10,000)

Precio de liquidación ≈ $30,000 × (1 − 1/10) = $27,000
```

Si BTC baja de $30,000 a $27,000 (−10%), te liquidan.

**Por eso el stop loss es esencial en futuros.**

---

## El Funding Rate

Los perpetual futures no expiran, así que necesitan un mecanismo para mantenerse anclados al precio real del spot. Ese mecanismo es el **funding rate**: un pago periódico entre longs y shorts.

| Situación del mercado | Funding Rate | ¿Quién paga? |
|----------------------|-------------|--------------|
| Mercado alcista, más longs que shorts | Positivo (ej: 0.01% cada 8h) | Los longs pagan a los shorts |
| Mercado bajista, más shorts que longs | Negativo | Los shorts pagan a los longs |

### Ejemplo práctico:
Tienes un long de $10,000 en BTC. El funding rate es 0.05% cada 8 horas.
Cada 8 horas pagas: $10,000 × 0.05% = **$5**.
En un día: $15. En una semana: $105.

**Implicación:** Si el mercado está muy sesgado hacia un lado, el funding rate castiga a esa mayoría. Esto es por qué en mercados muy alcistas con funding alto, los longs de largo plazo en futuros son costosos.

---

## Long vs Short

| Posición | Expectativa | Ganas cuando | Pierdes cuando |
|----------|-------------|-------------|---------------|
| **Long** | Precio sube | BTC sube | BTC baja |
| **Short** | Precio baja | BTC baja | BTC sube |

**Ejemplo de short:**
BTC está en $30,000. Abres un short de $10,000 (sin leverage por simplicidad).
- BTC cae a $24,000 (−20%). Ganaste $2,000 (20% de $10,000).
- BTC sube a $33,000 (+10%). Perdiste $1,000.

---

## Margen Aislado vs Cross Margin

| Tipo | Descripción | Riesgo |
|------|-------------|--------|
| **Isolated Margin** | Solo arriesgas el margen asignado a esa posición | Pierdes solo lo asignado a ese trade |
| **Cross Margin** | Toda tu cuenta respalda la posición | El exchange usa todo tu balance para evitar liquidación. Puedes perder todo |

> Para principiantes: **siempre Isolated Margin**. Sabes exactamente cuánto puedes perder.

---

## Checklist antes de abrir un future

- [ ] ¿Cuál es mi precio de liquidación?
- [ ] ¿Tengo stop loss puesto?
- [ ] ¿Cuál es el funding rate actual? (si es alto, cuidado con longs largos)
- [ ] ¿Cuánto arriesgo en este trade? (nunca más del 1-2% de tu cuenta por trade)
- [ ] ¿Usé Isolated Margin?

---

## Relacionado
- [[Trading/Supply and Demand|Supply & Demand]] — las zonas de S&D son los mejores puntos de entrada en futuros.
- [[Trading/Elliott Impulse Wave|Elliott Impulse Wave]] — las ondas 3 son la mejor oportunidad en futuros con apalancamiento.
- [[Trading/Fibonacci Retracement|Fibonacci Retracement]] — para ubicar entradas de precisión en futuros.
- [[Trader/Trader|Trader]] — proyecto práctico con BingX perpetual futures.

---

## ↩️ Volver
- [[Trading/Trading|📊 Trading Hub]]
