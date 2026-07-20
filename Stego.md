# Stego

Primero identifica el tipo real del archivo, luego aplica herramientas según formato. En CTF, la flag puede estar en metadatos, bits LSB, chunks, comentarios, archivos embebidos, password de `steghide`, espectrogramas o texto invisible.

## Checklist inicial

```bash
file archivo
exiftool archivo
strings -a archivo | grep -Ei "flag|ctf|password|secret|key"
binwalk archivo
```

Si `binwalk` detecta contenido:

```bash
binwalk -e archivo
find _archivo.extracted -type f -exec file {} \;
```

## PNG

**Contexto:** PNG con LSB, chunks raros, texto embebido, errores CRC o datos concatenados.

**Tools:** `zsteg`, `stegsolve`, `pngcheck`, `exiftool`, `binwalk`, `strings`.

**Ejemplos:**

```bash
file imagen.png
pngcheck imagen.png
pngcheck -v imagen.png
pngcheck -t imagen.png
zsteg imagen.png
zsteg -a imagen.png
```

Extraer payload concreto de `zsteg`:

```bash
zsteg -E "b1,rgb,lsb,xy" imagen.png > payload.bin
file payload.bin
strings -a payload.bin
```

Buscar archivos embebidos:

```bash
binwalk imagen.png
binwalk -e imagen.png
strings -a imagen.png | grep -Ei "flag|ctf|password|secret"
```

**Stegsolve GUI:** usar para revisar planos de color, invertir colores, alpha, XOR visual y diferencias. No hay sintaxis relevante: abre la imagen y navega por `Analyse` y los bit planes.

## JPG / JPEG

**Contexto:** JPEG con `steghide`, comentarios EXIF, thumbnails, datos al final o password crackeable.

**Tools:** `steghide`, `stegseek`, `exiftool`, `strings`, `binwalk`.

**Ejemplos:**

```bash
exiftool foto.jpg
strings -a foto.jpg | grep -Ei "flag|ctf|password|secret"
binwalk foto.jpg
```

Revisar si `steghide` detecta datos:

```bash
steghide info foto.jpg
```

Extraer si tienes passphrase:

```bash
steghide extract -sf foto.jpg -p "password"
```

Crackear passphrase con `stegseek`:

```bash
stegseek foto.jpg /usr/share/wordlists/rockyou.txt
stegseek --crack foto.jpg /usr/share/wordlists/rockyou.txt
```

Si encuentra password, normalmente extrae el archivo automáticamente o deja un output con el contenido recuperado.

## BMP

**Contexto:** BMP es común para LSB porque no comprime como JPEG.

**Tools:** `zsteg`, `stegsolve`, `binwalk`, `strings`.

**Ejemplos:**

```bash
zsteg imagen.bmp
zsteg -a imagen.bmp
zsteg -E "b1,b,lsb,xy" imagen.bmp > payload.bin
file payload.bin
strings -a payload.bin
```

## WAV / MP3 / audio

**Contexto:** flags escondidas en espectrograma, canal izquierdo/derecho, audio invertido, tonos DTMF, morse o datos LSB.

**Tools:** Audacity, Sonic Visualiser, `sox`, `ffmpeg`, `strings`, `binwalk`, `exiftool`.

**Audacity GUI:** abrir audio, cambiar vista a espectrograma, revisar canales, invertir/revertir audio, amplificar, cambiar velocidad. Si ves texto en espectrograma, esa suele ser la flag o pista.

**Ejemplos terminal:**

```bash
file audio.wav
exiftool audio.wav
strings -a audio.wav | grep -Ei "flag|ctf|password|secret"
binwalk audio.wav
```

Convertir o separar canales:

```bash
ffmpeg -i audio.mp3 audio.wav
sox audio.wav left.wav remix 1
sox audio.wav right.wav remix 2
sox audio.wav reversed.wav reverse
```

Morse aproximado:

```bash
sox audio.wav -n stat
```

Para DTMF, usa herramientas como `multimon-ng`:

```bash
multimon-ng -t wav -a DTMF audio.wav
```

## Texto

**Contexto:** texto con caracteres invisibles, zero-width, espacios/tabs, base64, ROT, Unicode raro o errores ortográficos intencionales.

**Tools:** CyberChef, `xxd`, `base64`, `tr`, `sed`, detectores zero-width, `grep`.

**Ejemplos:**

```bash
xxd texto.txt | head
cat -A texto.txt
strings -a texto.txt
```

Base64:

```bash
base64 -d texto.b64
echo "ZmxhZ3t0ZXN0fQ==" | base64 -d
```

Hex:

```bash
echo "666c61677b746573747d" | xxd -r -p
```

ROT13:

```bash
echo "synt{grfg}" | tr 'A-Za-z' 'N-ZA-Mn-za-m'
```

Zero-width:

```bash
xxd texto.txt | grep -Ei "e2808b|e2808c|e2808d|efbbbf"
```

CyberChef GUI: usar `Magic`, `From Base64`, `From Hex`, `ROT13`, `Remove whitespace`, `Decode text`, `Find / Replace`. Es muy útil cuando hay varias capas.

## HTML / CSS / JavaScript

**Contexto:** flags en source, comentarios, CSS oculto, JS ofuscado, endpoints, sourcemaps o atributos HTML.

**Tools:** DevTools, `curl`, `grep`, `js-beautify`, CyberChef.

**Ejemplos:**

```bash
curl -s https://target.ctf/ | tee index.html
grep -Ei "flag|ctf|password|secret|hidden|todo" index.html
grep -Eoi '<!--.*-->' index.html
```

Descargar JS/CSS:

```bash
grep -Eo 'src="[^"]+|href="[^"]+' index.html
curl -s https://target.ctf/static/app.js -o app.js
grep -Ei "flag|ctf|password|secret|api|token" app.js
```

Beautify JS:

```bash
js-beautify app.js -o app.pretty.js
rg -i "flag|ctf|password|secret|token|api" .
```

Sourcemaps:

```bash
curl -s https://target.ctf/static/app.js.map -o app.js.map
grep -Ei "flag|source|webpack|secret" app.js.map
```

## Otros formatos / todo lo demás

**Contexto:** archivo compuesto, firmware, documento, imagen con datos concatenados o formato no identificado.

**Tools:** `exiftool`, `binwalk`, Aperi'Solve, `foremost`, `strings`, `file`, `xxd`.

**Ejemplos:**

```bash
file reto
exiftool reto
binwalk reto
binwalk -Me reto
foremost -i reto -o foremost_out
strings -a reto | grep -Ei "flag|ctf|password|secret"
```

Extraer por offset:

```bash
binwalk reto
dd if=reto of=extraido.bin bs=1 skip=OFFSET
file extraido.bin
```

## Flujo por extensión

| Archivo | Primeras tools | Siguiente paso |
|---|---|---|
| `png` | `zsteg`, `pngcheck`, `stegsolve` | Extraer LSB/chunks |
| `jpg` | `exiftool`, `steghide`, `stegseek` | Crackear passphrase |
| `bmp` | `zsteg`, `stegsolve` | LSB por canales |
| `wav/mp3` | Audacity, `sox`, `ffmpeg` | Espectrograma/canales |
| `txt` | CyberChef, `xxd`, `cat -A` | Invisibles/base64/hex |
| `html` | View source, DevTools, `curl` | Comentarios/CSS/JS |
| otro | `file`, `binwalk`, `exiftool` | Carving/offsets |
