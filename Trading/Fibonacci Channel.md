---
title: Fibonacci Channel
tags: [trading, fibonacci, technical-analysis]
---

# 📐 Fibonacci Channel

## ¿Qué es?

El Fibonacci Channel no mide pullbacks verticales como el retracement. Mide el **ancho de una tendencia** y proyecta **canales paralelos** donde el precio puede moverse dentro de esa tendencia.

Es como darle "carriles" a la autopista del precio.

---

## Diferencia clave con el Retracement

| | Fibonacci Retracement | Fibonacci Channel |
|---|---|---|
| **Mide** | Cuánto retrocede verticalmente | Cuánto se desvía lateralmente de la tendencia |
| **Líneas** | Horizontales | Paralelas a la tendencia (diagonales) |
| **Útil para** | Encontrar entradas en pullback | Proyectar rangos dentro de una tendencia |

---

## Cómo se traza — La dirección importa todo

Necesitas **3 puntos** y el orden en que los pones determina hacia dónde se proyectan los canales.

> La regla: los 3 puntos deben seguir la dirección de la tendencia. Los canales se proyectan **más allá del punto B**, en la dirección donde el precio seguirá moviéndose.

---

### Tendencia alcista → A abajo, B arriba, C arriba (sobre la línea AB)

El precio sube. Quieres proyectar canales hacia arriba.

**Orden de clics:**
1. **Punto A** → mínimo del primer swing (clic aquí primero, abajo).
2. **Punto B** → máximo del primer swing (clic aquí, arriba).
3. **Punto C** → el siguiente mínimo de corrección, que queda **por encima de A** (clic aquí).

```
                    [canales proyectados hacia arriba: 1.618, 2.618]
          B ────────────────────────────────────
         /    ↑ canal 1.0 (pasa por C)
        /   C ─────────────────────────────────  ← tercer clic
       /   /   ↑ canal 0.618
      /   /
     A ──────────────────────────────────────── ← primer clic
```

Los canales aparecen paralelos a la línea AB, proyectándose hacia arriba y hacia abajo de ella. El canal 1.0 pasa por C y los canales 1.618, 2.618 son los targets de precio hacia arriba.

---

### Tendencia bajista → A arriba, B abajo, C abajo (bajo la línea AB)

El precio baja. Quieres proyectar canales hacia abajo.

**Orden de clics:**
1. **Punto A** → máximo del primer swing bajista (clic aquí primero, arriba).
2. **Punto B** → mínimo del primer swing bajista (clic aquí, abajo).
3. **Punto C** → el siguiente máximo de corrección, que queda **por debajo de A** (clic aquí).

**Ejemplo: BTC cae de $30,000 → $20,000, rebota a $24,000 (punto C).**

```
A = $30,000 ─────────────────────────────────── ← primer clic
      \   \
       \   C = $24,000 ──────────────────────── ← tercer clic (canal 1.0)
        \   \
         \   [canales proyectados hacia abajo: 1.618 ≈ $13,820, 2.618 ≈ $3,820]
          B = $20,000 ──────────────────────── ← segundo clic
```

Los canales proyectados por debajo de B son tus **targets bajistas**.

---

### Resumen de dirección

| Tendencia | Punto A | Punto B | Punto C | Canales proyectados |
|-----------|---------|---------|---------|---------------------|
| **Alcista** | Mínimo (abajo) | Máximo (arriba) | Siguiente mínimo (sobre A) | Hacia arriba de B |
| **Bajista** | Máximo (arriba) | Mínimo (abajo) | Siguiente máximo (bajo A) | Hacia abajo de B |

---

### ¿Qué pasa si pones C en el lado equivocado?

Si en un uptrend pones C debajo de A (en lugar de entre A y B), los canales se proyectan hacia abajo en lugar de hacia arriba. Los niveles quedan invertidos y sin sentido. Si ves que los canales van en la dirección opuesta a la tendencia, rehaz el trazado con C en el lado correcto.

---

### En TradingView

1. Selecciona **"Fib Channel"** en las herramientas de dibujo.
2. Haz clic en A → B → C en el orden correcto según la tendencia.
3. TradingView ajusta automáticamente los canales paralelos a A→B pasando por C.

---

## Ejemplo concreto: ETH en tendencia alcista

Supón que ETH forma estos puntos:
- A = $1,000 (mínimo)
- B = $2,000 (máximo, primer impulso)
- C = $1,500 (corrección, pullback al 50%)

La línea base va de A a B. El canal 1.0 pasa por C. Luego el canal proyecta:

| Canal Fib | Lo que representa | Precio proyectado (aprox.) |
|-----------|-------------------|---------------------------|
| 0.0 (línea AB) | El techo inicial | $2,000 en t0 |
| 0.618 | Resistencia inferior | Entre A y B |
| **1.0** | Pasa por el punto C | $1,500 (ya visible) |
| 1.618 | Primera resistencia arriba de B | ~$2,618 |
| 2.618 | Segunda resistencia | ~$3,618 |

> El precio tiende a "rebotar" entre canales. Si está entre el 1.0 y el 1.618, lo normal es que siga hasta el 1.618 antes de corregir.

---

## ¿Para qué sirve en la práctica?

- Estás en un **trade largo** en ETH. El precio está en el canal 1.0 ($1,500).
- El Fibonacci Channel te dice que el próximo objetivo natural es el canal 1.618 (~$2,618).
- Pones tu take profit ahí.
- Si el precio rompe el 2.618, la tendencia está muy fuerte y puedes re-entrar.

---

## Analogía

Imagina que una pelota rueda cuesta arriba en una escalera inclinada. Cada escalón es un canal de Fibonacci. La pelota sube un escalón, descansa un poco, y sigue al siguiente. El Fibonacci Channel dibuja esos escalones con anticipación.

---

## Relacionado
- [[Trading/Fibonacci Retracement|Fibonacci Retracement]] — los niveles de pullback dentro de esos canales.
- [[Trading/Trend-Based Fib Extension|Trend-Based Fib Extension]] — proyectar hasta dónde llega el movimiento fuera del canal.
- [[Trading/Elliott Impulse Wave|Elliott Impulse Wave]] — las ondas de impulso 1, 3, 5 suelen ir de canal en canal.

---

## ↩️ Volver
- [[Trading/Trading|📊 Trading Hub]]
