# Tanda Miranda — Landing y formulario de solicitud

Sitio estático que incluye **dos formularios** de solicitud de crédito. Ambos
**envían los datos por `fetch` (POST JSON)** a un endpoint/API externo que tú
desarrollas y hospedas por separado, y ambos traen cableado Cloudflare
Turnstile (desactivado por defecto).

| Archivo      | Qué es                                                    | `origen` en el payload         |
| ------------ | --------------------------------------------------------- | ------------------------------ |
| `index.html` | Landing completa con el formulario de solicitud integrado | `"landing-tanda-miranda"`      |
| `form.html`  | Formulario de préstamo en 3 pasos, página independiente   | `"form-html-prestamo-pasos"`   |

Esta guía explica cómo conectarlos a tu API. **Ambos envían exactamente el
mismo formato de payload** (mismas claves, mismo orden, mismos tipos); solo se
distinguen por los campos `origen` y `paginaUrl` (§2). Así tu API procesa las
dos fuentes con un único parser.

---

## 1. Configurar el endpoint (obligatorio)

En **cada** archivo (`index.html` y `form.html`), en el script del componente,
hay una constante al inicio. Cámbiala por la URL de tu API (puede ser la misma
para ambos o una distinta):

```js
const WEBHOOK_URL = "https://tu-api.com/endpoint";   // ← tu endpoint real
```

> Búscala con `Ctrl+F` → `WEBHOOK_URL`.

El formulario hace:

```
POST <WEBHOOK_URL>
Content-Type: application/json
Accept: application/json
```

No hay recargas ni redirecciones: todo ocurre de forma asíncrona en el
frontend.

---

## 2. Formato del payload (unificado)

`index.html` y `form.html` envían **el mismo JSON**: mismas claves, mismo orden
y mismos tipos. Todos los campos de valor viajan como **string** ya recortado
(`trim`); los que un formulario no captura llegan como `""`. Así tu API usa un
solo parser para ambas fuentes.

```json
{
  "origen": "landing-tanda-miranda",
  "paginaUrl": "https://tu-dominio-del-sitio.com/index.html",
  "enviadoEn": "2026-07-23T18:20:00.000Z",
  "nombre": "María",
  "apellidoPaterno": "López",
  "apellidoMaterno": "García",
  "telefono": "5512345678",
  "correo": "maria@ejemplo.com",
  "entidadNacimiento": "Ciudad de México",
  "fechaNacimiento": "1990-05-14",
  "escolaridad": "Licenciatura",
  "dependientes": "2",
  "ingresos": "18000",
  "gastos": "6000",
  "otrosIngresos": "Renta, $2,000.00 mensuales",
  "empresa": "Comercializadora del Norte",
  "producto": "Descuento por Nómina",
  "monto": "25000",
  "plazo": "12",
  "uso": "Consolidación de deudas",
  "domicilio": "Calle Falsa 123, Col. Centro",
  "antiguedadDomicilio": "5 años",
  "direccion": {
    "estado": "CDMX",
    "ciudad": "Benito Juárez",
    "cp": "03100",
    "colonia": "Del Valle",
    "calle": "Av. Insurgentes Sur",
    "numero": "1234"
  },
  "avisoPrivacidad": true,
  "captchaToken": "0.abc123..."
}
```

### Identificador de origen

Cada envío indica de dónde viene con dos campos:

| Campo       | Tipo   | Notas                                                                                 |
| ----------- | ------ | ------------------------------------------------------------------------------------- |
| `origen`    | string | Slug fijo de la fuente: `"landing-tanda-miranda"` (index) o `"form-html-prestamo-pasos"` (form). |
| `paginaUrl` | string | URL real de la página desde la que se envió (`window.location.href`).                 |

### Referencia de campos

| Campo                 | Tipo    | Notas                                                                 |
| --------------------- | ------- | --------------------------------------------------------------------- |
| `origen`              | string  | Ver arriba.                                                           |
| `paginaUrl`           | string  | Ver arriba.                                                           |
| `enviadoEn`           | string  | Timestamp ISO-8601 generado en el navegador.                          |
| `nombre`              | string  | **Requerido**. Nombre completo (o nombre de pila en `form.html`).     |
| `apellidoPaterno`     | string  |                                                                       |
| `apellidoMaterno`     | string  |                                                                       |
| `telefono`            | string  | **Requerido**. Validado en frontend a 10 dígitos.                     |
| `correo`              | string  | Opcional; si se envía, se valida el formato de email.                 |
| `entidadNacimiento`   | string  |                                                                       |
| `fechaNacimiento`     | string  | `AAAA-MM-DD`.                                                         |
| `escolaridad`         | string  |                                                                       |
| `dependientes`        | string  |                                                                       |
| `ingresos`            | string  |                                                                       |
| `gastos`              | string  | Gastos fijos mensuales.                                               |
| `otrosIngresos`       | string  |                                                                       |
| `empresa`             | string  | Solo `index.html` lo captura; en `form.html` llega como `""`.         |
| `producto`            | string  | Solo `index.html` lo captura; en `form.html` llega como `""`.         |
| `monto`               | string  | Monto solicitado.                                                    |
| `plazo`               | string  | Plazo del préstamo.                                                  |
| `uso`                 | string  | Uso del recurso.                                                      |
| `domicilio`           | string  | Domicilio en texto libre.                                            |
| `antiguedadDomicilio` | string  | P. ej. `"5 años"`.                                                   |
| `direccion`           | object  | Dirección desglosada: `estado`, `ciudad`, `cp`, `colonia`, `calle`, `numero` (todas string). Solo `form.html` la llena; en `index.html` sus campos llegan como `""`. |
| `avisoPrivacidad`     | boolean | `true` si aceptó el Aviso de Privacidad.                              |
| `captchaToken`        | string  | Token de Cloudflare Turnstile (vacío si Turnstile desactivado). Ver §4. |

