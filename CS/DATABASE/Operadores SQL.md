# Group By
Agrupa filas con los mismos valores en columnas específicas y permite aplicar funciones de agregación
> generar reportes agrupados como calculos de totales etc.

```
SELECT 
	categoria, 
	SUM(ventas) 
FROM productos 
GROUP BY categoria;
```

# ORDER BY
Ordena los resultados de una consulta en orden ascendente o descendente según una o más columnas.
>Mostrar datos ordenados, listar productos desc o asc (asc default)

```
SELECT nombre, ventas FROM productos ORDER BY ventas DESC;
```

# Funciones de Agregación
1. SUM()
	Calcula la suma total de una columna numérica.
```
SELECT SUM(salario) FROM empleados;
```
2. COUNT()
	Cuenta la cantidad de registros en un conjunto de datos.
```
SELECT COUNT(*) FROM empleados;
```
3. AVG()
	Calcula el promedio de los valores en una columna.
```
SELECT AVG(edad) FROM empleados;
```
4. MIN() / MAX()
	Encuentra el valor mínimo o máximo en un conjunto de datos.
```
SELECT MIN(salario), MAX(salario) FROM empleados;
```

# UNION
Combina los resultados de múltiples consultas en un solo conjunto de datos, eliminando duplicados
>Fusionar información de diferentes fuentes, listar empleados de distintas regiones
```
SELECT 
	nombre 
FROM clientes_usa 
UNION 
SELECT 
	nombre 
FROM clientes_mexico;
```

# UNION ALL
Similar a `UNION`, pero mantiene duplicados en los resultados.
>Necesita combinar conjuntos de datos pero sin perder información. 
```
SELECT 
	nombre 
FROM clientes_usa 
UNION 
SELECT 
	nombre 
FROM clientes_mexico;

```

# WITH
Crea una tabla temporal dentro de la consulta para mejorar la legibilidad y optimización.
>Calcular valores intermedios como promedio salarial y luego filtrar los empleados con salario superior
```
WITH salario_promedio AS ( 
	SELECT 
		AVG(salario) AS promedio 
	FROM empleados ) 
SELECT 
* 
FROM empleados 
WHERE salario > 10000;
```

# CASE WHEN THEN
Permite evaluar condiciones dentro de una consulta y devolver diferentes resultados en función de dichas condiciones.
>Crear nuevas categorías y manejar valores condicionales o transformar datos en consultas (if)
```
SELECT nombre, salario,
    CASE 
        WHEN salario > 5000 THEN 'Alto'
        WHEN salario BETWEEN 3000 AND 5000 THEN 'Medio'
        ELSE 'Bajo'
    END AS categoria_salarial
FROM empleados;
```
			

## ↩️ Volver al HOME
- [[HOME|🏠 HOME del Vault]]