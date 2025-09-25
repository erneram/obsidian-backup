Es una librería que actúa como "puente" entre el backend y el frontend, permitiendo construir aplicaciones de una sola página (SPAs) usando el framework de servidor (laravel) sin tener que construir una API REST completa.

- Sin API separada:
	  Con inertia, los controladores y rutas del servidor siguen funcionado como siempre pero en lugar de responder con vistas HTML o JSON, devuelven "props" que son consumidos por componentes de un framework moderno como (Vue, React, Svelte)
	  
- Transición a SPA:
	  Permite que la aplicación se comporte como un SPA (Sin recargar la página) mientras se sigue usando la lógica de routing y controladores del lado del servidor. 
	  
- Integración sencilla:
	  Se encarga de manejar la navegación y el estado de la aplicación en el cliente, enviando actualizaciones necesarias sin que se gestionen manualmente llamadas AJAX o API separada.

***
Es un integración porque no tiene un framework frontend independiente ni un framework de backend si no que conecta ambos mundos.
***

## ↩️ Volver al HOME
- [[HOME|🏠 HOME del Vault]]