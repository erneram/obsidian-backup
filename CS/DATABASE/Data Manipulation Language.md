## Select
 Selecciona datos de una tabla especifica
#### ej1
```
select * from empleados;
```
#### ej2 (filtrando filas)
```
SELECT nombre, salario 
FROM empleados 
WHERE cargo = 'Gerente' 
ORDER BY salario DESC;
```
 cuando se utiliza un %Gerente%  hace que se filtre con todo lo que hay antes o despues, pero si le importa el case (lower o upper)
#### ej3 (consultar con JOIN)
```
SELECT e.nombre, e.cargo, d.nombre AS departamento 
FROM empleados as e 
JOIN departamentos as d ON e.departamento_id = d.id;
```


## Insert
Se usa para insertar nuevos registros en una tabla.

#### ej1 
```
INSERT INTO empleados (nombre, cargo, salario) VALUES ('Juan Pérez', 'Desarrollador', 3500.00);
```
#### ej2 (agregando mas de uno)
```
INSERT INTO empleados (nombre, cargo, salario) 
VALUES 
	('Ana López', 'Gerente', 5000.00), 
	('Carlos Gómez', 'Analista', 3200.00);
```
#### ej3 (agrega registros basados en una consulta)
```
INSERT INTO empleados (nombre, cargo, salario) 
SELECT nombre, 'Nuevo Cargo', salario * 1.1 
FROM empleados WHERE cargo = 'Analista';
```

## Update
Permite modificar registros existentes en una tabla.

#### ej1
 ```
 UPDATE empleados SET salario = 4000 WHERE nombre = 'Juan Pérez';
 ```
#### ej2 (update con algo especifico)
```
UPDATE empleados SET salario = salario * 1.05 
WHERE cargo = 'Analista'; 
```
#### ej3 ()
```
UPDATE empleados 
SET salario = 
	(SELECT AVG(salario) 
		FROM empleados 
		WHERE cargo = 'Gerente' 
	) 
WHERE cargo = 'Analista';
```

## Delete
Hacer un SELECT antes de hacer un DELETE
Se usa para eliminar registros de una tabla.
No pregunta solo ejecuta
#### ej1
```
DELETE FROM empleados 
WHERE nombre = 'Juan Pérez';
```
#### ej2
```
DELETE FROM empleados 
WHERE cargo = 'Analista';
```
#### ej3
```
DELETE FROM empleados 
WHERE departamento_id IN 
	(SELECT id 
		FROM departamentos 
		WHERE nombre = 'Ventas' 
	);
```

