# De Lenguaje Regular a AFD

## (Construcción directa usando árbol sintáctico y funciones de posición)

## 📌 Contexto de la clase

En esta clase vimos cómo convertir una **expresión regular** en un **Autómata Finito Determinista (AFD)** utilizando:

- Árbol sintáctico
    
- PrimeraPos (firstpos)
    
- ÚltimaPos (lastpos)
    
- SiguientePos (followpos)
    
- Construcción de tabla de estados
    

Este método es el que se usa para construir analizadores léxicos en compiladores.

---

# 1️⃣ Idea general del algoritmo

El objetivo es:

```
Expresión Regular  →  Árbol Sintáctico  →  Tabla de posiciones  
→  Tabla de estados  →  AFD
```

Este método construye directamente un AFD (sin pasar por AFN).

---

# 2️⃣ Paso 1: Construir el árbol sintáctico

Primero:

- Se toma la expresión regular.
    
- Se le agrega el símbolo especial `#` al final.
    
- Se construye el árbol sintáctico.
    

Cada hoja del árbol:

- Representa un símbolo del alfabeto.
    
- Tiene un número de posición único.
    

Ejemplo conceptual:

```
(a|b)*a#
```

Cada símbolo tiene una posición numerada.

---

# 3️⃣ Conceptos importantes

## 🔹 PrimeraPos(n)

Es el conjunto de posiciones que pueden aparecer primero en el subárbol del nodo `n`.

---

## 🔹 ÚltimaPos(n)

Es el conjunto de posiciones que pueden aparecer al final del subárbol del nodo `n`.

---

## 🔹 SiguientePos(i)

Para una posición `i`, indica qué posiciones pueden venir inmediatamente después en la cadena.

Esta tabla es CLAVE para construir el AFD.

---

# 4️⃣ Reglas importantes (las que se aplicaron en clase)

## 🔸 Caso concatenación (.)

Si tenemos un nodo:

```
n = hijo1 . hijo2
```

Entonces:

- Para cada posición `i` en ÚltimaPos(hijo1):
    
    - Se agrega PrimeraPos(hijo2) a SiguientePos(i)
        

👉 Esto fue lo que en clase se repetía mucho:

> "Vamos a recorrer el valor de siguiente posición y le agregamos..."

Es exactamente esto:  
**Agregar conjuntos usando unión.**

---

## 🔸 Caso cerradura de Kleene (*)

Si tenemos:

```
n = hijo*
```

Entonces:

- Para cada posición `i` en ÚltimaPos(hijo):
    
    - Se agrega PrimeraPos(hijo) a SiguientePos(i)
        

Esto genera ciclos en el autómata.

---

# 5️⃣ Cuando ya tenemos la tabla de SiguientePos

Aquí empieza la construcción del AFD.

---

# 6️⃣ Construcción del AFD

## 🔹 Estado inicial

Se toma:

```
PrimeraPos(raíz)
```

Ese conjunto será el **estado inicial del AFD**.

En clase decían:

> "Primera posición de la raíz… ese será el estado inicial."

---

## 🔹 Cómo construir las transiciones

Regla general:

Para un estado S y un símbolo `a`:

1. Buscar en S todas las posiciones que correspondan al símbolo `a`.
    
2. Tomar la unión de sus SiguientePos.
    
3. Ese resultado será el nuevo estado.
    

Formalmente:

```
δ(S, a) = Unión de SiguientePos(i)
para todo i ∈ S tal que símbolo(i) = a
```

---

# 7️⃣ Ejemplo conceptual de lo que se hizo en clase

Supongamos:

Estado inicial = {1,2,3}

---

### 🔹 Transición con símbolo `a`

1. Buscamos qué posiciones dentro de {1,2,3} tienen símbolo `a`.
    
2. Supongamos que solo la posición 1 tiene `a`.
    
3. Entonces:
    

```
Nuevo estado = SiguientePos(1)
```

