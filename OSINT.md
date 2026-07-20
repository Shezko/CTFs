# OSINT

El objetivo no es usar una sola herramienta, sino construir una cadena de pivotes: usuario -> perfiles -> repos -> commits -> caches -> imágenes -> ubicación -> documentos -> dorks.

## Checklist inicial

1. Copiar exactamente nombres, aliases, correos, dominios, frases y handles.
2. Buscar el mismo identificador en redes, GitHub, buscadores y caches.
3. Revisar commits, historial, comentarios, issues y forks.
4. Si hay imagen, buscar metadatos y búsqueda inversa.
5. Si hay web, revisar Wayback/cache y rutas antiguas.
6. Documentar cada pivote: evidencia, URL, timestamp y razón.

## Perfil, bios, repos y comentarios

**Contexto:** tienes un username, nombre, alias, correo o frase de bio.

**Tools:** Sherlock, WhatsMyName, Maigret, buscadores, GitHub, Reddit, X/Twitter, LinkedIn.

**Ejemplos:**

```bash
sherlock usuario
sherlock usuario --csv
sherlock usuario --timeout 10
```

Con varios usuarios:

```bash
sherlock usuario1 usuario2 usuario3
```

Búsquedas manuales útiles:

```text
"usuarioExacto"
"usuarioExacto" "CTF"
"correo@dominio.com"
"Nombre Apellido" "github"
```

Qué mirar:

- Bio repetida en varias plataformas.
- Links a portfolio, repos, blog o Discord.
- Comentarios antiguos con datos personales.
- Nombres de usuario con pequeñas variaciones: `user`, `user_`, `user.dev`, `user1337`.

## Perfiles, GitHub y commits

**Contexto:** GitHub suele filtrar emails, nombres reales, endpoints, tokens falsos/reales de CTF, ramas viejas y archivos borrados.

**Tools:** GitHub Search, `git`, GitHub API, buscador web.

**Búsqueda en GitHub:**

```text
user:usuario
org:organizacion
repo:usuario/repo flag
repo:usuario/repo password
repo:usuario/repo filename:.env
repo:usuario/repo extension:py secret
```

Buscar commits por autor:

```text
author:usuario
author-email:correo@dominio.com
committer:usuario
```

Clonar y revisar historial:

```bash
git clone https://github.com/usuario/repo.git
cd repo
git log --oneline --all
git log --all --stat
git log --all --author="usuario"
git show <commit>
```

Buscar secretos en todo el historial:

```bash
git grep -n -i "flag\|password\|secret\|token\|api_key" $(git rev-list --all)
```

Ver archivos borrados:

```bash
git log --diff-filter=D --summary
git checkout <commit_anterior> -- ruta/borrada.txt
```

Ver cambios de una línea o archivo:

```bash
git log --all -- ruta/archivo
git blame ruta/archivo
git show <commit>:ruta/archivo
```

## Wayback y cache

**Contexto:** una página fue modificada, el flag estuvo en una versión vieja, robots.txt cambió o existían endpoints antiguos.

**Tools:** Wayback Machine, `waybackurls`, `gau`, cache de buscadores.

**Ejemplos:**

```bash
echo "example.com" | waybackurls > urls.txt
cat urls.txt | sort -u
grep -Ei "flag|admin|backup|old|dev|api|login|robots|\.zip|\.bak" urls.txt
```

Con varios dominios:

```bash
cat domains.txt | waybackurls | sort -u > wayback_all.txt
```

Filtrar extensiones interesantes:

```bash
grep -Ei "\.(zip|tar|gz|sql|bak|old|txt|pdf|json|env)(\?|$)" urls.txt
```

Uso manual:

```text
https://web.archive.org/web/*/https://example.com/*
https://webcache.googleusercontent.com/search?q=cache:https://example.com
```

## Maps y Street View

**Contexto:** tienes una foto de calle, edificio, montaña, señal, placa, cartel, idioma, vegetación o sombra.

**Tools:** Google Maps, Street View, Google Earth, OpenStreetMap, Mapillary, SunCalc.

**Método rápido:**

