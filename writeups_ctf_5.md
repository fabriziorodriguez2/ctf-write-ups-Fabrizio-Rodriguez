# Write-ups CTF — 5 desafíos

En este documento explico cómo resolví los cinco desafíos, qué herramientas utilicé y cuál era la vulnerabilidad o el error que había que aprovechar en cada caso.

---

## 1. Fool the Lockout — picoCTF 2026

**Categoría:** Web Exploitation

### Objetivo

Tenía que entrar a una página con login usando un archivo de 100 combinaciones de usuarios y contraseñas. El problema era que la web bloqueaba los intentos repetidos para evitar fuerza bruta.

### Cómo lo resolví

Primero revisé el código de Flask para entender cómo funcionaba el bloqueo. Vi que el sistema permitía 10 intentos cada 30 segundos y, si me pasaba, bloqueaba la IP por 120 segundos.

La forma de evitarlo era bastante simple: hacer **9 intentos, esperar a que se reiniciara el contador y seguir**. Para no probar las 100 credenciales a mano, usé un script de Python con `requests` que leía el archivo, enviaba peticiones al login y esperaba **32 segundos** después de cada grupo de 9 intentos.

El script también revisaba si la respuesta era un **HTTP 302**, que indicaba una redirección después de un login correcto. Así encontré las credenciales válidas, entré y obtuve la flag.

### Conclusión

El sitio tenía un límite de intentos, pero se podía esquivar esperando entre grupos de peticiones. Aprendí que un rate limiting mal diseñado no alcanza para frenar un ataque de prueba de credenciales.

---

## 2. SQL Map1 — picoCTF 2026

**Categoría:** Web Exploitation

### Objetivo

Tenía que conseguir la flag de una web que tenía un buscador vulnerable a SQL Injection. La idea era aprovechar ese error para acceder a información de la base de datos y después entrar con otro usuario.

### Cómo lo resolví

Primero me registré, inicié sesión y probé una inyección básica en el buscador:

```sql
' OR 1=1 --
```

La búsqueda devolvió un montón de resultados, incluyendo flags falsas. Entonces seguí revisando la base de datos SQLite hasta encontrar una tabla llamada `users`.

Para sacar los usuarios y sus hashes de contraseña utilicé otra inyección:

```sql
' UNION SELECT 1, GROUP_CONCAT(username || ': ' || password) FROM users --
```

Ahí vi que las contraseñas estaban guardadas como hashes **MD5**. Recuperé la contraseña del usuario `ctf-player`, que era `dyesebel`, cerré mi sesión e ingresé con esas credenciales. Así pude entrar al área protegida y conseguir la flag real.

### Conclusión

Acá se combinaron dos problemas: la SQL Injection dejaba consultar datos que no debería, y MD5 no era adecuado para almacenar contraseñas. Con ambos errores pude obtener acceso a otra cuenta.

---

## 3. No FA — picoCTF 2026

**Categoría:** Web Exploitation

### Objetivo

Tenía que entrar como `admin` a una web y pasar una verificación de dos factores (2FA) para conseguir la flag.

### Cómo lo resolví

Primero revisé el código de Flask y la base de datos. Vi que las contraseñas se comparaban usando **SHA-256 sin salt**, por lo que podía intentar recuperar la contraseña a partir del hash.

Consulté la base con SQLite:

```bash
sqlite3 users.db ".dump"
```

Después analicé el hash de `admin` con una herramienta de diccionario, como Hashcat (modo `1400` para SHA-256), y recuperé la contraseña: `apple@123`.

Con eso pude iniciar sesión, pero la web me pidió un código de 4 dígitos. Revisando `app.py` encontré que el OTP se guardaba en `session['otp_secret']`.

El detalle importante es que **las cookies de sesión de Flask están firmadas, pero no cifradas** por defecto. Entonces abrí las herramientas de desarrollador del navegador, busqué la cookie `session` y decodifiqué su contenido en Python (Base64 y, si hacía falta, zlib). Ahí estaba el valor de `otp_secret`.

Ingresé ese código antes de que caducara, completé el 2FA y la página me mostró la flag por estar autenticado como administrador.

### Conclusión

El segundo factor no servía de mucho porque el propio navegador recibía el código secreto dentro de la cookie. Además, guardar contraseñas con SHA-256 sin salt facilitaba recuperarlas por diccionario. Me quedó claro que **firmar una cookie no significa ocultar su contenido**.

---

## 4. Stego — Clave RSA oculta en una imagen

**Categoría:** Esteganografía / Criptografía

### Objetivo

Me dieron dos archivos, `image.jpg` y `flag.enc`. Tenía que encontrar cómo descifrar el segundo para obtener la flag.

### Cómo lo resolví

Primero inspeccioné ambos archivos con:

```bash
file image.jpg flag.enc
```

El comando mostró que la imagen JPEG tenía un comentario con una cadena hexadecimal muy larga. Me llamó la atención porque empezaba por `2d2d2d2d2d`, que en ASCII representa `-----`, algo típico del encabezado de una clave PEM.

Convertí esa cadena hexadecimal a texto con Python usando `bytes.fromhex(...)` y obtuve una **clave privada RSA**. La guardé en `clave.pem`.

Después intenté descifrar `flag.enc` con esa clave. Al principio probé OAEP con SHA-256, pero falló. Probé también OAEP con SHA-1 y finalmente **PKCS#1 v1.5**, que sí funcionó.

La flag que obtuve fue:

```text
academy{rs4_k3y_1n_1mg_21d0a083}
```

### Conclusión

La clave no estaba escondida en los píxeles, sino en los **metadatos de la imagen**. Aprendí que antes de probar métodos complicados de esteganografía conviene revisar los comentarios y la información del archivo.

---

## 5. Failure Failure

**Categoría:** Web / Configuración de servidores

### Objetivo

La web usaba Flask detrás de **HAProxy**, con dos servidores: `s1` como principal y `s2` como respaldo. La flag solamente aparecía cuando respondía `s2`, así que tenía que conseguir que el balanceador dejara de usar `s1`.

### Cómo lo resolví

Primero revisé `haproxy.cfg`. Ahí vi que HAProxy hacía un **health check** con `GET /` cada 2 segundos y esperaba una respuesta HTTP `200`. Si el servidor principal fallaba dos chequeos seguidos, HAProxy pasaba al backup.

Después miré `app.py` y encontré el error: el límite de **300 peticiones por minuto** usaba una clave fija llamada `global`, en vez de tener un contador por IP. Esto hacía que todas las peticiones compartieran el mismo límite, incluso las de verificación de HAProxy.

Entonces armé un script llamado `loop.py` usando Python, `ThreadPoolExecutor` y `urllib` para enviar varias peticiones concurrentes al servidor del desafío. El objetivo era superar el límite global y provocar respuestas distintas de `200`.

Una vez agotado el límite, los health checks empezaron a fallar y HAProxy marcó a `s1` como caído. Eso hizo que el tráfico pasara a `s2`, que tenía configurada la variable `IS_BACKUP=yes`. Al consultar nuevamente `/`, recibí la flag.

También comprobé que, después de recuperarse el servicio principal y superar los chequeos necesarios, el tráfico volvía a `s1`.

### Conclusión

El problema no era que HAProxy cambiara al servidor de respaldo, porque eso era lo esperado. El error estaba en que **el rate limiting era global y afectaba también a los health checks**. Así pude provocar un failover sin tener que comprometer directamente el servidor principal.
