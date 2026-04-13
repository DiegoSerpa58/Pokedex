# Despliegue de Pokédex en Azure Static Web Apps

Este documento describe, paso a paso, cómo se desplegó la aplicación **Pokédex Angular** en Azure Static Web Apps, cómo se configuraron los encabezados de seguridad y cómo se resolvieron los errores encontrados (CSP, 500 y 404).

---

## 1. Preparación del entorno local

### 1.1. Clonado del repositorio

```bash
git clone https://github.com/DiegoSerpa58/Pokedex.git
cd Pokedex
```

### 1.2. Ubicación del proyecto Angular

El proyecto de la Pokédex se encuentra en:

```text
sistemas-distribuidos/poke-dex-lab/source/pokedex-angular
```

### 1.3. Instalación y ejecución local

```bash
cd sistemas-distribuidos/poke-dex-lab/source/pokedex-angular

npm install

npm start
# o
ng serve --open
```

Resultado esperado:

- La aplicación se muestra correctamente en `http://localhost:4200/`.

### 1.4. Build de producción

```bash
ng build --configuration production
```

Genera la carpeta:

```text
dist/pokedex-angular
```

---

## 2. Creación de Azure Static Web App

Desde el **Azure Portal**:

1. Crear recurso de tipo **Static Web App**.
2. Elegir la suscripción **Azure for Students**.
3. Asignar un grupo de recursos, por ejemplo: `rg-pokedex`.
4. Nombre de la app: `pokedex-diego` (ejemplo).
5. Origen de código: **GitHub**.
6. Seleccionar:
   - Cuenta: `DiegoSerpa58`
   - Repositorio: `Pokedex`
   - Rama: `master`
7. Configuración de la app:
   - **App location:**  
     `sistemas-distribuidos/poke-dex-lab/source/pokedex-angular`
   - **Api location:** *(vacío, no se usa API Functions)*
   - **Output location:**  
     `dist/pokedex-angular`

Al crear el recurso, Azure:

- Generó automáticamente un workflow en `.github/workflows/...yml`.
- Configuró el despliegue continuo (cada commit en `master` dispara un build y publicación).

---

## 3. Encabezados de seguridad (Static Web Apps config)

### 3.1. Creación de `staticwebapp.config.json`

En la raíz del proyecto Angular:

```text
sistemas-distribuidos/poke-dex-lab/source/pokedex-angular/staticwebapp.config.json
```

Contenido final:

```json
{
  "globalHeaders": {
    "Content-Security-Policy": "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; style-src-elem 'self' 'unsafe-inline' https://fonts.googleapis.com; font-src 'self' https://fonts.gstatic.com data:; img-src 'self' data: https://raw.githubusercontent.com https://pokeapi.co https://assets.pokemon.com https://beta.pokeapi.co; connect-src 'self' https://pokeapi.co https://beta.pokeapi.co https://beta.pokeapi.co/graphql/v1beta; object-src 'none'; base-uri 'self'; form-action 'self'; frame-ancestors 'none'; upgrade-insecure-requests",
    "Strict-Transport-Security": "max-age=31536000; includeSubDomains; preload",
    "X-Content-Type-Options": "nosniff",
    "X-Frame-Options": "DENY",
    "Referrer-Policy": "no-referrer",
    "Permissions-Policy": "camera=(), microphone=(), geolocation=(), payment=()",
    "X-XSS-Protection": "1; mode=block"
  },
  "navigationFallback": {
    "rewrite": "/index.html",
    "exclude": ["/images/*.{png,jpg,gif,svg}", "/css/*", "/js/*"]
  }
}
```

Este archivo se versiona en GitHub y es leído automáticamente por Azure Static Web Apps en cada despliegue.

### 3.2. Propósito de cada header

