## title: CLI básica (sin interfaz gráfica)  
tags: [cli, linux, mac, terminal, bash, zsh]

> [!summary] En una línea  
> Navega, crea/mueve archivos, busca con `grep/find`, usa tuberías `|` y redirecciones `>`, gestiona permisos con `chmod/chown`, procesos con `ps/top/kill`, comprime con `tar`, transfiere con `scp/rsync`, y automatiza con alias e historial.

---

# 0) Conceptos clave

- **Ruta absoluta**: desde `/` (raíz) → `/home/user/proyecto`.
    
- **Ruta relativa**: desde el directorio actual → `../imagenes`.
    
- **Globbing**: `*` (cualquier cosa), `?` (un carácter), `[abc]` (clase).
    
- **Pipelines**: `comando1 | comando2` (la salida del 1 entra al 2).
    
- **Redirecciones**: `>` sobrescribe, `>>` agrega, `2>` errores, `&>` todo.
    

---

# 1) Navegación y listado

```bash
pwd                    # ruta actual
ls                     # lista archivos
ls -la                 # con ocultos y detalles
cd carpeta/            # entrar
cd ..                  # salir al padre
cd -                   # volver al directorio anterior
pushd carpeta && popd  # pila de directorios (ir/volver)
```

---

# 2) Archivos y directorios

```bash
mkdir nueva           # crear carpeta
mkdir -p a/b/c        # crear anidado
touch archivo.txt     # crear/actualizar marca de tiempo
cp origen dest        # copiar archivo
cp -r dir1 dir2       # copiar carpeta recursivo
mv a.txt b.txt        # mover/renombrar
rm archivo.txt        # eliminar archivo
rm -r carpeta         # eliminar carpeta y contenido
rm -ri carpeta        # interactivo (más seguro)
rmdir vacia           # eliminar carpeta vacía
ln archivo enlace     # hard link
ln -s /ruta/real link # symlink
file archivo          # tipo de archivo
```

> [!warning] Cuidado con `rm -rf`  
> `rm -rf /` o rutas mal escritas pueden borrar TODO. Prefiere `-i` o revisa con `ls` antes.

---

# 3) Ver y editar contenido

```bash
cat archivo.txt            # volcar contenido
nl archivo.txt             # con numeración de líneas
less archivo.txt           # paginado (q para salir, /buscar)
head -n 20 archivo.txt     # primeras 20 líneas
tail -n 50 archivo.txt     # últimas 50 líneas
tail -f logs.txt           # seguir en vivo
tac archivo.txt            # al revés
wc -lwm archivo.txt        # líneas/palabras/bytes
diff a.txt b.txt           # diferencias
```

### Editores

```bash
nano archivo.txt           # simple
vi archivo.txt             # vim (modo comando/insertar)
# En vim:
# i = insertar, ESC = salir a comando, :w = guardar, :q = salir, :wq = guardar y salir
```

---

# 4) Búsqueda y filtrado (power tools)

```bash
grep "patrón" archivo              # buscar texto
grep -R "patrón" carpeta/          # recursivo
grep -Rn "error" .                 # línea + número
cut -d',' -f1,3 archivo.csv        # columnas 1 y 3 (CSV)
sort archivo | uniq -c | sort -nr  # frecuencia de líneas
awk '{print $1}' archivo           # columna 1 (espaciado)
sed -n '1,20p' archivo             # imprime líneas 1-20
sed -E 's/foo/bar/g' archivo       # reemplazo
tr '[:lower:]' '[:upper:]'         # minúsc→MAYÚSC
xargs -0 -I{} cmd {}               # construir comandos
find . -type f -name "*.log"       # buscar por nombre
find . -size +100M                 # mayores a 100 MB
which comando                      # ruta ejecutable
whereis comando                    # binario/man/source
```

> [!tip] Pipe patrón  
> `comando | grep ... | cut ... | sort | uniq -c | sort -nr | head`

---

# 5) Permisos y propietarios

```
r=4, w=2, x=1        # suma por rol
- rwx r-x r--        # dueño / grupo / otros
```

|Octal|Dueño|Grupo|Otros|Uso típico|
|--:|:-:|:-:|:-:|---|
|700|rwx|---|---|solo dueño ejecuta/lee/escribe|
|755|rwx|r-x|r-x|binarios/dirs ejecutables|
|644|rw-|r--|r--|archivos de texto|
|600|rw-|---|---|llaves/credenciales|

