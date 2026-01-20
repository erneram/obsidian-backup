Las métricas de evaluación permiten medir el rendimiento de un modelo de Machine Learning y determinar qué tan bien generaliza a datos nuevos. La métrica adecuada depende del tipo de problema: clasificación, regresión o aprendizaje no supervisado.

---

## Métricas para Modelos de Clasificación

Se utilizan cuando el objetivo es predecir una clase o categoría.

### Accuracy (Exactitud)

Mide la proporción de predicciones correctas sobre el total de predicciones realizadas.

```formula
Accuracy = (TP + TN) / (TP + TN + FP + FN)
```
Es fácil de interpretar, pero puede ser engañosa cuando las clases están desbalanceadas.a

---

### Precision (Precisión)

Indica cuántas de las predicciones positivas fueron realmente correctas.
```formula
Precision = TP / (TP + FP)
```
Es importante cuando el costo de los falsos positivos es alto.

---

### Recall (Sensibilidad)

Mide la capacidad del modelo para encontrar todos los casos positivos reales.
```formula
Recall = TP / (TP + FN)
```
Es clave cuando es crítico no perder casos positivos.

---

### F1 Score

Es la media armónica entre precision y recall.
```formula
F1 = 2 × (Precision × Recall) / (Precision + Recall)
```
Se usa cuando se necesita un balance entre precisión y sensibilidad.

---

### Log Loss

Evalúa la incertidumbre de las predicciones probabilísticas. Penaliza con mayor fuerza las predicciones incorrectas con alta confianza.

---

### Curva ROC y AUC

La curva ROC muestra la relación entre la tasa de verdaderos positivos y la tasa de falsos positivos para distintos umbrales.  
El AUC representa el área bajo la curva ROC.

Un AUC cercano a 1 indica un modelo excelente, mientras que 0.5 representa un modelo aleatorio.

---

### Matriz de Confusión

Es una tabla que resume los resultados de clasificación.

|               | Predicción Negativa | Predicción Positiva |
|---------------|--------------------|--------------------|
| Real Negativo | TN                 | FP                 |
| Real Positivo | FN                 | TP                 |

---

## Métricas para Modelos de Regresión

Se utilizan cuando el modelo predice valores numéricos continuos.

---

### Mean Absolute Error (MAE)

Promedio de los valores absolutos del error entre el valor real y el valor predicho.
```formula
MAE = (1/N) × Σ |y − ŷ|
```
---

### Mean Squared Error (MSE)

Promedio de los errores al cuadrado.
```formula
MSE = (1/N) × Σ (y − ŷ)²
```
Penaliza fuertemente los errores grandes.

---

### Root Mean Squared Error (RMSE)

Raíz cuadrada del MSE.
```formula
RMSE = √MSE
```
Devuelve el error en las mismas unidades que la variable objetivo.

---

### Root Mean Squared Logarithmic Error (RMSLE)

Métrica útil cuando los valores varían en grandes rangos. Penaliza más las subestimaciones.

---

### R² (Coeficiente de Determinación)

Indica qué proporción de la variabilidad de los datos es explicada por el modelo.
```formula
R² = 1 − [Σ(y − ŷ)² / Σ(y − ȳ)²]
```
Un valor cercano a 1 indica un buen ajuste del modelo.

---

## Métricas para Clustering (Aprendizaje No Supervisado)

Se utilizan cuando no existen etiquetas reales.

---

### Silhouette Score

Mide qué tan bien un punto se ajusta a su propio cluster en comparación con otros clusters.

Valores cercanos a 1 indican buena separación; valores cercanos a -1 indican mala asignación.

---

### Davies–Bouldin Index

Evalúa la similitud promedio entre clusters.

Un valor más bajo indica mejores agrupaciones.

---

## Conclusión

Las métricas de evaluación son fundamentales para medir el desempeño real de un modelo de Machine Learning. Elegir la métrica correcta depende del tipo de problema y del objetivo del modelo, ya que una mala elección puede llevar a conclusiones incorrectas sobre su desempeño.
