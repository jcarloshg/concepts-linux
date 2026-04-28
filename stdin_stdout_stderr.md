# stderr & stdout

> Todo lo que necesitas saber sobre los flujos de salida estándar en Linux/Unix.

---

## 📍 Concepto Básico

| Stream | File Descriptor | Propósito | pipe |
|---|---|---|---|
| **stdout** | 1 | Salida normal (resultados, datos) | ✅ Se puede pipe |
| **stderr** | 2 | Mensajes de error, advertencias | ❌ No se pipe por default |

```
stdin (0) ← entrada
stdout (1) → salida normal
stderr (2) → errores
```

---

## 🔧 Redirección Básica

### stdout a archivo
```bash
command > output.txt
# equivale a:
command 1> output.txt
```

### stderr a archivo
```bash
command 2> errors.txt
```

### Ambos al mismo archivo
```bash
command > all.txt 2>&1
# o en bash moderno:
command &> all.txt
```

### stderr a stdout ( juntos)
```bash
command &>> all.txt   # append
command > all.txt 2>&1 # overwrite
```

### Solo stdout a archivo, stderr a pantalla
```bash
command > output.txt
```

---

## 🎯 Casos de Uso

### 1. Silenciar errores (dev/null)
```bash
command 2>/dev/null   # stderr a la nada
command >/dev/null 2>&1  # todo silenciado
```

### 2. Separar errores de resultados
```bash
# Guardar output válido, ver errores en pantalla
make > build.log 2>&1

# Solo errores limpios
make 2> errors.log
```

### 3. Pipe stdout pero mantener stderr
```bash
command 2>&1 | grep "error"
# Ambos van al pipe

command | grep "error"
# Solo stdout va al pipe, stderr sigue a pantalla
```

### 4. Logging con timestamps
```bash
command 2>&1 | while read line; do
  echo "[$(date '+%H:%M:%S')] $line"
done > log.txt
```

---

## 🏆 Best Practices & Tricks

### 1. Silenciar output innecesario
```bash
# Silenciar stdout (útil en scripts)
command > /dev/null

# Silenciar ambos
command &> /dev/null

# Silenciar solo stderr
command 2>/dev/null
```

### 2. El经典的 2>&1 orden importa
```bash
# ✅ CORRECTO — stderr va al stdout actual
command > file.txt 2>&1

# ❌ INCORRECTO — stderr va al stdout original (pantalla)
command 2>&1 > file.txt
```

### 3. Agregar a archivo existente
```bash
command >> output.log 2>&1
```

### 4. Separar errores para debugging
```bash
# En scripts: separar para encontrar problemas rápido
./script.sh 2> errors.log
# Si script falla, revisar errors.log
```

### 5. Redirect a múltiples destinos con tee
```bash
# stdout a pantalla + archivo
command | tee output.txt

# ambos streams a pantalla + archivo
command 2>&1 | tee output.txt

# append mode con tee
command 2>&1 | tee -a output.txt
```

### 6. stderr a stdout con | (limpiar errores)
```bash
# stderr va a stdout, luego se filtra
myprogram 2>&1 | grep -i "failed"
```

### 7. Usar stderr para logs de scripts
```bash
# En scripts, usar stderr para errores intencionales
echo "Error: algo falló" >&2
# así no se mezcla con output normal en logs
```

### 8. Intercambiar stdout y stderr
```bash
# Para pruebas: swap streams
command 3>&1 1>&2 2>&3
```

### 9. Capturar solo errores (útil para CI/CD)
```bash
# Guardar errores, output normal a pantalla
command 2> errors.txt
# errors.txt tendrá solo stderr
```

### 10. Describir la redirección en un script
```bash
# Función helper para readability
log_output() {
  "$@" > >(tee -a stdout.log) 2> >(tee -a stderr.log >&2)
}

log_output make build
```

---

## 🧠 Resumen Rápido

| Situación | Comando |
|---|---|
| Guardar stdout | `cmd > file` |
| Guardar stderr | `cmd 2> file` |
| Guardar ambos | `cmd > file 2>&1` |
| Silenciar stdout | `cmd > /dev/null` |
| Silenciar stderr | `cmd 2> /dev/null` |
| Silenciar todo | `cmd &> /dev/null` |
| Pipe stdout | `cmd \| other` |
| Pipe ambos | `cmd 2>&1 \| other` |
| Append | `cmd >> file 2>&1` |

---

## 📚 Recursos

- 🔗 https://www.gnu.org/software/bash/manual/html_node/Redirections.html
- 🔗 https://stackoverflow.com/questions/4614492/bash-redirect-stderr-and-stdout
- 🔗 https://linux.die.net/man/1/bash

---

> **Tip final:** Si algo falla silenciosamente — revisa `2>` stderr. Si algo se muestra pero no，你应该 captured — stdout.