```bash
chmod 755 script.sh           # por octal
chmod u+x,g-w archivo         # simbólico
chmod -R 755 carpeta          # recursivo
chown usuario archivo         # cambiar dueño
chown usuario:grupo archivo   # dueño y grupo
chgrp grupo archivo           # cambiar grupo
umask                         # máscara por defecto
```

> [!warning] `sudo` ≠ “forzar”  
> `sudo` ejecuta **como superusuario**. Úsalo sólo cuando entiendas el impacto.

---

# 6) Procesos, trabajos y sistema

```bash
ps aux | grep nombre        # listar procesos
top                         # monitor básico
htop                        # monitor mejor (si instalado)
pgrep -fl nombre            # buscar PIDs por nombre
kill PID                    # terminar proceso
kill -9 PID                 # SIGKILL (último recurso)
pkill nombre                # matar por nombre
jobs                        # trabajos en segundo plano
comando &                   # ejecutar en background
fg %1 / bg %1               # traer/mandar trabajo
time comando                # medir duración
uptime                      # carga del sistema
uname -a                    # kernel/sistema
whoami; id                  # usuario actual / IDs
date                        # fecha/hora
env                         # variables de entorno
export VAR=valor            # setear variable
```

### systemd (Linux)

```bash
sudo systemctl status nginx
sudo systemctl start|stop|restart nginx
journalctl -u nginx --since "10 min ago"
```

---

# 7) Red y descargas

```bash
ip a                # interfaces (Linux)
ifconfig            # (legacy/mac)
ping -c 4 ejemplo.com
traceroute ejemplo.com
nslookup ejemplo.com
dig ejemplo.com +short
ss -tulpen          # sockets/puertos (Linux)
curl -I https://sitio.com        # solo cabeceras
curl -O https://url/archivo.zip  # descarga
wget https://url/archivo.zip
```

---

# 8) Compresión y empaquetado

```bash
# TAR Gzip
tar -czf backup.tar.gz carpeta/     # comprimir
tar -xzf backup.tar.gz              # descomprimir

# ZIP
zip -r archivos.zip carpeta/        # comprimir
unzip archivos.zip                  # extraer
```

---

# 9) Disco y archivos grandes

```bash
df -h                 # espacio en discos
du -sh *              # tamaño de cada ítem en cwd
du -sh carpeta        # tamaño total de carpeta
lsblk                 # bloques/dispositivos (Linux)
```

---

# 10) Transferencias y remoto

```bash
ssh usuario@host                      # conectar
ssh -i ~/.ssh/llave usuario@host      # con llave

scp archivo usuario@host:/ruta/       # copiar (simple)
scp -r carpeta usuario@host:/ruta/

# rsync (rápido e incremental)
rsync -avh --progress carpeta/ usuario@host:/ruta/
```

---

# 11) Paquetes (según sistema)

```bash
# Debian/Ubuntu
sudo apt update && sudo apt install paquete

# Fedora/CentOS/RHEL
sudo dnf install paquete

# Arch
sudo pacman -S paquete

# macOS (Homebrew)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
brew install paquete
```

---

# 12) Historial, alias y productividad

```bash
history                 # ver historial
!!                      # repetir último comando
sudo !!                 # repetir último con sudo
!git                    # último que empezó con "git"
Ctrl+r                  # búsqueda inversa en historial

alias ll='ls -la'       # alias útil (ponlo en ~/.bashrc o ~/.zshrc)
alias gs='git status'
source ~/.bashrc        # recargar configuración
```

---

# 13) Ejemplos rápidos (plantillas)

```bash
# Buscar en logs y ver últimos ocurrencias
grep -Rni "ERROR" /var/log | tail

# Encontrar archivos grandes (>200MB) ordenados
find . -type f -size +200M -print0 | xargs -0 du -h | sort -h

# Extraer columna 2 de CSV, contar valores únicos
cut -d',' -f2 data.csv | sort | uniq -c | sort -nr | head

# Reemplazo in-place (hacer copia antes)
sed -i.bak -E 's/http:/https:/g' rutas.txt
```

---

## ⚠️ Peligros comunes

- `sudo rm -rf` sin verificar ruta.
    
- Sobrescribir con `>` en lugar de `>>`.
    
- `chmod 777` indiscriminado (abre todo). Prefiere `755/644` o simbólicos.
    
- Claves privadas con permisos incorrectos (usa `chmod 600`).
    

---

## ↩️ Volver al HOME

- [[HOME|🏠 HOME del Vault]]