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
		
2. Cadena o palabra
	Secuencia finita de simbolos de un alfabeto
	01101 => palabra generada => ∑ = { 0, 1 }
	a, b , c, ab, cb ... => generada por => ∑ = { a, b, c }
	Epsilon (ε) es la palabra vacia, tiene cero simbolos de algun alfabeto
	
3. Lenguaje 