Si SiguientePos(1) = {4}

Entonces:

```
δ({1,2,3}, a) = {4}
```

Y a ese conjunto se le da un nuevo nombre, por ejemplo:

```
Estado 1
```

---

### 🔹 Transición con símbolo `b`

1. Buscamos dentro de {1,2,3} quién tiene símbolo `b`.
    
2. Supongamos que es la posición 2.
    
3. Entonces:
    

```
δ({1,2,3}, b) = SiguientePos(2)
```

Si SiguientePos(2) = {1,2,3}

Entonces el estado se regresa a sí mismo.

En clase dijeron algo como:

> "El estado inicial con B se va a sí mismo."

Eso significa que hay un ciclo.

---

# 8️⃣ Proceso iterativo

Después de crear un nuevo estado:

- Se repite el proceso para ese nuevo estado.
    
- Se calculan sus transiciones con todos los símbolos.
    
- Si aparece un conjunto nuevo → se convierte en nuevo estado.
    
- Si ya existía → solo se conecta.
    

Así se llena la tabla del AFD.

---

# 9️⃣ Estado de aceptación

Un estado es de aceptación si:

- Contiene la posición del símbolo `#`
    

Porque eso indica que la cadena terminó correctamente.

---

# 🔟 Resultado final

Al completar la tabla:

- Esa tabla YA ES el AFD.
    
- Luego se puede dibujar como diagrama.
    

---

# 1️⃣1️⃣ Relación con compiladores

Esto es importante porque:

- Los autómatas construidos aquí
    
- Se usan para crear analizadores léxicos
    
- Que son la primera fase de un compilador
    

En clase mencionaron:

> "Los autómatas nos van a permitir construir un analizador léxico."

Y eso es totalmente correcto.

---

# 1️⃣2️⃣ Resumen mental del algoritmo

Si lo quieres recordar fácil:

```
1. Construyo árbol.
2. Calculo PrimeraPos.
3. Calculo ÚltimaPos.
4. Calculo SiguientePos.
5. Estado inicial = PrimeraPos(raíz).
6. Transiciones = unión de SiguientePos.
7. Estados con # = aceptación.
```

---

# 🧠 Cómo estudiarlo mejor

Para entenderlo bien:

- Haz un ejemplo pequeño.
    
- Numera las hojas.
    
- Haz la tabla de SiguientePos.
    
- Luego construye la tabla de estados.
    

En la siguiente clase seguramente harán:

> Expresión regular → AFD con lápiz y papel en grupos.

---

# ⚠️ Lo más importante de la clase

Lo que más costó entender fue:

👉 Que TODO es unión de conjuntos.  
👉 Que las transiciones salen de la tabla SiguientePos.  
👉 Que los estados del AFD son conjuntos de posiciones.

Cuando eso se entiende, el resto fluye.

---

Si quieres, puedo ahora:

- Hacer un ejemplo completo paso a paso con una expresión concreta
    
- O convertir este contenido en un resumen aún más simplificado para estudiar antes del parcial

# 📌 Ejemplo completo

## Expresión regular:

[  
(a|b)^* a #  
]

Agregamos `#` como símbolo de fin.

---

# 1️⃣ Numeración de hojas

Numeramos las hojas de izquierda a derecha:

|Símbolo|Posición|
|---|---|
|a|1|
|b|2|
|a|3|
|#|4|

---

# 2️⃣ Árbol Sintáctico

La expresión realmente es:

[  
((a|b)^* . a) . #  
]

Árbol:

```
                (.)
              /     \
            (.)      #
           /   \      4
         (*)     a
          |      3
         (|)
        /   \
       a     b
       1     2
```

---

# 3️⃣ Cálculo de FirstPos y LastPos

## 🔹 Nodo (a|b)

- FirstPos = {1,2}
    
- LastPos = {1,2}
    

---

## 🔹 Nodo (a|b)*

