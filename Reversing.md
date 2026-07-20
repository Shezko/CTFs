# Reversing

Piensa en capas: bytes -> ensamblador -> C/C++ decompilado -> lógica en Python/JavaScript. La meta suele ser entender la función `check`, recuperar el input correcto o parchear un salto.

## Primeros 60 segundos

```bash
file binario
checksec --file=binario
strings -a binario | grep -Ei "flag|ctf|password|correct|wrong|usage|key"
./binario
```

Preguntas:

1. Es ELF, PE, Mach-O, Python, Java, .NET?
2. Es 32 o 64 bits?
3. Esta stripped?
4. Pide input?
5. Aparece la flag en strings?
6. Hay función `main`, `check`, `verify`, `validate`?

## Las 4 capas

| Capa | Qué buscas | Tools |
|---|---|---|
| Python / JavaScript | Código fuente, bytecode, ofuscación simple | `python`, `uncompyle6`, `pycdc`, `node`, beautifiers |
| C / C++ fuente/decompilado | Lógica de validación | Ghidra, Binary Ninja, IDA Free |
| Ensamblador | Saltos, comparaciones, registros | GDB + GEF, objdump, radare2 |
| Bytes | Parches, strings, magic, XOR | `xxd`, `hexedit`, `rizin`, `objcopy` |

## Strings + file

**Contexto:** primer paso para saber formato, arquitectura y pistas obvias.

**Tools:** `file`, `strings`, `checksec`, `ltrace`, `strace`.

**Ejemplos:**

```bash
file binario
strings -a binario | head
strings -a binario | grep -Ei "flag|ctf|pass|correct|wrong|try|key"
checksec --file=binario
```

Librerias y llamadas:

```bash
ltrace ./binario
strace -f ./binario
```

Buscar comparaciones de strings:

```bash
ltrace ./binario 2>&1 | grep -Ei "strcmp|strncmp|memcmp|flag|pass"
```

## Ghidra

**Contexto:** decompilador gratis, muy usado para ELF Linux, PE Windows y binarios C/C++.

**Use para:** entender `main`, renombrar variables, localizar `check`, ver pseudocodigo, seguir xrefs.

**Workflow GUI:**

1. Crear proyecto.
2. Importar binario.
3. Ejecutar Auto Analyze.
4. Ir a `Symbol Tree -> Functions -> main`.
5. Ignorar `_libc_start_main`, `_start`, `frame_dummy`, `__do_global_dtors_aux`.
6. Buscar strings interesantes en `Search -> For Strings`.
7. Doble click en string y revisar Xrefs.
8. Renombrar funciones/variables mientras entiendes la lógica.

**Comandos externos para apoyar Ghidra:**

```bash
objdump -d binario | less
objdump -M intel -d binario | less
readelf -h binario
readelf -s binario | grep -Ei "main|check|flag|verify"
```

**Qué mirar en el decompilado:**

- `strcmp(input, "valor") == 0`
- `memcmp(input, key, len)`
- Bucles que transforman input con XOR/suma/rotación.
- Arrays de bytes usados para comparar.
- Condiciones que llevan a `puts("Correct")`.

## Binary Ninja

**Contexto:** decompilador comercial con versión free disponible; buena alternativa a Ghidra cuando quieres una UI rápida y análisis interactivo.

**Use para:** alternativa a Ghidra, HLIL, graph view, renombrar funciones, navegar xrefs.

**Workflow GUI:**

1. Abrir binario.
2. Esperar auto-analysis.
3. Ir a `main` o buscar strings.
4. Usar vista HLIL para pseudocodigo.
5. Seguir llamadas hacia funciones tipo `check`, `validate`, `decrypt`.

**Apoyo desde terminal:**

```bash
strings -a binario | grep -Ei "correct|wrong|flag|password"
nm -an binario | grep -Ei "main|check|validate|verify"
```

## GDB + GEF

**Contexto:** análisis dinámico: correr el programa, parar en funciones, ver registros/memoria y cambiar saltos o valores en caliente.

**Tools:** `gdb`, GEF, `pwndbg` como alternativa.

**Empezar:**

```bash
gdb ./binario
```

Dentro de GDB:

```gdb
run
break main
run
disassemble main
info functions
info registers
x/s $rdi
x/20gx $rsp
continue
```

Breakpoints útiles:

```gdb
break *main
break strcmp
break strncmp
break memcmp
break puts
```

Ver argumentos en x86_64 Linux:

