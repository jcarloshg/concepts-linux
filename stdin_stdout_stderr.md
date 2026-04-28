# stdin, stdout & stderr

> Todo lo que necesitas saber sobre los flujos de entrada y salida estándar en Linux/Unix.

---

## 📍 Concepto Básico

Todo proceso en Unix/Linux tiene **3 flujos estándar**:

| Stream | File Descriptor | Propósito | Dirección |
|---|---|---|---|
| **stdin** | 0 | Entrada de datos (teclado, pipe, archivo) | ← entrada |
| **stdout** | 1 | Salida normal (resultados, datos) | → salida |
| **stderr** | 2 | Mensajes de error, advertencias | → salida |

```
        ┌──────────────┐
 stdin  │              │
   (0) ─┼──►  PROGRAMA ├───► stdout (1)
        │              │
        │              ├───► stderr (2)
        └──────────────┘
```

**stdin** es el único flujo de entrada. Por defecto viene del teclado (tty), pero puede ser redirigido desde un archivo o desde otro comando via pipe.

---

## ⌨️ stdin — La Entrada Padrón

### ¿Qué es stdin?

Es el canal por donde los programas **reciben datos**. Sin stdin, un programa no sabe de dónde leer.

### ¿De dónde viene stdin?

| Fuente | Ejemplo |
|---|---|
| **Teclado** (tty interactivo) | `cat` sin argumentos espera input del teclado |
| **Pipe** (de otro comando) | `echo "hola" | cat` |
| **Archivo redireccionado** | `cat < archivo.txt` |
| **Here-string** | `cat <<< "texto directo"` |
| **Here-doc** | `cat << EOF ... EOF` |

### Ejemplo básico: leer de stdin
```bash
# Sin redirección — espera del teclado
cat
# Escribe algo y presiona Ctrl+D para terminar

# Con pipe — el output de echo se convierte en stdin de cat
echo "hola mundo" | cat

# Con redirección — archivo se convierte en stdin
cat < archivo.txt

# Here-string — texto directo como stdin
cat <<< "este texto viene de stdin"
```

---

## 📖 stdin en Programas

### ¿Cómo usan stdin los programas?

Los programas en C/C++/Python/etc. usan el file descriptor 0 para leer:

```c
// C: read from stdin
scanf("%s", buffer);
// o
read(0, buffer, sizeof(buffer));

// Python: read from stdin
nombre = input("Dame tu nombre: ")
// o
import sys
sys.stdin.read()
```

### stdin en scripts
```bash
#!/bin/bash
# Lee línea por línea de stdin
while read line; do
  echo "Leído: $line"
done

# Enviar texto a un script via pipe
echo -e "línea1\nlínea2\nlínea3" | ./mi_script.sh

# Aquí-document (here-doc)
./mi_script.sh << EOF
primera línea
segunda línea
EOF
```

### stdin con here-doc avanzado
```bash
# here-doc con delimitador personalizado
cat << 'FIN'
Esto NO se expande $HOME ni variables
FIN

# here-doc SIN comillas — se expanden variables
cat << FIN
Mi home es: $HOME
FIN

# here-doc con stdin en scripts interactivos
ftp -n << EOF
open servidor.com
user usuario password
get archivo.txt
bye
EOF
```

---

## 🔧 Redirección de stdin

### `<` — Redirigir stdin desde archivo
```bash
# Equivalentes:
cat < archivo.txt
cat archivo.txt

# ¿Cuándo importa? En programas que no aceptan filenames
read -p "Nombre: " nombre < /dev/tty

# Leer múltiples archivos
programa < entrada.txt
```

### `<<` — Here-document
```bash
# Enviar varias líneas como stdin
mysql -u root -p << EOF
CREATE DATABASE app;
USE app;
SELECT * FROM users;
EOF
```

### `<<<` — Here-string
```bash
# Una línea directa como stdin
wc -c <<< "hola mundo"   # 11 (bytes)
tr 'a-z' 'A-Z' <<< "hola"  # HOLA

# Útil para comandos que esperan stdin pero no argumentos
read -r linea <<< "texto inicial"
```

---

## 🎯 Casos de Uso de stdin