1. Extrae texto visible: nombres de tiendas, calles, dominios, teléfonos, idioma.
2. Busca combinaciones cortas entre comillas.
3. Usa Maps para verificar geometría: cruces, fachadas, montañas, semáforos.
4. Confirma con Street View o Mapillary.

**Dorks útiles:**

```text
"nombre de tienda" "ciudad"
"texto exacto del cartel"
"teléfono" "dirección"
"calle 1" "calle 2"
```

**Pistas visuales que valen oro:**

- Formato de placas.
- Idioma y dominio nacional (`.cr`, `.mx`, `.ar`, `.fr`).
- Señales de tránsito.
- Lado de manejo.
- Vegetación/clima.
- Sombras: pueden sugerir hemisferio y hora aproximada, pero no lo uses como única prueba.

## Ubicación por imagen: Yandex, Lens y TinEye

**Contexto:** tienes una imagen y necesitas encontrar origen, lugar, autor o copias previas.

**Tools:** Google Lens, Yandex Images, TinEye, Bing Visual Search, ExifTool.

**Ejemplos terminal previos:**

```bash
file imagen.jpg
exiftool imagen.jpg
exiftool -n -GPSLatitude -GPSLongitude imagen.jpg
```

**Uso manual recomendado:**

- Google Lens: bueno para objetos, lugares populares y productos.
- Yandex: suele ser fuerte con rostros, fachadas y coincidencias visuales aproximadas.
- TinEye: bueno para encontrar copias antiguas y ordenarlas por fecha.

**Búsqueda complementaria:**

```text
"nombre visible" "ciudad"
"marca del cartel" "calle"
"texto de playera" "evento"
```

## Frase, site, filetype e intitle

**Contexto:** tienes una frase única, un fragmento de flag, un nombre de archivo, un dominio o quieres encontrar documentos filtrados.

**Tools:** Google/Bing/DuckDuckGo dorks.

**Operadores clave:**

| Operador | Uso |
|---|---|
| `"frase exacta"` | Coincidencia exacta |
| `site:dominio.com` | Limita a un sitio |
| `filetype:pdf` | Busca extensión/tipo |
| `intitle:admin` | Palabra en título |
| `inurl:backup` | Palabra en URL |
| `-palabra` | Excluye ruido |

**Ejemplos CTF:**

```text
"flag{" site:example.com
"CTF{" -writeup
site:example.com filetype:pdf
site:example.com intitle:index.of
site:example.com inurl:backup
site:example.com ext:sql OR ext:bak OR ext:old
"nombre completo" "github" -linkedin
```

## Dominios, DNS y subdominios

**Contexto:** tienes un dominio y necesitas ampliar superficie sin explotar nada.

**Tools:** `whois`, `dig`, `nslookup`, crt.sh, SecurityTrails, `subfinder`, `amass`.

**Ejemplos:**

```bash
whois example.com
dig example.com A
dig example.com MX
dig TXT example.com
dig +short NS example.com
```

Certificados:

```text
https://crt.sh/?q=%25.example.com
```

Subdominios pasivos:

```bash
subfinder -d example.com -silent
amass enum -passive -d example.com
```

## Correos y usernames

**Contexto:** un correo puede pivotar a GitHub, Gravatar, breaches de CTF, commits o perfiles.

**Tools:** GitHub Search, `git log`, Holehe, Epieos, HaveIBeenPwned, Gravatar.

**Ejemplos:**

```bash
git log --all --format='%an <%ae>' | sort -u
```

Buscar en GitHub:

```text
"correo@dominio.com"
author-email:correo@dominio.com
```

Buscar avatar Gravatar:

```bash
printf "correo@dominio.com" | tr '[:upper:]' '[:lower:]' | md5sum
```

Luego:

```text
https://www.gravatar.com/avatar/<md5>
```

## Plantilla de notas para OSINT

```md
## Hipótesis
- Alias inicial:
- Fuente inicial:
- Pivote:

## Evidencia
- URL:
- Captura / fecha:
- Por qué importa:

## Resultado
- Dato encontrado:
- Cómo se confirma:
- Flag / respuesta:
```
