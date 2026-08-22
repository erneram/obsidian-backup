---
title: Trend-Based Fib Extension
tags: [trading, fibonacci, technical-analysis]
---

# 📐 Trend-Based Fibonacci Extension

## ¿Qué es?

El Fibonacci Retracement te dice **dónde para el pullback**. La Trend-Based Fib Extension te dice **hasta dónde llega el precio después** de ese pullback.

Mientras el retracement mira hacia atrás ("¿cuánto retrocede?"), la extensión mira hacia adelante ("¿cuánto avanza después?").

---

## Cómo se traza — La dirección importa todo

Necesitas **3 puntos** y el orden en que los colocas determina si los targets se proyectan hacia arriba o hacia abajo.

> La regla: A→B marca el impulso original. C marca dónde terminó la corrección. Los targets se proyectan **más allá de B**, en la misma dirección que A→B.

---

### Tendencia alcista → A abajo, B arriba, C en medio

El precio subió (A→B), corrigió (B→C), y quieres saber hasta dónde sube en el próximo impulso.

**Orden de clics:**
1. **Punto A** → mínimo del swing (clic aquí primero, abajo).
2. **Punto B** → máximo del swing (clic aquí, arriba).
3. **Punto C** → fin del pullback, entre A y B (clic aquí).

```
                         ─── 2.618 (~$41,180)
                         ─── 2.0   (~$35,000)
                         ─── 1.618 (~$31,180)  ← target clásico
                         ─── 1.0   (~$25,000)
                         ─── 0.618 (~$21,180)
          B = $20,000   ─── 0.0   (nivel base = B)
         /
        /
       C = $15,000  ← tercer clic (aquí terminó la corrección)
      /
     A = $10,000   ← primer clic
```

Los targets aparecen **por encima de B**. Eso es correcto: son los precios hacia donde irá el próximo impulso alcista.

---

### Tendencia bajista → A arriba, B abajo, C en medio

El precio bajó (A→B), rebotó (B→C), y quieres saber hasta dónde cae en el próximo impulso bajista.

**Orden de clics:**
1. **Punto A** → máximo del swing bajista (clic aquí primero, arriba).
2. **Punto B** → mínimo del swing bajista (clic aquí, abajo).
3. **Punto C** → fin del rebote, entre A y B (clic aquí).

**Ejemplo: BTC cae de $30,000 → $18,000, rebota a $22,000 (punto C).**

```
A = $30,000  ← primer clic
      \
       C = $22,000  ← tercer clic (aquí terminó el rebote)
        \
         B = $18,000  ← segundo clic
              ─── 0.0   (nivel base = B)
              ─── 0.618 (~$14,182)
              ─── 1.0   (~$10,000)
              ─── 1.618 (~$4,364)   ← target clásico bajista
```

Los targets aparecen **por debajo de B**. Son los precios donde el siguiente impulso bajista podría frenar.

---

### Resumen de dirección

| Tendencia | Punto A | Punto B | Punto C | Targets proyectados |
|-----------|---------|---------|---------|---------------------|
| **Alcista** | Mínimo (abajo) | Máximo (arriba) | Fin del pullback (entre A y B) | Por encima de B |
| **Bajista** | Máximo (arriba) | Mínimo (abajo) | Fin del rebote (entre A y B) | Por debajo de B |

---

### ¿Qué pasa si pones C fuera del rango A→B?

Si C queda **por encima de B** en un uptrend (es decir, el precio hizo un nuevo máximo antes de corregir), la estructura del swing es diferente y debes reidentificar los puntos. Si C queda **por debajo de A** en un uptrend, la corrección invalidó el swing completo y la extensión no aplica.

---

### En TradingView

1. Selecciona **"Trend-Based Fib Extension"** en las herramientas de dibujo.
2. **Uptrend:** clic en el mínimo (A) → clic en el máximo (B) → clic en el fin del pullback (C).
3. **Downtrend:** clic en el máximo (A) → clic en el mínimo (B) → clic en el fin del rebote (C).
4. Los niveles 0.618, 1.0, 1.618, 2.618 aparecen automáticamente más allá de B.

---

---

## Ejemplo concreto: BTC de $10k → $20k → $15k

- A = $10,000 (fondo del swing)
- B = $20,000 (techo del swing)
- C = $15,000 (pullback del 50% → Fib retracement)

El movimiento A→B fue de **$10,000**. La extensión proyecta desde C:

| Nivel de extensión | Cálculo | Precio proyectado |
|-------------------|---------|-------------------|
| 0.618 | $15,000 + ($10,000 × 0.618) | ~$21,180 |
| 1.0 | $15,000 + ($10,000 × 1.000) | ~$25,000 |
| **1.618** | $15,000 + ($10,000 × 1.618) | **~$31,180** ← target clásico |
| 2.0 | $15,000 + ($10,000 × 2.000) | ~$35,000 |
| 2.618 | $15,000 + ($10,000 × 2.618) | ~$41,180 |

> El **1.618** es el target más usado y respetado. Si el mercado está en una tendencia alcista fuerte, el precio tiende a llegar al 1.618 antes de la siguiente corrección significativa.

---

## Diferencia entre Extension y Retracement

```
A ──────────────────── B  (impulso)
                        \
                         C  (corrección)
                          \
                   Extension targets más allá de B:
                           ─── 0.618
                           ─── 1.0
                           ─── 1.618  ← aquí vas a poner tu take profit
```

---

## Flujo de trading completo con Fib

1. Ves un impulso fuerte de A ($10k) a B ($20k).
2. Esperas el pullback hasta el 61.8% de Fib Retracement ($13,820).
3. El precio rebota desde $13,820 con una vela alcista fuerte.
4. Entras largo en ~$14,000.
5. Pones tu take profit en el 1.618 de la Trend-Based Extension (~$31,180).
6. Stop loss por debajo del 78.6% del retracement (~$12,140).

Relación riesgo/beneficio: arriesgas ~$2,000 para ganar ~$17,000. **R/R de 1:8.5**.

---

## Cuándo NO funciona

- En mercados sin tendencia clara (rangos laterales). Sin impulso A→B limpio, la extensión no tiene base.
- Si el precio rompe el 100% de Fib Retracement (vuelve al punto A), la estructura del swing quedó inválida.

---

## Analogía

Imaginas que lanzas una pelota hacia arriba (A→B). La pelota baja un poco (B→C). Ahora quieres saber **hasta dónde va a subir** la próxima vez que la lances con esa energía. La Trend-Based Extension te da esa respuesta basada en las proporciones de la naturaleza (Fibonacci).

---

## Relacionado
- [[Trading/Fibonacci Retracement|Fibonacci Retracement]] — el pullback (punto C) se ubica con retracement.
- [[Trading/Elliott Impulse Wave|Elliott Impulse Wave]] — la extensión suele coincidir con el target de la onda 3 o 5.
- [[Trading/Supply and Demand|Supply & Demand]] — combinar zonas de oferta con niveles de extensión mejora la precisión.

---

## ↩️ Volver
- [[Trading/Trading|📊 Trading Hub]]
