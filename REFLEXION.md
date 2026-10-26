# Reflexión · CI/CD con Firebase Hosting y hardening de headers

- Repo: https://github.com/Juanceto01-jpg/Practica_CICD_Firebase
- Producción: https://practica-firebase-cicd-8a670.web.app
- PR mergeado con el comentario del bot y la URL de preview: PR #2 (`feat/headers`)

## Evidencia: medición de headers

### Antes (B1, producción sin cambios)

```
▶ HTML: https://practica-firebase-cicd-8a670.web.app/
  status 200
  ✘ content-security-policy (falta)
  ✔ strict-transport-security: max-age=31556926; includeSubDomains; preload
  ✘ x-content-type-options (falta)
  ✘ x-frame-options (falta)
  ✘ referrer-policy (falta)
  ✘ permissions-policy (falta)
  · cache-control: max-age=3600

▶ Asset estático: /_next/static/chunks/1x6obpmlu1_c2.js
  ✘ cache-control: max-age=3600

❌ 6 control(es) pendiente(s).
```

securityheaders.com: **A**

### Después (B5, producción tras mergear el PR #2)

```
▶ HTML: https://practica-firebase-cicd-8a670.web.app/
  status 200
  ✔ content-security-policy: default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self'; img-src 'self' https://picsum.photos …
  ✔ strict-transport-security: max-age=31556926; includeSubDomains; preload
  ✔ x-content-type-options: nosniff
  ✔ x-frame-options: DENY
  ✔ referrer-policy: strict-origin-when-cross-origin
  ✔ permissions-policy: camera=(), microphone=(), geolocation=()
  · cache-control: no-cache

▶ Asset estático: /_next/static/chunks/1x6obpmlu1_c2.js
  ✔ cache-control: public, max-age=31536000, immutable

✅ Todos los controles pasaron.
```

securityheaders.com: **COMPLETAR con la nota del escaneo nuevo**

Nota: el HSTS que se ve en producción es el que Firebase agrega por defecto en `web.app` (`max-age=31556926; includeSubDomains; preload`); el valor que puse en `firebase.json` (`max-age=31536000`) no se nota ahí.

## Preguntas de reflexión

**1. ¿Por qué el preview tiene vencimiento? ¿Qué riesgos tendría uno que no vence nunca?**

Cada PR crea un canal con una URL pública. Si no vencieran, se acumularían versiones viejas de la app, algunas con código sin revisar o con bugs y vulnerabilidades ya corregidas en `main`, accesibles para cualquiera que tenga el link. Además aumentan la superficie de ataque y el desorden en el proyecto. Con `expires: 3d` el canal desaparece solo (en la consola de Firebase se ve "Hasta el 29/10").

**2. ¿Qué diferencia hay entre desplegar y liberar? ¿Dónde aparece cada concepto?**

Desplegar es poner una versión del código en la infraestructura. Liberar es hacerla disponible para los usuarios finales. En este flujo, cada PR **despliega** a un preview channel pero no **libera** nada, porque producción no cambia. El merge a `main` publica en el canal `live`, que es la liberación. El rollback también es una liberación: vuelve a publicar una versión anterior como la vigente.

**3. ¿Por qué la clave de la service account va en un secreto y no en el repo? ¿Qué podría hacer quien la obtenga? ¿Cómo limitarías el daño?**

El repo es público, así que cualquier clave commiteada quedaría expuesta. Quien la obtenga puede desplegar contenido arbitrario en mi sitio (por ejemplo, reemplazar la app por una página falsa) y, según los permisos que tenga la cuenta, hacer más cosas dentro del proyecto. Para limitar el daño: darle solo el rol mínimo necesario (Hosting), rotar o revocar la clave si se filtra, usar environments con aprobación manual y, mejor aún, reemplazar la clave de larga duración por federación de identidad (Workload Identity) para no tener una clave guardada.

**4. La CSP rompió la app: ¿por qué? ¿Qué opciones evalué y qué costo de seguridad tiene cada una?**

Con la CSP estricta aparecieron cinco errores en la consola:
- Dos de `img-src 'self'`: la imagen de portada viene de `picsum.photos`, un origen externo.
- Tres de `script-src 'self'`: Next.js mete scripts inline en el HTML (los que arrancan React), y `'self'` solo permite archivos externos. Sin ellos el filtro de `/cursos` no funciona.

Opciones para los scripts:
- `'unsafe-inline'`: es la que usé. Es simple y funciona, pero si un atacante lograra inyectar un `<script>` en el HTML, el navegador lo ejecutaría (reduce la protección contra XSS).
- Hashes `'sha256-...'`: permiten solo los scripts exactos, pero el contenido cambia en cada build, así que habría que calcularlos en el pipeline después de `next build`.
- Nonce: requiere un servidor que genere uno distinto por request; con Firebase Hosting (solo estático) no es posible.

Para la imagen agregué a `img-src` los orígenes exactos desde los que se carga, en vez de abrirlo a cualquier origen.

**5. Para cada header de B2, ¿qué ataque mitiga?**

- `Strict-Transport-Security`: evita que el navegador use HTTP en visitas futuras (ataques de downgrade y SSL stripping en redes no confiables).
- `X-Content-Type-Options: nosniff`: impide que el navegador "adivine" el tipo de un archivo; evita que un archivo subido como texto o imagen se ejecute como script.
- `X-Frame-Options: DENY`: impide que otro sitio embeba el mío en un iframe (clickjacking).
- `Referrer-Policy`: limita qué parte de la URL se envía a otros sitios al navegar, para no filtrar rutas o parámetros sensibles.
- `Permissions-Policy`: deshabilita APIs del navegador que la página no usa (cámara, micrófono, geolocalización), para que código inyectado o de terceros no las aproveche.

**6. ¿Por qué los assets de `/_next/static/` pueden cachearse un año y el HTML no?**

Los archivos de `/_next/static/` tienen un hash del contenido en el nombre: si el contenido cambia, cambia la URL, así que un archivo con esa URL nunca cambia y puede cachearse como `immutable`. El HTML tiene una URL fija y es el que referencia a los assets vigentes; si se cacheara sin revalidar, después de un deploy o un rollback los usuarios seguirían viendo la versión vieja. Por eso va con `no-cache`.

**7. Si la app necesitara SSR, ¿qué cambiaría en la arquitectura, en la CSP y en el costo?**

Firebase Hosting con plan Spark solo sirve archivos estáticos. Con SSR habría que usar App Hosting, que corre sobre Cloud Run y exige el plan Blaze (cuenta de facturación), es decir, deja de ser gratis. Con un servidor sí se podrían usar nonces por request en la CSP y sacar `'unsafe-inline'`. Los headers dejarían de definirse solo en `firebase.json`; también se podrían aplicar desde el servidor.

## Extra: qué pasa con el próximo merge después de un rollback

Hice el rollback desde la consola de Firebase y producción volvió a la versión anterior (`bcd3cfb`), pero `main` seguía teniendo el código nuevo. Lo comprobé en la Parte B: al mergear el PR de headers, el deploy a producción publicó el estado completo de `main`, y volvió también el título que había revertido con el rollback. El rollback de Firebase es una medida de emergencia temporal; no cambia el repositorio. En un equipo real, después del rollback se hace `git revert` del commit problemático (por PR) para que `main` coincida con lo que está en producción, y se corrige el problema en una rama aparte antes de volver a liberar.
