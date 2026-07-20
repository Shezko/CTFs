# Crypto

Primero decide qué tienes: encoding, hash o cifrado. Esa clasificación evita perder tiempo probando herramientas al azar.

## Árbol de decisión

| Qué ves | Probable categoría | Pistas | Tools |
|---|---|---|---|
| Texto raro con charset limitado | Encoding | Base64, hex, rot, morse, base32, URL encoding | CyberChef, dCode, `base64`, `xxd` |
| Cadena hex fija de 32/40/64 chars | Hash | MD5/SHA1/SHA256; no se "descifra", se crackea | hashcat, John, hashid |
| Mensaje + archivo cifrado | Cifrado | Hay key/password/iv o algoritmo mencionado | OpenSSL, CyberChef |
| `n`, `e`, `c` o `.pub` | RSA | Módulo pequeño, e bajo, primos débiles | RsaCtfTool, FactorDB, Sage/Python |

## Encoding

**Contexto:** no hay secreto criptográfico real; solo representaciones: base64, hex, decimal, binary, rot, URL, HTML entities, Morse, Bacon, etc.

**Tools:** CyberChef Magic, dCode Cipher Identifier, `base64`, `xxd`, Python.

**CyberChef GUI:** pegar input, usar `Magic`. Si detecta capas, aplicar receta sugerida. Luego probar `From Base64`, `From Hex`, `URL Decode`, `ROT13`, `From Charcode`, `Gunzip`.

**dCode GUI:** usar Cipher Identifier cuando parece cifrado clásico o símbolos raros.

**Ejemplos terminal:**

Base64:

```bash
echo "ZmxhZ3t0ZXN0fQ==" | base64 -d
base64 -d archivo.b64 > salida.bin
```

Hex:

```bash
echo "666c61677b746573747d" | xxd -r -p
xxd -r -p hex.txt > salida.bin
```

ROT13:

```bash
echo "synt{grfg}" | tr 'A-Za-z' 'N-ZA-Mn-za-m'
```

URL decode con Python:

```bash
python3 - <<'PY'
from urllib.parse import unquote
print(unquote("flag%7Burl_decode%7D"))
PY
```

Varias capas típicas:

```bash
cat data.txt | base64 -d | xxd -r -p
```

## Hash

**Contexto:** cadena de longitud fija. No se descifra: se intenta encontrar el input original probando candidatos.

**Tools:** `hashid`, `hashcat --identify`, hashcat, John, rockyou.

**Identificar:**

```bash
hashid hash.txt
hashcat --identify hash.txt
```

Longitudes frecuentes:

| Longitud hex | Posible tipo |
|---:|---|
| 32 | MD5 / NTLM |
| 40 | SHA1 |
| 64 | SHA256 |
| 128 | SHA512 |

**Crackear con hashcat:**

```bash
hashcat -m 0 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
hashcat -m 100 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
hashcat -m 1400 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
hashcat --show -m 0 hash.txt
```

Modos comunes:

| `-m` | Tipo |
|---:|---|
| `0` | MD5 |
| `100` | SHA1 |
| `1400` | SHA256 |
| `1700` | SHA512 |
| `1000` | NTLM |

**John:**

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
john --show hash.txt
```

## Hash cracking

**Cuándo verlo:**

- Cadena hex de 32, 40, 64 o 128 caracteres.
- El reto pide "find the password".
- Hay `hash.txt` + lista de candidatos.
- Aparece `/etc/shadow`, NTLM, JWT o dump de base de datos.

**Workflow rápido:**

1. Identifica tipo.
2. Elige modo correcto.
3. Usa diccionario.
4. Si falla, prueba reglas o masks.

**Ejemplos:**

```bash
hashcat --identify hash.txt
hashcat -m 0 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
hashcat -m 0 -a 0 hash.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule
hashcat -m 0 -a 3 hash.txt "?l?l?l?l?d?d"
```

Ataques:

| Ataque | `-a` | Ejemplo |
|---|---:|---|
| Diccionario | `0` | rockyou |
| Combinator | `1` | wordlist + wordlist |
| Mask/bruteforce | `3` | `?l?l?l?d?d` |
| Wordlist + mask | `6` | `password` + `?d?d` |
| Mask + wordlist | `7` | prefijo numérico |

John con formatos:

```bash
john --list=formats | grep -i sha
john --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

## Cifrado simétrico

**Contexto:** te dan ciphertext + password/key/iv o el reto menciona AES/DES/ChaCha. Si no tienes clave ni debilidad, no hay magia: busca pistas en otros archivos.

**Tools:** OpenSSL, CyberChef, Python.

**OpenSSL con password:**

```bash
openssl enc -aes-256-cbc -d -pbkdf2 -in cipher.bin -out plain.txt -pass pass:miPassword
```

Si está en base64:

```bash
openssl enc -aes-256-cbc -d -a -pbkdf2 -in cipher.b64 -out plain.txt -pass pass:miPassword
```

Con key e IV hex:

```bash
openssl enc -aes-128-cbc -d -in cipher.bin -out plain.txt \
  -K 00112233445566778899aabbccddeeff \
  -iv 0102030405060708090a0b0c0d0e0f10
```

