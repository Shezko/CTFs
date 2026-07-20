# Forensics

La idea es primero clasificar la evidencia, luego aplicar herramientas por contexto y finalmente reconstruir la historia: qué archivo era, qué contiene, qué fue borrado, qué tráfico pasó, qué proceso corrió o qué secreto quedó escondido.

## Checklist inicial

1. Identificar tipo real, aunque la extensión mienta.
2. Sacar strings y metadatos.
3. Buscar archivos embebidos o comprimidos.
4. Si es imagen de disco, listar particiones y archivos borrados.
5. Si es RAM, listar procesos, comandos, conexiones y archivos.
6. Si es PCAP, filtrar protocolos y reconstruir streams/objetos.
7. Si hay password/hash, convertir al formato correcto y crackear.

```bash
file evidencia
strings -a evidencia | head
strings -a evidencia | grep -Ei "flag|ctf|password|key|user|admin"
exiftool evidencia
binwalk evidencia
```

## Unknown / archivo desconocido

**Contexto:** cuando recibes un archivo sin extensión, con extensión falsa o con contenido mezclado.

**Tools:** `file`, `strings`, `exiftool`, `binwalk`, `xxd`, `hexdump`, `grep`.

**Ejemplos:**

```bash
file reto.bin
xxd -l 64 reto.bin
hexdump -C reto.bin | head
strings -a reto.bin | grep -Ei "flag|ctf|pass|key|http|zip|png|pdf"
exiftool reto.bin
binwalk reto.bin
binwalk -e reto.bin
```

Notas útiles:

- Si `file` dice `data`, no significa que no haya nada: prueba `binwalk`, `strings` y mira magic bytes con `xxd`.
- Si `binwalk` encuentra offsets pero no extrae bien, puedes cortar manualmente con `dd`.

```bash
dd if=reto.bin of=extraido.zip bs=1 skip=12345
file extraido.zip
```

## Texto en archivos

**Contexto:** primer barrido para encontrar flags, rutas, usuarios, tokens, comandos o pistas.

**Tools:** `strings`, `grep`, `ripgrep`, `xxd`, `less`.

**Ejemplos:**

```bash
strings -a archivo.bin > strings.txt
grep -Ei "flag|ctf|password|passwd|secret|token|key" strings.txt
grep -Eo "CTF\{[^}]+\}|flag\{[^}]+\}" strings.txt
```

Si extraiste muchos archivos:

```bash
rg -i "flag|ctf|password|secret|token|key" ./extraido
```

## ZIP con password

**Contexto:** ZIP protegido, normalmente se resuelve con `fcrackzip` o `zip2john` + `john`. `rockyou.txt` suele estar en Kali como `/usr/share/wordlists/rockyou.txt.gz`.

**Tools:** `fcrackzip`, `zip2john`, `john`, `rockyou.txt`, `7z`.

**Ejemplos:**

```bash
gzip -d /usr/share/wordlists/rockyou.txt.gz
fcrackzip -v -u -D -p /usr/share/wordlists/rockyou.txt secreto.zip
```

Con John:

```bash
zip2john secreto.zip > zip.hash
john --wordlist=/usr/share/wordlists/rockyou.txt zip.hash
john --show zip.hash
7z x secreto.zip
```

Uso típico:

- `zip2john`: convierte el ZIP a un hash entendible por John.
- `john --wordlist`: prueba candidatos del diccionario.
- `john --show`: muestra la password si ya fue crackeada.

## Cracking general

**Contexto:** hashes sueltos, passwords de archivos, hashes extraidos de ZIP/PDF/Office o credenciales encontradas en dumps.

**Tools:** `hashcat`, `john`, `hashid`, `hash-identifier`, `rockyou.txt`.

**Ejemplos:**

```bash
hashid hash.txt
hashcat --identify hash.txt
hashcat -m 0 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
hashcat -m 100 -a 0 sha1.txt /usr/share/wordlists/rockyou.txt
hashcat -m 1000 -a 0 ntlm.txt /usr/share/wordlists/rockyou.txt
hashcat --show -m 0 hash.txt
```

Modos comunes de `hashcat`:

| Modo | Tipo |
|---:|---|
| `0` | MD5 |
| `100` | SHA1 |
| `1400` | SHA256 |
| `1000` | NTLM |
| `1800` | sha512crypt Linux |
| `22000` | WPA/WPA2 |

