1. Software que en su ore son automatas, entienten circuitos digitales
2. En un compilador, utiliza un analizador lexico (Automata finito)
3. Procesar largas cantidades de texto (Automatas finitos)
4. Modela una maquina de maquina de estados finitos

Maquina de estados finitos, modela protocolos de comunicación. 
![[Pasted image 20250702200117.png]]

1. Alfabeto
	Conjunto finito (no vacio) de simbolos SETS o ∑
	∑ = { 0, 1 }
	∑ = { a, b ,c , ... }
	Conjunto de caracteres ASCII (Hex o Decimal) Codificación del caracter

	Potencia de un alfabeto 
	k es el numero de palabras que tengan longitud 2
	∑^k = { w, |w| = k }
	Cerradura Estrella
	∑^* = ∑^0 U ∑^1 U ∑^2 ....
		
2. Cadena o palabra
	Secuencia finita de simbolos de un alfabeto
	01101 => palabra generada => ∑ = { 0, 1 }
	a, b , c, ab, cb ... => generada por => ∑ = { a, b, c }
	Epsilon (ε) es la palabra vacia, tiene cero simbolos de algun alfabeto
	|w| nos dice la longitud de una cadena

	# Concatenación
	De alfabeto cualquiera  ∑ = { a, b }
		x = aba
		y = bab
		xy = ababab
	
3. Lenguaje
	Un lenguaje sobre ∑ es cualquier subconjunto de ∑
	∑ = { a, b }
	Cerradura Kleene sobre alfabeto
	∑^* = { ε, a, b, aa, ab, bb, ba, ... }
		L = { aabb, babb } subconjunto construido a partir de la primera ∑
	Operaciones de lenguajes
	1. Union
	2. Concatenación
	3. Cerradura Kleene sobre lenguaje
	4. Cerradura Positiva

# Problemas vs Lenguajes
---
Problema: 
Tener un conjunto de simbolos "∑", tomar una palabra, Contruyo un lenguaje "L",
El problema es determinar si una palabra pertenece o no a este lenguaje "L"


# Expresiones Regulares
---
Se construye a partir de un lenguaje
L = {a, b, c, ... , z} & D = { 0, 1, 2, ..., 9 }
L U D = Letras y digitos
LD = 520 palabras de longitud 2
L^* = Palabras infinitas, lenguajes, etc

L(LUD)^* = Variables que empiezan con una letra seguido de cualquier cosa

Expresión regular
Los conjuntos contruyen mas expresiones regulares
a => L(a) = { 'a' }
b => L(b) = { 'b' }
a|b => L(a|b) => L(a) U L(b) = { 'a', 'b' }
ab => L(a) * L(b) = { 'ab' }
a* => L(a)^* => { ε, a, aa, aaa, aaaa, .....}

Ser asociativo por la izquierda representa el orden de la presedencia


Expresion regular es notacion entre lenguajes que definen palabras,
esa expresión regular, representa de forma compacta cual es la forma que tienen todas esas palabras


Problema: 
![[EC859E85-8F6F-4914-BBD9-100BD25B544F.jpeg]]

# Automatas finitos
---
Maquina o reconocedor que entra una cadena "string" y responde con "yes" o "no"
- Determinista: 
	- No tiene restricciones
	- Rapido
- No Determinista: 
	- Tiene restricciones
	- Es más facil de determinar
Mismo nivel de reconocimiento de Lenguajes Regulares (e.r.)

## Determinista
Q: Conjunto de estados
Σ: Conjunto finito de estados 
Función de transición  
Q0: Estado inicial
F: Set de estados aceptados

Una maquina que procesa palabras según un conjunto de reglas preestablecidas

Leer una cadena implica ir revisando la cadena uno por uno
![[Pasted image 20250702200117.png]]
W = "110100" (hacer split masomenos)
Ver estado iniciar (q1) y agarrar primer elemento "1", ver flechas y a donde es que lleva, si repite q1 o lleva a q2, siendo estas el (estado final). Ir haciendolo hasta terminar la cadena.
La cadena pertenece cuando llega al estado final. Respondiendo "Yes" o "No"