### 1. Enviar output de un comando como stdin de otro
```bash
# grep espera stdin, el pipe le envía las líneas
ls -la | grep "\.md"

# sort espera stdin
echo -e "zebra\napple\nbanana" | sort

# head/tail
cat archivo.log | tail -20
```

### 2. Leer archivos grandes en partes
```bash
# Procesar línea por línea (eficiente en memoria)
while IFS= read -r line; do
  echo "$line"
done < archivo_grande.txt

# Con pipe se carga todo a memoria primero
cat archivo_grande.txt | while read line; do
  echo "$line"
done
```

### 3. Scripts interactivos que preguntan
```bash
# Sin redirección, read funciona con el keyboard
read -p "Tu nombre: " nom

# Con redirección de archivo, no pregunta (ya hay datos)
echo "Carlos" | ./script.sh  # read recibe "Carlos"

# En producción: mejor usar argumentos, no stdin para datos
./script.sh "Carlos"  # más claro
```

### 4. stdin en programas compilados
```bash
./programa < input.txt > output.txt 2>&1
# input.txt → stdin de programa
# output.txt ← stdout del programa
# errores.txt ← stderr
```

---

## 🏆 Best Practices & Tricks con stdin

### 1. No uses stdin para datos en scripts de producción
```bash
# ❌ Mal — stdin se mezcla con logs
./proceso.sh < datos.txt

# ✅ Mejor — argumentos explícitos
./proceso.sh datos.txt
```

### 2. Saber si stdin viene de terminal o pipe
```bash
#!/bin/bash
if [ -t 0 ]; then
  echo "stdin viene del teclado (interactivo)"
else
  echo "stdin viene de pipe o archivo"
fi

# Uso práctico: auto-detectar modo
if [ -t 0 ]; then
  read -p "Dame el valor: " valor
else
  read valor
fi
```

### 3. Cerrar stdin explícitamente (para demonios)
```bash
# En scripts que se vuelven daemon
programa < /dev/null
# Garantiza que stdin no bloquee el proceso
```

### 4. Desacoplar stdin para background jobs
```bash
# stdin's redirected incorrectly can hang background processes
nohup ./script.sh < /dev/null > output.log 2>&1 &
```

### 5. stdin en pipelines con read
```bash
# El classic problema: pipe crea subshell
# Esto NO funciona como esperado:
echo -e "a\nb\nc" | while read line; do
  echo $line
done

# ✅ Alternativa: usar here-doc o redirección
while read line; do
  echo "$line"
done <<< $'a\nb\nc'

# o desde archivo
printf 'a\nb\nc\n' > /tmp/temp.txt
while read line; do
  echo "$line"
done < /tmp/temp.txt
```

### 6. stdin con xargs
```bash
# stdin se convierte en argumentos
echo -e "file1.txt\nfile2.txt" | xargs rm

# stdin vacío vs argumentos
xargs -I {} echo "procesando {}" < input.txt
```

### 7. Probar stdin sin matar el terminal
```bash
# Cierra stdin y verifica que programa no cuelgue
programa < /dev/null

# Test con timeout
timeout 5 ./programa < input.txt
```

### 8. Leer stdin sin consumirlo (tee trick)
```bash
# Quiero ver stdin Y procesarlo
echo "datos" | tee /dev/tty | programa

# tee muestra en terminal y pasa los datos al programa
```

### 9. stdin en comandos que esperan un filename
```bash
# diff puede leer de stdin
diff <(ls dir1) <(ls dir2)
# Process substitution: ls output → diff

# patch espera stdin
patch -p1 < changes.patch
```

### 10. stdin buffer en programas interactivos
```bash
# Esperar entrada sin bloquear con timeout
timeout 2 read -t 2 linea < /dev/tty
echo "input: $linea"
```

---

## 🔄 stdin, stdout & stderr Juntos

### Ejemplo completo
```bash
# Un comando: stdin del teclado, stdout/stderr a pantalla
cat

# Redirigir stdin desde archivo
cat < datos.txt

# Solo stdout a archivo
cat archivo.txt > output.txt

# Solo stderr a archivo
cat noexiste.txt 2> errores.txt

# Ambos a archivos diferentes
cat archivo.txt > output.txt 2> errores.txt

# Ambos al mismo archivo
cat archivo.txt > todo.txt 2>&1

# stdin de pipe, stdout a archivo, stderr a archivo
cat archivo.txt | grep "patron" > resultados.txt 2> errores.txt
```

