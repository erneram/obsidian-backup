# Controladores: 
- Manejan la logica de las solicitudes HTTP, reciben peticiones, interactúan con los modelos y devuelven respuestas (HTML o JSON).
- Separan la logica de negocio haciendo el código mas ordenado y reutilizables. 
- Se definen acciones listar, crear, almacenar, actualizar o eliminar.


# Modelos: 
- Representan las entidades y tablas de la base de datos. Permite realizar la operación de consulta.
- Permite definir las relaciones entre las tablas.

# Seeders:  
- Se utilizan para llenar la base de datos con datos iniciales o de prueba.
- Utiles en desarrollo pero no en producción.  
- Facilita la inserción de datos de manera manual.  

# Migraciones:  
- Contienen instrucciones para crear, modificar, o eliminar tablas o columnas de la base de datos.
- Facilita la estructura de la base de datos cuando se este trabajando con equipos.
- Asegura un mismo esquema para todos en el equipo.

# Router:  
- Define como se responden las solicitudes HTTP.
- Cada ruta especifica URL, el método HTTP (GET, POST, DELETE) y la acción a ejecutar (controlador).

# Resourses:  
- API Resources: clases que transforman y formatean la respuesta cuando se devuelve la respuesta en un JSON.
- Blade Components: Plantillas reutilizables para fragmentos de interfaz (botones, inputs, tablas).

# Authentication:  
- Gestiona proceso de iniciar sesión, registrar usuarios, cerrar sesión.  
- Se utiliza el recurso Breeze

# Relations:  
- Establece como se relacionan y conectan entre si los modelos, por ende, las tablas en la base de datos. (1:N) (N:M).

## ↩️ Volver al HOME
- [[HOME|🏠 HOME del Vault]]