> Todos los valores llegan como **string** (o `""` si están vacíos), incluidos
> los numéricos. Conviértelos/valídalos en el servidor si necesitas números.

### Folio en la respuesta (opcional)

Si tu API responde con `{ "folio": "..." }`, `form.html` muestra ese folio en su
pantalla de confirmación. Si no devuelves folio, el frontend genera uno local
(`TM-AÑO-XXXXXX`) solo para la UI. (`index.html` no usa folio.)

---

## 3. Respuesta esperada y manejo de errores

- **Éxito:** responde con un status **2xx** (p. ej. `200` o `201`). El
  frontend limpia el formulario y muestra el mensaje de "¡Solicitud
  enviada!".
- **Error:** cualquier status **no-2xx**, o un fallo de red, muestra un
  mensaje de error elegante en la UI. El frontend no interpreta el cuerpo de
  la respuesta de error, así que basta con devolver el status adecuado.

El cuerpo de la respuesta no es obligatorio; puedes devolver JSON si quieres,
pero el frontend solo evalúa `response.ok`.

### CORS (importante)

El `fetch` es **cross-origin** (el sitio y tu API están en dominios
distintos). Tu endpoint debe responder con las cabeceras CORS adecuadas, e
implementar el preflight `OPTIONS`:

```
Access-Control-Allow-Origin: https://tu-dominio-del-sitio.com
Access-Control-Allow-Methods: POST, OPTIONS
Access-Control-Allow-Headers: Content-Type, Accept
```

---

## 4. Cloudflare Turnstile (anti-spam)

Como es un endpoint público en un sitio estático, se dejó cableado Cloudflare
Turnstile. **Está desactivado por defecto** y se activa poniendo tu Site Key.

### Activarlo

En el script al final del `<body>` de **cada archivo** (`index.html` y
`form.html`):

```js
window.__TURNSTILE_SITE_KEY = "";   // ← pega tu Site Key de Turnstile
```

- **Vacío** → Turnstile desactivado; el formulario funciona sin verificación.
- **Con Site Key** → se carga el widget oficial de Cloudflare, y el formulario
  **exige resolver el reto** antes de enviar. El token resultante viaja en
  `payload.captchaToken`.

### Validación en el servidor (obligatoria)

El token del frontend **no protege nada por sí solo**: debes verificarlo en tu
API contra el `siteverify` de Cloudflare usando tu **Secret Key**.

```js
// Ejemplo (Node / Cloudflare Worker)
async function verificarTurnstile(token, ip) {
  const r = await fetch(
    "https://challenges.cloudflare.com/turnstile/v0/siteverify",
    {
      method: "POST",
      headers: { "Content-Type": "application/x-www-form-urlencoded" },
      body: new URLSearchParams({
        secret: TURNSTILE_SECRET_KEY, // ← tu Secret Key (variable de entorno)
        response: token,
        remoteip: ip,
      }),
    }
  );
  const data = await r.json();
  return data.success === true;
}
```

Rechaza la solicitud si `verificarTurnstile()` devuelve `false`.

---

## 5. Ejemplo mínimo de endpoint

```js
// Cloudflare Worker de ejemplo
export default {
  async fetch(request, env) {
    const cors = {
      "Access-Control-Allow-Origin": "https://tu-dominio-del-sitio.com",
      "Access-Control-Allow-Methods": "POST, OPTIONS",
      "Access-Control-Allow-Headers": "Content-Type, Accept",
    };
    if (request.method === "OPTIONS") return new Response(null, { headers: cors });
    if (request.method !== "POST") return new Response("Method Not Allowed", { status: 405, headers: cors });

    const data = await request.json();

    // 1) Validaciones mínimas
    if (!data.nombre || !data.telefono) {
      return new Response("Faltan campos", { status: 400, headers: cors });
    }

    // 2) Verifica Turnstile (si lo activaste)
    if (env.TURNSTILE_SECRET_KEY) {
      const ok = await verificarTurnstile(data.captchaToken, request.headers.get("CF-Connecting-IP"));
      if (!ok) return new Response("Verificación fallida", { status: 403, headers: cors });
    }

    // 3) Procesa la solicitud (guarda en DB, notifica, etc.)
    // await guardarSolicitud(data);

    return new Response(JSON.stringify({ ok: true }), {
      status: 201,
      headers: { ...cors, "Content-Type": "application/json" },
    });
  },
};
```

---

## 6. Checklist de puesta en marcha

- [ ] Cambiar `WEBHOOK_URL` por la URL real de tu API.
- [ ] Configurar CORS en el endpoint para el dominio del sitio.
- [ ] Devolver `2xx` en éxito y un status de error en fallo.
- [ ] (Opcional) Poner tu Site Key en `window.__TURNSTILE_SITE_KEY`.
- [ ] (Si usas Turnstile) Validar `captchaToken` con `siteverify` + Secret Key
      en el servidor.
- [ ] Normalizar/validar de nuevo los datos en el servidor (nunca confíes solo
      en el frontend).

---

## Notas sobre los archivos

`index.html` y `form.html` son páginas autocontenidas exportadas desde
artifacts de diseño; ambos formularios son apps de componentes reactivos, por
eso la lógica de envío vive dentro del script del componente (`submitSol` en
`index.html`, `goNext` en `form.html`) y no en un `<form>` HTML nativo. La
validación nativa HTML5 se sustituyó por validación equivalente en JavaScript.
