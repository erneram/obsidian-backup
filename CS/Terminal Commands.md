*APRENDIENDO A UTILIZAR TEXTO :00000000 SIN INTERFAZ GRAFICA*

COMANDOS DE CLI Basicos

1. pwd: imprimir directorio actual (muestra ruta)

2. sudo: superuser do (forzar ?)
3. ls: List content- lista archivos y carpetas en directorio actual (ls -l (detalles como permisos o tamaños) (ls -a lista archivos ocultos)
4. cd: Change directory - cambia de directorio (path absoluto es desde raiz hasta actual) (path relativo es desde la carpeta donde se encuentra)
     al utilizar cd .. sales del directorio actual 
     al utilizar cd otra_carpeta/ entra al directorio
5. mkdir: make directory
6. touch: crea archivos
7. cp: copiar nuevo archivo “cp nuevo_archivo.text otra_carpeta/nuevo.txt”
8. mv: Mover archivos o renombrar archivos “mv archivo.txt ../archivo_movido.txt”
9. rm: remove (elimina carpetas que eliminan para siempre algo)
    1. rm archivo.txt elimina el archivo
    2. rm -r carpeta elimina carpetas con contenido adentro
10. cat: imprimir contenido de un archivo (cat archivo.txt
11. head: imprime los primeros 10 registros (head archivo.txt)
12. tail: Muestra ultimas lineas del archivo (tail archivo.txt)
13. vi arhivo.txt: Editar archivo de texto (escribir)
14. :wq: write it and quit
15. nano archivo.txt es mas facil

PERMISOS y PROPIETARIOS

1. chmod: cambia los permisos de archivo (PEGRILOSOOO)|
    1. 755: Lectura, escritura y ejecucion para el propieratrio, solo lectura y ejecucion para otros
    2. 644: lectura y escritura para propietario, solo lectura para otros
    3. 777: Lectura, escritura y ejecución para todos
    4. chmod 777 archivo.txt
2. chown: Change owner
    1. chown usuario archivo.txt