Regla de Kleene:

- FirstPos = FirstPos(hijo)
    
- LastPos = LastPos(hijo)
    

Entonces:

- FirstPos = {1,2}
    
- LastPos = {1,2}
    

---

## 🔹 Nodo ((a|b)* . a)

Regla concatenación:

- FirstPos = {1,2} ∪ {3} si el primero es anulable  
    Como `(a|b)*` ES anulable:
    

→ FirstPos = {1,2,3}

- LastPos = {3}
    

---

## 🔹 Nodo raíz (((a|b)* . a) . #)

- FirstPos = {1,2,3}
    
- LastPos = {4}
    

---

# 4️⃣ Tabla FollowPos

Ahora aplicamos reglas.

---

## 🔹 Regla concatenación (.)

Para cada posición en LastPos(hijo1), agregar FirstPos(hijo2).

---

### Nodo ((a|b)* . a)

LastPos((a|b)*) = {1,2}

FirstPos(a) = {3}

Entonces:

FollowPos(1) → agregar {3}  
FollowPos(2) → agregar {3}

---

### Nodo raíz (((a|b)* . a) . #)

LastPos((a|b)* . a) = {3}

FirstPos(#) = {4}

Entonces:

FollowPos(3) → agregar {4}

---

## 🔹 Regla Kleene (*)

Para cada posición en LastPos(hijo), agregar FirstPos(hijo).

LastPos(a|b) = {1,2}  
FirstPos(a|b) = {1,2}

Entonces:

FollowPos(1) → agregar {1,2}  
FollowPos(2) → agregar {1,2}

---

# 📋 Tabla Final FollowPos

|Posición|FollowPos|
|---|---|
|1|{1,2,3}|
|2|{1,2,3}|
|3|{4}|
|4|∅|

---

# 5️⃣ Construcción del AFD

## 🔹 Estado inicial

FirstPos(raíz) = {1,2,3}

Lo llamamos:

[  
S0 = {1,2,3}  
]

---

# 6️⃣ Tabla de Estados

Alfabeto: {a,b}

---

## 🔹 Desde S0 = {1,2,3}

### Transición con 'a'

Posiciones con símbolo 'a' en S0:

- 1
    
- 3
    

Unimos FollowPos(1) ∪ FollowPos(3):

FollowPos(1) = {1,2,3}  
FollowPos(3) = {4}

Unión:

[  
{1,2,3,4}  
]

Nuevo estado:

[  
S1 = {1,2,3,4}  
]

---

### Transición con 'b'

Posiciones con 'b':

- 2
    

FollowPos(2) = {1,2,3}

Entonces:

[  
S2 = {1,2,3}  
]

Pero ese ya es S0.

Entonces:

δ(S0,b) = S0

---

## 🔹 Ahora desde S1 = {1,2,3,4}

### Con 'a'

Posiciones con 'a':

- 1
    
- 3
    

FollowPos(1) = {1,2,3}  
FollowPos(3) = {4}

Unión:

[  
{1,2,3,4}  
]

Se queda en S1.

---

### Con 'b'

Posición con 'b':

- 2
    

FollowPos(2) = {1,2,3}

[  
S0  
]

---

# 📋 Tabla Final del AFD

|Estado|a|b|
|---|---|---|
|S0 {1,2,3}|S1|S0|
|S1 {1,2,3,4}|S1|S0|

---

# 7️⃣ Estados de aceptación

Un estado es de aceptación si contiene la posición del símbolo `#`.

Posición de `#` = 4

Entonces:

- S0 → NO
    
- S1 → SÍ (porque tiene 4)
    

Estado final:

[  
S1  
]

---

# 8️⃣ AFD Final (diagrama conceptual)

```
          a
     S0 ------> S1*
      ^          |
      |          |
      |          a
      |          |
      b          |
      |          |
      +----------+
             b
```

S1 es estado de aceptación.

---
