***
### Herramienta moderna para desarrollo frontend que actúa como servidor como empacador (bundler) para producción

- **Servidor de desarrollo rápido**: 
  Aprovecha la capacidad de los navegadores modernos para trabajar con módulos ES nativos.
  En lugar de empaquetar todos los archivos a la vez, Vite sirve los archivos de forma individual y actualiza los módulos modificados casi instantáneamente. Mejora los tiempos de arranque
- **HMR (Hot Module Replacement)**: 
  Vite ofrece HRM nativo, permite que los cambios en el código se reflejen en el navegador sin necesidad de recargar toda la página. Esto es permite mantener el estado de la aplicación mientras se hacen cambios.
- **Empaquetado para producción**: 
  Cuando llega el momento de desplegar, Vite usa herramientas como "Rollup" para empaquetar, optimizar y minificar los recursos (JS, CSS, IMG, etc.) para producción.
***
# Funcionamiento:

1. Durante desarrollo: 
	1. Carga directa de módulos:
	   Vite sirve los archivos JS directamente al navegador como módulos ES, eliminando la etapa de empaquetado en cliente.
	2. Transformaciones sobre la marcha:
	   Si se usan preprocesadores como Typescript o SASS, Vite los transforma en tiempo real.
	3. HMR: 
	   Cuando modificas un archivo, Vite actualiza únicamente el módulo modificado, sin recargar toda la aplicación.
2. En Producción:
	1. Compilación:
	   Al ejecutar comandos como ```vite build``` o  ```npm run build```, vite empaqueta todos los módulos en uno o varios archivos optimizados.
	2. Optimización:
	   Se aplican técnicas de tree-shaking, minificación y división de código para generar recursos que se carguen rápidamente.

***