John como alternativa:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
john --show hash.txt
```

## Imagen de disco

**Contexto:** te dan `.dd`, `.img`, `.raw`, `.E01`, VHD o una partición. Hay que listar particiones, recuperar borrados, extraer archivos y revisar historial.

**Tools principales:** Autopsy, The Sleuth Kit (`mmls`, `fls`, `icat`, `tsk_recover`), `foremost`, `bulk_extractor`, `mount`.

**Tools gráficas:** Autopsy es ideal para navegar filesystem, timeline, archivos borrados e historial. No necesita sintaxis: crear caso, agregar imagen, analizar.

**Ejemplos con Sleuth Kit:**

```bash
mmls disk.img
fsstat -o 2048 disk.img
fls -o 2048 disk.img
fls -r -o 2048 disk.img
fls -rd -o 2048 disk.img
```

Extraer un archivo por inode:

```bash
icat -o 2048 disk.img 12345 > recuperado.txt
file recuperado.txt
```

Recuperar estructura completa:

```bash
mkdir recovered
tsk_recover -o 2048 disk.img recovered/
rg -i "flag|password|secret|ctf" recovered/
```

Montar read-only, si el formato lo permite:

```bash
sudo mkdir /mnt/ctf
sudo mount -o ro,loop,offset=$((2048*512)) disk.img /mnt/ctf
```

Tip: el `2048` sale de `mmls` y representa el sector de inicio de la partición. El offset en bytes suele ser `sector * 512`.

## Archivos borrados / carving

**Contexto:** no hay filesystem claro, hay archivos borrados o necesitas recuperar por firmas.

**Tools:** `foremost`, `binwalk`, `bulk_extractor`, `photorec`.

**Ejemplos:**

```bash
foremost -i disk.img -o foremost_out
foremost -t jpg,png,pdf,zip -i disk.img -o carved
binwalk -e disk.img
bulk_extractor -o bulk_out disk.img
```

Despues de recuperar:

```bash
find carved -type f -exec file {} \;
rg -i "flag|ctf|password|secret" carved/ foremost_out/ bulk_out/
```

## Memoria RAM

**Contexto:** dumps `.raw`, `.mem`, `.vmem`. Se buscan procesos, comandos ejecutados, conexiones, archivos cargados, credenciales o malware.

**Tools:** Volatility 3, Volatility 2, `strings`.

**Workflow Volatility 3:**

```bash
python3 vol.py -f memory.raw windows.info
python3 vol.py -f memory.raw windows.pslist
python3 vol.py -f memory.raw windows.pstree
python3 vol.py -f memory.raw windows.cmdline
python3 vol.py -f memory.raw windows.netscan
```

Buscar comandos/pistas:

```bash
python3 vol.py -f memory.raw windows.consoles
python3 vol.py -f memory.raw windows.cmdline | grep -Ei "flag|pass|powershell|cmd|curl|certutil"
strings -a memory.raw | grep -Ei "flag|ctf|password|secret"
```

Dump de proceso sospechoso:

```bash
python3 vol.py -f memory.raw windows.pslist
python3 vol.py -f memory.raw windows.dumpfiles --pid 1337
```

Volatility 2, si el reto viejo lo requiere:

```bash
volatility -f memory.raw imageinfo
volatility -f memory.raw --profile=Win7SP1x64 pslist
volatility -f memory.raw --profile=Win7SP1x64 cmdline
volatility -f memory.raw --profile=Win7SP1x64 netscan
```

## Trafico de red / PCAP

**Contexto:** capturas `.pcap` o `.pcapng`. Se buscan credenciales, archivos transferidos, endpoints, DNS, HTTP, TCP streams y objetos exportables.

**Tools principales:** Wireshark, `tshark`, NetworkMiner.

**Wireshark GUI:** abrir PCAP, usar filtros como `http`, `dns`, `tcp.stream eq 0`, `ip.addr == 10.10.10.5`, `frame contains "flag"`. Para reconstruir conversaciones: `Analyze -> Follow -> TCP Stream/HTTP Stream`. Para archivos: `File -> Export Objects -> HTTP`.

**Ejemplos con tshark:**

```bash
tshark -r captura.pcap
tshark -r captura.pcap -Y "http"
tshark -r captura.pcap -Y "dns" -T fields -e dns.qry.name
tshark -r captura.pcap -Y "http.request" -T fields -e http.host -e http.request.uri
tshark -r captura.pcap -Y "frame contains \"flag\""
```

Extraer objetos HTTP:

```bash
mkdir http_objects
tshark -r captura.pcap --export-objects "http,http_objects"
find http_objects -type f -exec file {} \;
rg -i "flag|ctf|password|secret" http_objects/
```

Separar un stream:

```bash
tshark -r captura.pcap -Y "tcp.stream eq 0" -w stream0.pcap
tshark -r captura.pcap -q -z conv,tcp
```

## PDF

**Contexto:** PDF con JavaScript, objetos escondidos, streams comprimidos, adjuntos o texto oculto.

**Tools:** `pdfid.py`, `pdf-parser.py`, `qpdf`, `exiftool`, `strings`, `binwalk`.

**Ejemplos:**

```bash
pdfid.py sospechoso.pdf
pdfid.py -a sospechoso.pdf
pdf-parser.py sospechoso.pdf
pdf-parser.py --search flag sospechoso.pdf
pdf-parser.py --search /JavaScript sospechoso.pdf
```

Ver y extraer objetos:

```bash
pdf-parser.py -o 5 sospechoso.pdf
pdf-parser.py -o 5 -f sospechoso.pdf > obj5.bin
file obj5.bin
strings -a obj5.bin
```

Descomprimir/normalizar:

```bash
qpdf --qdf --object-streams=disable sospechoso.pdf limpio.pdf
strings -a limpio.pdf | grep -Ei "flag|ctf|password|secret"
binwalk -e sospechoso.pdf
```

## Metadata

**Contexto:** imagen, PDF, audio, video o documento con autor, coordenadas, timestamps, software usado o comentarios.

**Tools:** `exiftool`, `exiv2`, `mediainfo`.

**Ejemplos:**

```bash
exiftool imagen.jpg
exiftool -gps* imagen.jpg
exiftool -Comment -Description -Artist -Author archivo
exiftool -time:all archivo
exiv2 pr imagen.jpg
```

Si aparece GPS:

```bash
exiftool -n -GPSLatitude -GPSLongitude imagen.jpg
```

## Reconstruir historia

**Contexto:** cuando el reto pide responder qué pasó, quién lo hizo o recuperar la flag desde varios artefactos.

**Tools:** Autopsy timeline, `mactime`, `fls -m`, Volatility, Wireshark.

**Ejemplo con timeline de Sleuth Kit:**

```bash
fls -r -m / -o 2048 disk.img > bodyfile.txt
mactime -b bodyfile.txt > timeline.csv
grep -Ei "flag|secret|download|history|bash|powershell" timeline.csv
```

Orden sugerido:

1. Evidencia: qué tipo de archivo es.
2. Herramienta correcta: segun disco/RAM/PCAP/PDF/ZIP.
3. Reconstruir historia: timestamps, usuario, acción, artefacto y flag.