### El clásico diagrama de redirección
```
                ┌── stdin (0) ────────────┐
                │  keyboard / pipe / file  │
                └───┬─────────────────────┘
                    │
               PROGRAMA
                    │
         ┌──────────┴──────────┐
         │                     │
    stdout (1)           stderr (2)
         │                     │
         ▼                     ▼
    > archivo             2> archivo
    | otro_cmd            2>&1 (a stdout)
```

---

## 🧠 Resumen Rápido

| Situación | Comando |
|---|---|
| Leer stdin de teclado | `cat`, `read` |
| stdin desde archivo | `cmd < file.txt` |
| stdin desde pipe | `echo "x" | cmd` |
| stdin desde texto directo | `cmd <<< "texto"` |
| stdin multi-línea | `cmd << EOF ... EOF` |
| Here-doc sin expansión | `cmd << 'EOF' ... EOF` |
| Ver stdin en terminal | `tee /dev/tty` |
| Cerrar stdin | `cmd < /dev/null` |
| Detectar si es terminal | `[ -t 0 ]` |
| stdin + stdout + stderr juntos | `cmd < in.txt > out.txt 2> err.txt` |

---

## 📚 Recursos

- 🔗 https://www.gnu.org/software/bash/manual/html_node/Redirections.html
- 🔗 https://www.gnu.org/software/bash/manual/html_node/Here-Documents.html
- 🔗 https://linux.die.net/man/1/bash
- 🔗 https://www.geeksforgeeks.org/input-and-output-in-c/

---

> **Tip final:** Si un comando espera input y se queda "colgado", probablemente necesita stdin desde un archivo o pipe. Presiona Ctrl+D para enviar EOF o Ctrl+C para cancelar.

---

# El Character `&` en Shell

> Guía completa sobre los múltiples usos del ampersand en bash/shell.

---

## 📍 Concepto Básico

El character `&` tiene **6 usos principales** en shell:

| Uso | Sintaxis | Propósito |
|---|---|---|
| **Background job** | `cmd &` | Ejecutar en background |
| **AND lógico** | `cmd1 && cmd2` | Ejecutar cmd2 si cmd1 succeed |
| **Scoped variable** | `${var}` | Expansión de variables |
| **Process substitution** | `<(cmd)` / `>(cmd)` | Crear FIFOs temporales |
| **Job ID en background** | `%1`, `%2` | Referencia a jobs |
| **Scoped substitution** | `${var/proc}` | Pattern replacement |

---

## 1. ▶️ Background Jobs — `command &`

### Uso básico
```bash
# Ejecutar en background (libera el terminal)
./script_largo.sh &

# Ver output después
jobs
fg  # traer al foreground
```

### ¿Qué pasa cuando usas `&`?

1. El proceso **se desacopla** del terminal
2. El shell **no espera** a que termine
3. Puedes seguir usando la terminal
4. El proceso sigue corriendo aunque cierres el terminal (si no usas `nohup`)

```bash
# Ejemplo real
sleep 30 &
# Immediately returns to prompt
# El sleep corre en background
```

### ¿Cuándo usar background jobs?

| Situación | Ejemplo |
|---|---|
| Script que tarda mucho | `./backup.sh &` |
| Servidor que espera conexiones | `./server.sh &` |
| Tarea que no necesita interacción | `find / -name "*.log" &` |
| Mientras esperas, haces otra cosa | `sleep 5 && echo "listo"` |

---

## 2. 🔗 AND Lógico — `command1 && command2`

### Funcionamiento

Ejecuta **command2 solo si command1** termina con exit status 0 (éxito).

```bash
# Ejemplo clásico: compilar y si funciona, ejecutar
gcc programa.c -o programa && ./programa

# Cadena de comandos: uno tras otro, se detiene si uno falla
cd /tmp && cp archivo.txt backup/ && rm archivo.txt
```

### Tabla de verdad

| command1 exit | command2 ejecutado? |
|---|---|
| 0 (éxito) | ✅ Sí |
| ≠0 (fallo) | ❌ No |

```bash
# Demostración
true && echo "primero succeed"   # imprime
false && echo "nunca"            # no imprime (false = exit 1)
```

### `&&` vs `;` (punto y coma)

