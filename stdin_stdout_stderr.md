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

>