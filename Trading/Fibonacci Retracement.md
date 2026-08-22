---
title: Fibonacci Retracement
tags: [trading, fibonacci, technical-analysis]
---

# 📐 Fibonacci Retracement

## ¿Qué es?

El precio **nunca sube o baja en línea recta**. Sube, descansa (pullback), y luego sigue. El Fibonacci Retracement te dice **dónde es probable que descanse** antes de continuar.

Los niveles vienen de la secuencia de Fibonacci (0, 1, 1, 2, 3, 5, 8, 13, 21…). Las proporciones entre esos números generan los porcentajes: 23.6%, 38.2%, 50%, 61.8%, 78.6%.

---

## ¿Cómo se traza? — La dirección importa todo

La herramienta Fibonacci Retracement tiene un punto de inicio y un punto final. **El orden en que los pones determina dónde aparecen los niveles.**

La regla es simple:

> Siempre trazas **desde donde empezó el movimiento** hacia **donde terminó**. Los niveles aparecen en el espacio que el precio tiene que recorrer de regreso.

---

### Tendencia alcista (uptrend) → trazas de ABAJO hacia ARRIBA

El precio subió. Quieres saber hasta dónde va a bajar antes de seguir subiendo.

**Acción:** Haz clic en el mínimo (punto A, abajo) → arrastra hacia el máximo (punto B, arriba).

```
Punto B = $20,000  ← terminas aquí (arrastras hasta acá)
  |  78.6% → $12,140
  |  61.8% → $13,820  ← golden ratio
  |  50.0% → $15,000
  |  38.2% → $16,180
  |  23.6% → $17,640
Punto A = $10,000  ← empiezas aquí (primer clic)
```

Los niveles quedan **entre A y B**, es decir, en la zona de caída. Eso es correcto: son los soportes donde el precio puede detenerse mientras baja.

---

### Tendencia bajista (downtrend) → trazas de ARRIBA hacia ABAJO

El precio bajó. Quieres saber hasta dónde va a subir (rebote) antes de seguir bajando.

**Acción:** Haz clic en el máximo (punto A, arriba) → arrastra hacia el mínimo (punto B, abajo).

**Ejemplo: BTC cae de $30,000 a $18,000.**

```
Punto A = $30,000  ← empiezas aquí (primer clic)
  |  23.6% → $21,832
  |  38.2% → $22,584
  |  50.0% → $24,000
  |  61.8% → $25,416  ← golden ratio (resistencia más fuerte)
  |  78.6% → $27,432
Punto B = $18,000  ← terminas aquí (arrastras hasta acá)
```

Los niveles quedan **entre A y B**, en la zona de rebote. Son las resistencias donde el precio puede frenarse mientras sube.

---

### Resumen de dirección

| Tendencia | Primer clic (A) | Segundo clic (B) | Los niveles son |
|-----------|----------------|-----------------|----------------|
| **Alcista** | Mínimo (abajo) | Máximo (arriba) | Soportes (el precio cae hacia ellos) |
| **Bajista** | Máximo (arriba) | Mínimo (abajo) | Resistencias (el precio sube hacia ellos) |

---

### ¿Qué pasa si lo trazas al revés?

Los niveles aparecen fuera del rango del movimiento, proyectados hacia abajo del mínimo (en alcista) o hacia arriba del máximo (en bajista). Eso no tiene sentido como retracement. Si ves los niveles fuera del swing, invierte el orden de los puntos.

---

### En TradingView

1. Selecciona la herramienta **"Fib Retracement"** (icono de regla con niveles).
2. **Uptrend:** clic en el mínimo del swing → arrastra y suelta en el máximo.
3. **Downtrend:** clic en el máximo del swing → arrastra y suelta en el mínimo.
4. Los niveles aparecen automáticamente en el medio.

---

---

## Ejemplo concreto: BTC de $10,000 a $20,000

El movimiento total fue **$10,000** (de $10k a $20k).

| Nivel Fib | Cálculo | Precio de soporte |
|-----------|---------|-------------------|
| 23.6% | $20,000 − ($10,000 × 0.236) | ~$17,640 |
| 38.2% | $20,000 − ($10,000 × 0.382) | ~$16,180 |
| 50.0% | $20,000 − ($10,000 × 0.500) | ~$15,000 |
| **61.8%** | $20,000 − ($10,000 × 0.618) | **~$13,820** ← golden ratio |
| 78.6% | $20,000 − ($10,000 × 0.786) | ~$12,140 |

> El 61.8% es el nivel más respetado en los mercados. Se llama el **"golden ratio"** o la razón áurea. Si el precio mantiene este nivel, la tendencia alcista está intacta.

---

## ¿Qué esperas ver?

Después del impulso de $10k → $20k, el precio cae. Si se detiene en el 61.8% ($13,820) y rebota, eso es una **señal de compra**. Buscas:
- Vela de reversión (pin bar, engulfing).
- Volumen bajo en la caída (los vendedores se agotan).
- Volumen alto al rebotar (compradores llegan).

---

## Analogía

Imagina que empujas un columpio muy fuerte hacia adelante. Antes de que llegue al otro lado, el columpio regresa un poco. Ese "regresa un poco" es el retracement. Los niveles de Fibonacci te dicen **hasta dónde suele regresar** el columpio antes de seguir yendo hacia adelante.

---

## Errores comunes

| Error | Por qué está mal |
|-------|-----------------|
| Trazar el Fib en movimientos pequeños o laterales | Los niveles pierden significado sin un swing claro |
| Asumir que el precio SIEMPRE respetará el nivel | Es una zona de probabilidad, no garantía |
| Ignorar el contexto del mercado | Un Fib en tendencia alcista fuerte es más confiable que en rango |

---

## Relacionado
- [[Trading/Trend-Based Fib Extension|Trend-Based Fib Extension]] — para proyectar targets después del rebote.
- [[Trading/Fibonacci Channel|Fibonacci Channel]] — para proyectar rangos paralelos a la tendencia.
- [[Trading/Elliott Correction Wave|Elliott Correction Wave]] — el retracement suele ocurrir en las ondas correctivas ABC.

---

## ↩️ Volver
- [[Trading/Trading|📊 Trading Hub]]
