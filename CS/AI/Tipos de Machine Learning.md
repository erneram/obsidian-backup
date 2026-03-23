## 🔍 Tipos principales de Machine Learning

Los algoritmos de Machine Learning se agrupan en **cinco categorías principales**, cada una con características y usos distintos:

---

## 1. Aprendizaje Supervisado

El aprendizaje supervisado entrena modelos con datos **etiquetados**, donde se conoce la respuesta que se espera que el modelo prediga.

**Cómo funciona:**
- El modelo aprende a mapear entradas a salidas correctas.
- Se utiliza cuando se necesita predecir resultados específicos.

**Ejemplos de algoritmos:**
- **Regresión** — predice valores continuos, como precios o temperaturas.
- **Clasificación** — predice categorías, como “spam” o “no spam”.
- **Clasificadores Naïve Bayes** — métodos generativos para tareas de clasificación.
- **Redes neuronales** — imitan el cerebro humano para tareas complejas.
- **Random Forest** — combina múltiples árboles de decisión para mejorar predicciones.

**Usos comunes:**
- Detección de fraude
- Análisis predictivo
- Reconocimiento de imágenes

---

## 2. Aprendizaje No Supervisado

Este tipo trabaja con datos **sin etiquetas**. El objetivo es encontrar patrones ocultos o estructuras en los datos.

**Cómo funciona:**
- El algoritmo intenta aprender la distribución de los datos sin un objetivo específico.
- Se usa para descubrir relaciones no evidentes.

**Ejemplos de métodos:**
- **Clustering** (agrupamiento) — como K-means, que agrupa datos similares.
- **Asociación** — identifica relaciones entre variables en grandes conjuntos de datos.
- **Análisis de componentes principales (PCA)** — reduce la dimensionalidad de los datos.

**Usos comunes:**
- Segmentación de clientes
- Detección de anomalías
- Visualización de datos

---

## 3. Aprendizaje Autosupervisado

Este enfoque permite a los modelos entrenarse con **datos no etiquetados** generando etiquetas automáticamente.

**Cómo funciona:**
- El algoritmo predice una parte del dato a partir de otra parte.
- Convierte problemas no supervisados en tareas que parecen supervisadas al crear etiquetas internas.

**Por qué es útil:**
- Particularmente valioso cuando no hay suficientes datos etiquetados.
- Muy usado en visión artificial y procesamiento de lenguaje natural.

---

## 4. Aprendizaje por Refuerzo

En este tipo de aprendizaje, un **agente** aprende a tomar decisiones interactuando con un entorno.

**Cómo funciona:**
- El agente recibe **recompensas o penalizaciones** según sus acciones.
- Aprende a maximizar las recompensas tomando mejores decisiones.

**Ejemplos de uso:**
- Entrenamiento de robots
- Juegos y simulaciones
- Vehículos autónomos

---

## 5. Aprendizaje Semisupervisado

El aprendizaje semisupervisado combina características de supervisado y no supervisado.

**Cómo funciona:**
- Se entrena con una pequeña parte de datos etiquetados y una gran parte de datos sin etiquetar.
- El componente etiquetado guía el aprendizaje general.

**Ejemplo avanzado:**
- **Redes Generativas Adversarias (GANs)** — usan dos redes para generar datos y mejorar la calidad del aprendizaje incluso con pocos datos etiquetados.

---

## 🧠 Resumen de Tipos

| Tipo de aprendizaje | Necesita etiquetas              | Uso principal                           |
| ------------------- | ------------------------------- | --------------------------------------- |
| Supervisado         | Sí                              | Predicción con objetivo definido        |
| No supervisado      | No                              | Descubrir patrones en datos             |
| Autosupervisado     | No                              | Entrenar a partir de datos crudos       |
| Refuerzo            | No (evaluación por recompensas) | Aprender estrategias                    |
| Semisupervisado     | Parcialmente                    | Combinar aprendizaje guiado y no guiado |

---

## 📌 Conclusión

Los distintos tipos de Machine Learning ofrecen herramientas adaptadas a diferentes tipos de problemas y datos. Elegir el tipo correcto depende de la naturaleza de los datos, la cantidad de etiquetas disponibles y el objetivo del proyecto. El avance de estos métodos ha ampliado enormemente las aplicaciones de la IA y continúa transformando industrias completas.

---


---

## ↩️ Navegación
- [[CS/AI/ML|🤖 ML Hub]] → [[CS/CS|💻 CS]] → [[HOME|🏠 HOME]]