| Argumento | Registro |
|---|---|
| 1 | `rdi` |
| 2 | `rsi` |
| 3 | `rdx` |
| 4 | `rcx` |
| 5 | `r8` |
| 6 | `r9` |

Ejemplo para pillar password comparada:

```gdb
break strcmp
run
x/s $rdi
x/s $rsi
continue
```

GEF comandos útiles:

```gdb
checksec
context
pattern create 100
pattern search
got
plt
vmmap
```

Cambiar flujo en caliente:

```gdb
set $eax=0
set $rip=0x401234
jump *0x401234
```

## Framework de ataque: primeros 60 segundos

**Paso 1: strings**

```bash
strings -a binario | grep -i flag
strings -a binario | grep -Ei "correct|wrong|password|key"
```

Si aparece la flag o password, listo. Si aparece texto de éxito/error, usarlo como ancla en Ghidra.

**Paso 2: file**

```bash
file binario
```

Responde:

- Es ELF?
- 32/64-bit?
- Stripped?
- Dynamically linked?

**Paso 3: abrir decompilador**

En Ghidra/Binary Ninja:

- Ir a `main`.
- Buscar strings de éxito/error.
- Seguir Xrefs.
- Identificar función `check`.

**Paso 4: entender check**

Patrones comunes:

```c
if (strcmp(input, "secret") == 0) puts("Correct");
```

```c
for (i = 0; i < len; i++) {
    if ((input[i] ^ 0x13) != expected[i]) fail();
}
```

```c
if (strlen(input) != 32) fail();
```

**Paso 5: bypass o resolver**

Opciones:

- Leer el valor esperado.
- Escribir script inverso.
- Parchear salto.
- Cambiar registro en GDB.

## Patching simple

**Contexto:** quieres cambiar un `jne` por `je`, o convertir un salto condicional en NOPs para llegar a la rama correcta.

**Tools:** Ghidra Patch Instruction, Binary Ninja patch, `radare2`, `hexedit`.

**radare2 ejemplo:**

```bash
r2 -w ./binario
```

Dentro:

```text
aaa
s main
pdf
s 0x401234
wa nop
wa jmp 0x401260
wq
```

Con GDB sin parchear archivo:

```gdb
break *0x401234
run
set $eax=0
continue
```

## Resolver transformaciones

**Contexto:** el binario transforma input y compara contra bytes esperados.

**Tools:** Python, Ghidra para copiar arrays, CyberChef si es XOR/base64.

**XOR inverso:**

```python
expected = [0x75, 0x7f, 0x72, 0x74]
key = 0x13
print(bytes([b ^ key for b in expected]))
```

**Suma/resta:**

```python
expected = [110, 119, 106, 112]
print(bytes([(b - 3) & 0xff for b in expected]))
```

**Comparacion por posición:**

```python
expected = [0x66, 0x6c, 0x61, 0x67]
chars = ["?"] * len(expected)
for i, b in enumerate(expected):
    chars[i] = chr(b)
print("".join(chars))
```

## Python / JavaScript reversing

**Contexto:** el reto no es binario nativo sino script, bytecode u ofuscación.

**Tools:** `python`, `node`, beautifiers, `uncompyle6`, `pycdc`, `strings`.

**Python source:**

```bash
python3 script.py
python3 -m dis script.py
```

Bytecode `.pyc`:

```bash
file challenge.pyc
uncompyle6 challenge.pyc > challenge.py
pycdc challenge.pyc > challenge.py
```

JavaScript:

```bash
node challenge.js
js-beautify challenge.js -o challenge.pretty.js
grep -Ei "flag|check|verify|password|correct" challenge.pretty.js
```

## Binarios empacados o raros

**Contexto:** `strings` no muestra casi nada, entropía alta, UPX, secciones raras.

**Tools:** `upx`, `binwalk`, `readelf`, `diec`, Detect It Easy.

**Ejemplos:**

```bash
strings -a binario | grep -i upx
upx -d binario -o unpacked
file unpacked
```

Secciones:

```bash
readelf -S binario
objdump -h binario
```

## Resumen GDB corto

```gdb
break main
run
ni
si
finish
continue
disassemble
x/10i $pc
x/s $rdi
x/s $rsi
x/32bx $rsp
info registers
set $eax=0
quit
```

## Orden recomendado

1. `strings binario | grep -i flag`.
2. `file binario`.
3. Abrir Ghidra o Binary Ninja.
4. Ir a `main`, no perder tiempo en `_start`.
5. Buscar función `check`.
6. Usar GDB si necesitas ver valores reales.
7. Resolver con script o parchear salto.