- **Content-Security-Policy**  
  Protege contra XSS y controla la carga de recursos.

  - `default-src 'self'`: solo el mismo origen por defecto.
  - `script-src 'self' 'unsafe-inline'`: permite scripts del mismo origen y scripts inline que la aplicación original ya utiliza (evitando errores CSP en consola).
  - `style-src` / `style-src-elem`: permiten estilos inline y estilos desde `https://fonts.googleapis.com`.
  - `font-src`: `self`, `https://fonts.gstatic.com` y `data:`.
  - `img-src`: `self`, `data:`, `https://raw.githubusercontent.com`, `https://pokeapi.co`, `https://assets.pokemon.com`, `https://beta.pokeapi.co`.
  - `connect-src`: solo `self`, `https://pokeapi.co`, `https://beta.pokeapi.co` y el endpoint GraphQL.
  - `object-src 'none'`: bloquea objetos tipo `<object>`, `<embed>`, `<applet>`.
  - `base-uri 'self'`: evita cambiar la base de la URL con `<base>`.
  - `form-action 'self'`: solo permite enviar formularios al mismo origen.
  - `frame-ancestors 'none'`: impide que la app se embelezca en iframes (clickjacking).
  - `upgrade-insecure-requests`: fuerza a pasar peticiones HTTP a HTTPS cuando sea posible.

- **Strict-Transport-Security**  
  Fuerza el uso de HTTPS con `max-age=31536000` (1 año), `includeSubDomains` y `preload`.

- **X-Content-Type-Options: nosniff**  
  Evita que el navegador interprete archivos con un tipo de contenido diferente al declarado.

- **X-Frame-Options: DENY**  
  Impide que otros sitios inserten la app en un iframe.

- **Referrer-Policy: no-referrer**  
  Elimina el envío del encabezado `Referer`.

- **Permissions-Policy**  
  Deshabilita APIs que no se usan: geolocalización, micrófono, cámara y pago.

- **X-XSS-Protection**  
  Activa el filtro XSS heredado en navegadores que todavía lo soportan.

---

## 4. Validación con SecurityHeaders.com

Se utilizó la herramienta:

- `https://securityheaders.com/`

### 4.1. Resultado principal

- **Calificación estable en producción:** **A**
- Encabezados presentes y en verde:
  - `Content-Security-Policy`
  - `Strict-Transport-Security`
  - `X-Content-Type-Options`
  - `X-Frame-Options`
  - `Referrer-Policy`
  - `Permissions-Policy`
  - `X-XSS-Protection`

### 4.2. Ajustes de CSP y relación con la consola

Durante las pruebas se endureció la `Content-Security-Policy` eliminando `'unsafe-inline'` de `script-src` (dejando `script-src 'self'`).  
Con esa política estricta se alcanzó una calificación **A+** en securityheaders.com, pero aparecieron errores CSP en la consola del navegador:

> Executing inline script violates the following Content Security Policy directive 'script-src 'self''...

Esto se debe a que la aplicación original utiliza scripts y manejadores de eventos inline, que quedan bloqueados cuando se quita `'unsafe-inline'`.

Para cumplir con la rúbrica del laboratorio, que exige que **no haya errores en la consola**, se decidió relajar ligeramente la directiva de scripts a:

```text
script-src 'self' 'unsafe-inline';
```

manteniendo el resto de la política restrictiva (restricciones en imágenes, conexiones, `object-src 'none'`, `frame-ancestors 'none'`, etc.).  
Con esta configuración:

- La aplicación funciona correctamente en Azure.
- La consola del navegador queda limpia, sin errores CSP.
- La calificación de securityheaders.com se mantiene en un nivel alto (**A**), y además se obtuvo **A+** en la auditoría independiente de SSL Labs para la configuración TLS del servidor.

---

## 5. Errores detectados y solución

### 5.1. Errores CSP (bloqueo de fuentes y API GraphQL)

En la consola aparecían mensajes como:

- Bloqueo de `https://fonts.googleapis.com/...` y `https://fonts.gstatic.com/...`
- Bloqueo de `https://beta.pokeapi.co/graphql/v1beta` por `connect-src`.

**Solución:**

