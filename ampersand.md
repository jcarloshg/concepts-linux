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