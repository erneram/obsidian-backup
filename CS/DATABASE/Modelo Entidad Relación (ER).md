MODELO ER: Herramienta para hacer representación de datos estructuradas (UML) estandar del mercado.

Diagrama de flujo (No pegado a un lenguaje)

Componentes clave:

Entidades: TABLAS

Representan conceptos del mundo real. (Clientes, Productos)

Representación grafica de datos

Atributos: Columnas

Columnas como Nombre, correo, genero, edad, etc.

Relaciones:

Asociación entre entidades (Un cliente realiza un pedido)

Claves en modelo ER

PK Atributo único que identifica cada registro

FK Atributo que apunta a clave primaria

**FK hace referencia a PK**

CLAVE COMPUESTA: Llave primaria que se conforma de dos o mas columnas

![IMG_0455.HEIC](attachment:804e72b4-83e8-4cb2-b818-ceb114eb5571:IMG_0455.heic)

Tipos de relaciones

Uno a Uno: (1:1) Un estudiante tiene una carrera

Uno a Muchos: (1:N) una entidad tiene relacionada a muchas pero no alrevez

Muchos a Muchos: (N:M): Muchas instancias están relacionadas con muchas instancias de otra entidad.

Cardinalidad:

Define la cantidad de veces que la instancia esta asociada con otra.

Nos define las reglas de negocio

Que podemos hacer y que no

DIAGRAMA ER:

Representación gráfica que muestra entidades. Es un sistema estandar que no permite ambiguedad.

Atributos derivados. Calculados a partir de otros valores. (Calcular edad a traves de fecha de nacimiento)

Reducir la redundancia: NO almacenar datos que pueden calcularse

Tipos de relaciones

Relaciones fuertes: Identidades con propias claves primarias

Relaciones débiles: Depende de otras para identificarse (Sin papa mueren)

Identificar estas relaciones permite estruvcturar el modelo ER mas preciso

Jerarquias:

Generalización: Combina entidades específicas en una entidad general (carros, motos, lanchas agrupadas en vehiculos)

Especialización: Divide una entidad general en subentidades específicas (Persona puede ser un cliente o un empleado)

Errores comunes:

redundancia de datos , generando inconsistencias

relaciones mal definidas o ausentes

diseño que no escala bien para datos futuros

Notación de Martin o Crow’s Foot

![image.png](attachment:d456b575-898d-452e-bfc4-d2d9e813566e:image.png)