```bash
# ; Ejecuta sin importar el resultado
cd /tmp ; echo "siempre corre"

# && Solo ejecuta si el anterior succeedió
cd /noexiste && echo "esto no corre"
```

---

## 3. 🔄 OR Lógico — `command1 || command2`

No es `&` pero está relacionado — funciona al revés:

Ejecuta **command2 solo si command1 falla** (exit ≠ 0).

```bash
# Si falla gcc, usar cc como fallback
gcc archivo.c -o prog || cc archivo.c -o prog

# Si no existe archivo, crear uno vacío
[ -f config.txt ] || touch config.txt
```

### Combinar `&&` y `||`

```bash
# If-then-else en una línea
[ -f archivo.txt ] && echo "existe" || echo "no existe"

# ⚠️ Cuidado: esto puede fallar si echo succeed
# Mejor usar:
if [ -f archivo.txt ]; then echo "existe"; else echo "no existe"; fi
```

---

## 4. 📦 Process Substitution — `<(cmd)` y `>(cmd)`

### ¿Qué es?

Crea un **FIFO temporal** (named pipe) que conecta la salida de un comando como si fuera un archivo.

### `<(command)` — Leer como archivo

```bash
# diff puede comparar outputs sin archivo temporal
diff <(ls dir1) <(ls dir2)

# wc puede contar sin archivo
wc -l <(grep "ERROR" logfile.txt)

# cat lee el output de un subshell
cat <(echo "hola desde subprocess")
```

### `>(command)` — Escribir como archivo

```bash
# Capturar output que iría a un archivo
tee >(wc -l > lineas.txt) < archivo.txt

# Verificar output sin alterar
./programa | tee >(grep "ERROR" > errores.log)
```

### ¿Cuándo usar process substitution?

| Uso | Ejemplo |
|---|---|
| Comandos que esperan **filename** | `diff <(cmd1) <(cmd2)` |
| Evitar archivos temporales | `grep patron <(curl url)` |
| Comparar outputs | `comm <(sort a) <(sort b)` |

---

## 5. 🏗️ Scoped Variable Expansion — `${variable}`

### Uso básico con `&`

El `&` en `${var}` sirve para:

```bash
nombre="archivo"
echo "${nombre}_backup"    # archivo_backup

ruta="/home/user"
echo "${ruta}/documents"   # /home/user/documents

# Parameters avanzados
texto="hello"
echo "${texto^}"          # Hello (capitalize)
echo "${texto,,}"         # hello (lowercase)

unset_var=""
echo "${unset_var:-default}"  # default (porque está vacía)
echo "${unset_var:=default}"  # default (Y asigna a la variable)
```

### Pattern substitution con `&`

```bash
# & en expansión reemplaza el match
texto="123abc456"
echo "${texto//[0-9]/X}"   # XXXabcXXX

# En sed: & significa toda la línea matcheada
echo "hola" | sed 's/hola/& mundo/'
# hola mundo
```

---

## 6. 🎯 Job Control — `%n` y friends

### Jobs del shell actual

```bash
# Background jobs activos
jobs
# [1]+  Running    sleep 30 &
# [2]-  Stopped    vim archivo.txt

# Traer al foreground
fg %1

# Mandar a background (si está stopped)
bg %2

# Matar un job
kill %1
```

### ¿Cuándo usar job control?

| Situación | Comando |
|---|---|
| Hiciste Ctrl+Z (pausaste) | `fg` para recuperar |
| quieres seguir usando terminal | `bg` para que corra en bg |
| tienes múltiples tareas | `jobs` para ver todas |

---

## 🏆 Best Practices & Tricks con `&`

### 1. Background con nohup (para que sobreviva al terminal)
```bash
# ❌ Terminal cerrado = proceso muerto
./servidor &

# ✅ Proceso sobrevive al terminal
nohup ./servidor > server.log 2>&1 &
```

### 2. El clásico OR-AND combo
```bash
# Compilar: si gcc falla, si cc falla, fallar
(gcc archivo.c -o prog && ./prog) || (cc archivo.c -o prog && ./prog) || echo "Compilación falló"
```

### 3. Multiple background jobs en paralelo
```bash
# Lanzar 3 procesos a la vez
./script1.sh &
./script2.sh &
./script3.sh &

# Esperar a que todos terminen
wait

echo "Todos los procesos terminados"
```