## RSA con primos débiles

**Cuándo verlo:**

- Te dan `public.pem`, `key.pub`, `n`, `e`, `c`.
- `n` es pequeño, por ejemplo menor a 256 bits.
- `p` y `q` están dados o son factorizables.
- `e = 3` o exponente bajo.
- Hay varios ciphertexts con mismo `n` o mismo mensaje.

**Idea:** RSA depende de que factorizar `n = p*q` sea difícil. Si `n` es pequeño o FactorDB ya conoce sus factores, puedes recuperar `p` y `q`, calcular `phi`, obtener `d` y descifrar.

**Tools:** FactorDB, RsaCtfTool, SageMath, Python (`pycryptodome`, `gmpy2`).

**Workflow rápido:**

1. Copiar `n` en FactorDB.
2. Si está factorizado, obtener `p` y `q`.
3. Usar RsaCtfTool con public key/ciphertext.
4. Si hace falta, resolver manual con Python.

**RsaCtfTool con public key:**

```bash
python3 RsaCtfTool.py --publickey public.pem --private
python3 RsaCtfTool.py --publickey public.pem --uncipherfile cipher.bin
python3 RsaCtfTool.py --publickey public.pem --uncipher 123456789
```

Crear public key desde `n` y `e`:

```bash
python3 RsaCtfTool.py --createpub --n 123456789 --e 65537 > public.pem
```

Manual con Python si tienes `p`, `q`, `e`, `c`:

```python
from Crypto.Util.number import long_to_bytes, inverse

n = 0
e = 65537
c = 0
p = 0
q = 0

phi = (p - 1) * (q - 1)
d = inverse(e, phi)
m = pow(c, d, n)
print(long_to_bytes(m))
```

Caso `e = 3` y mensaje pequeño sin padding:

```python
import gmpy2
from Crypto.Util.number import long_to_bytes

c = 0
m, exact = gmpy2.iroot(c, 3)
print(exact, long_to_bytes(int(m)))
```

## OpenSSL

**Contexto:** OpenSSL sirve para base64, hashes, certificados, llaves RSA y cifrado simétrico. En CTF es especialmente útil cuando CyberChef falla o cuando quieres reproducir la solución en terminal.

### Decodificar Base64

```bash
openssl base64 -d -in archivo.b64 -out archivo.bin
openssl enc -base64 -d -in archivo.b64 -out archivo.bin
```

Verificar salida:

```bash
file archivo.bin
strings -a archivo.bin | head
```

### Generar/verificar hashes

```bash
openssl dgst -md5 archivo.txt
openssl dgst -sha1 archivo.txt
openssl dgst -sha256 archivo.txt
```

Solo el hash:

```bash
openssl dgst -sha256 -r archivo.txt
```

HMAC:

```bash
openssl dgst -sha256 -hmac "secret" archivo.txt
```

### Descifrar AES con password

```bash
openssl enc -aes-256-cbc -d -pbkdf2 -in cipher.bin -out plain.txt -pass pass:miPassword
```

Si el ciphertext está en base64:

```bash
openssl enc -aes-256-cbc -d -a -pbkdf2 -in cipher.b64 -out plain.txt -pass pass:miPassword
```

Si el reto fue cifrado con OpenSSL viejo, a veces no usó PBKDF2:

```bash
openssl enc -aes-256-cbc -d -in cipher.bin -out plain.txt -pass pass:miPassword
```

### Revisar llaves RSA

```bash
openssl rsa -in private.pem -check
openssl rsa -in private.pem -pubout -out public.pem
openssl rsa -pubin -in public.pem -text -noout
```

Extraer módulo de public key:

```bash
openssl rsa -pubin -in public.pem -modulus -noout
```

### Descifrar con llave privada RSA

```bash
openssl pkeyutl -decrypt -inkey private.pem -in cipher.bin -out plain.txt
```

Si es un reto antiguo con `rsautl` disponible:

```bash
openssl rsautl -decrypt -inkey private.pem -in cipher.bin -out plain.txt
```

## Cifrados clásicos

**Contexto:** texto alfabético, frecuencia rara, pista histórica o símbolos.

**Tools:** dCode, CyberChef, quipqiup, Python.

**Casos comunes:**

| Cifrado | Pista | Tool |
|---|---|---|
| Caesar/ROT | letras legibles tras rotar | CyberChef ROT13/ROT47, dCode |
| Vigenere | palabra clave | dCode, CyberChef |
| Morse | puntos/rayas o audio | CyberChef, dCode |
| Bacon | grupos A/B | dCode |
| XOR | bytes raros, key corta | CyberChef XOR Brute Force |

**XOR con CyberChef:** probar `XOR Brute Force` y buscar salida con `flag`, `ctf`, texto legible.

**XOR single-byte con Python:**

```python
data = bytes.fromhex("1f0a...")
for k in range(256):
    out = bytes(b ^ k for b in data)
    if b"flag" in out.lower() or b"ctf" in out.lower():
        print(k, out)
```