- Modificar la cabecera CSP en `staticwebapp.config.json` para incluir:
  - `style-src 'self' 'unsafe-inline' https://fonts.googleapis.com`
  - `style-src-elem 'self' 'unsafe-inline' https://fonts.googleapis.com`
  - `font-src 'self' https://fonts.gstatic.com data:`
  - `connect-src 'self' https://pokeapi.co https://beta.pokeapi.co https://beta.pokeapi.co/graphql/v1beta`

Después de estos cambios la app dejó de mostrar el error 500 y la Pokédex cargó correctamente.

### 5.2. Errores 404 de sprites (`pokemon-green.png`) solo en producción

En Azure, la consola mostraba:

```text
GET https://.../pokedex-angular/assets/images/pokemon-green.png 404 (Not Found)
```

En el código:

```ts
// sistemas-distribuidos/poke-dex-lab/source/pokedex-angular/src/environments/environment.prod.ts
imagesPath: '/pokedex-angular/assets/images',
```

Mientras que en desarrollo:

```ts
// environment.ts
imagesPath: '/assets/images',
```

Esto hacía que en producción se generaran rutas incorrectas:

- `/pokedex-angular/assets/images/pokemon-green.png` (404)

**Solución:**

Unificar la configuración en `environment.prod.ts`:

```ts
export const environment = {
  production: true,
  pokeApi: 'https://pokeapi.co/api/v2',
  pokeApiGraphQL: 'https://beta.pokeapi.co/graphql/v1beta',
  homeAngular: 'https://angular.io/',
  homePokeApi: 'https://pokeapi.co/',
  keilerLinkedin: 'https://www.linkedin.com/in/keilermora/',
  pokedexGithub: 'https://github.com/keilermora/pokedex-angular',
  imagesPath: '/assets/images',
  language: 'en',
  languageId: 9,
};
```

Tras este ajuste:

- Los sprites de la Pokédex cargan correctamente.
- La consola queda sin errores 404 de imágenes.

---

## 6. Desafío Maestro – Auditoría SSL con SSL Labs

Como parte del desafío maestro se realizó una auditoría adicional de seguridad usando **Qualys SSL Labs** sobre la URL pública de la aplicación:

- URL analizada: `https://ambitious-coast-0bf395010.6.azurestaticapps.net`
- Herramienta: [SSL Labs – SSL Server Test](https://www.ssllabs.com/ssltest/)

### 6.1. Resultados

- **Calificación general:** **A+**.
- Certificado TLS válido emitido para `*.azurestaticapps.net`.
- Soporte de **TLS 1.3** y suites criptográficas modernas.
- `HTTP Strict Transport Security (HSTS)` activo con duración larga.
- Política CAA configurada correctamente para el dominio.

No se detectaron vulnerabilidades críticas en la configuración SSL/TLS del servidor.  
Esto refuerza la seguridad del canal de comunicación, complementando los encabezados HTTP configurados en `staticwebapp.config.json`.

---

## 7. Verificación final

Después de todos los cambios:

1. **Workflow de GitHub Actions:** en verde (build y deploy correctos).
2. **Aplicación en Azure:**
   - URL accesible: `https://ambitious-coast-0bf395010.6.azurestaticapps.net`
   - Listado de 151 Pokémon funcionando.
   - Navegación entre detalles sin errores.
3. **Consola del navegador:**
   - Sin errores JavaScript.
   - Sin errores de red (404/500).
   - Sin errores CSP con la configuración final (`script-src 'self' 'unsafe-inline'; ...`).
4. **SecurityHeaders.com:**
   - Calificación **A**, encabezados de seguridad correctamente configurados.
5. **SSL Labs:**
   - Calificación **A+** en la configuración SSL/TLS del dominio de Azure.

Con esto se cumplen todos los requisitos del laboratorio:

- Despliegue funcional en la nube.
- Seguridad mediante encabezados HTTP.
- Ausencia de errores en la consola.
- Auditoría adicional de seguridad (Desafío Maestro).
- Documentación clara del proceso y de los problemas encontrados y solucionados.