### 4. Background job con timeout
```bash
# Matar si tarda más de 60 segundos
timeout 60 ./script.sh &
```

### 5. La diferencia crítica entre `&>` y `>&`
```bash
# ✅ Bash moderno: ambos streams a archivo
cmd &> archivo.txt

# ✅ Equivalente clásico
cmd > archivo.txt 2>&1

# ❌ INCORRECTO (orden importa)
cmd 2>&1 > archivo.txt  # stderr va a stdout ORIGINAL (pantalla), no al archivo
```

### 6. Process substitution para comparar sin archivos
```bash
# Diff de dos outputs
diff <(ls /bin) <(ls /usr/bin) | head -20

# Ordenar y quitar duplicados sin archivo
comm <(ls dir1 | sort) <(ls dir2 | sort)
```

### 7. Background jobs en scripts (wait + error check)
```bash
#!/bin/bash
./task1.sh &
pid1=$!
./task2.sh &
pid2=$!

# Esperar ambos
wait $pid1 $pid2

# Verificar si alguno falló
if [ $? -ne 0 ]; then
  echo "Algo falló"
fi
```

### 8. Usar `&&` para validación previa
```bash
# Solo correr si hay permisos
[ -w archivo.txt ] && echo "puedo escribir" || echo "sin permisos"

# Solo correr si existe
[ -f config.sh ] && source config.sh || echo "config no existe"
```

### 9. Evitar subshell pitfalls con background
```bash
# ❌ El pipe crea subshell — variable no persiste
cat archivo | while read line; do
  queue="$line"
done
echo "$queue"  # vacío!

# ✅ Usar while con redirección (sin pipe)
while read line; do
  queue="$line"
done < archivo.txt
echo "$queue"  # funciona
```

### 10. Disown — liberar job del shell
```bash
# Si no quieres que sea controlado por el shell
./script_largo.sh &
disown %1
# Ahora el shell puede cerrarse sin matar el proceso
```

---

## 🧠 Resumen Rápido

| Sintaxis | Significado | Ejemplo |
|---|---|---|
| `cmd &` | Background | `./script.sh &` |
| `cmd1 && cmd2` | AND lógico | `gcc && ./prog` |
| `cmd1 \|\| cmd2` | OR lógico | `gcc \|\| cc` |
| `${var}` | Expansión | `${HOME}/docs` |
| `${var/proc}` | Substitution | `${str//a/b}` |
| `<(cmd)` | Read como archivo | `diff <(ls a) <(ls b)` |
| `>(cmd)` | Write como archivo | `tee >(wc -l > f)` |
| `%1`, `%2` | Job ID | `kill %1` |
| `jobs` | Ver bg jobs | `jobs` |
| `fg` | Traer foreground | `fg` |
| `bg` | Mandar background | `bg` |
| `nohup cmd &` | Background immortal | `nohup ./srv &` |
| `disown %n` | Liberar del shell | `disown %1` |
| `wait` | Esperar bg jobs | `wait` |

---

## ⚠️ Errores Comunes

```bash
# ❌ Error: background con stdin interactivo cuelga
./programa_interactivo &
# Se queda esperando stdin

# ✅ Solución: cerrar stdin
./programa_interactivo < /dev/null &

# ❌ Error: stderr no redirigido en background
./script.sh &  # stderr sigue a terminal

# ✅ Solución: redirigir todo
./script.sh > output.log 2>&1 &

# ❌ Error: asumir orden de && vs ;
cd /noexiste ; echo "siempre corro"
# Primero falla, pero echo igual corre

# ✅ Si quieres parar en error:
cd /noexiste && echo "solo corro si worked"
```

---

## 📚 Recursos

- 🔗 https://www.gnu.org/software/bash/manual/html_node/Job-Control.html
- 🔗 https://www.gnu.org/software/bash/manual/html_node/Process-Substitution.html
- 🔗 https://www.gnu.org/software/bash/manual/html_node/Job-Control-BUILTINS.html
- 🔗 https://linux.die.net/man/1/bash

---

> **Tip final:** Usa `&` para background, `&&` para encadenar comandos seguros, y `${var}` para expansiones. Si algo falla silenciosamente en background, revisa que stdin no esté esperando — usa `< /dev/null`.