# Tanda Miranda — Landing y formulario de solicitud

Sitio estático de una sola página (`index.html`). Incluye un formulario de
solicitud de crédito multi-paso que **envía los datos por `fetch` (POST JSON)**
a un endpoint/API externo que tú desarrollas y hospedas por separado.

Esta guía explica cómo conectar el formulario a tu API.

---

## 1. Configurar el endpoint (obligatorio)

Dentro de `index.html`, en el script del componente, hay una constante al
inicio. Cámbiala por la URL de tu API:

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

## 2. Formato del payload (lo que recibe tu API)

El cuerpo es un JSON. Todos los campos de texto llegan ya recortados
(`trim`). Ejemplo completo:

```json
{
  "nombre": "María López García",
  "telefono": "5512345678",
  "correo": "maria@ejemplo.com",
  "empresa": "Comercializadora del Norte",
  "producto": "Descuento por Nómina",
  "monto": "25000",
  "apellidoPaterno": "López",
  "apellidoMaterno": "García",
  "entidadNacimiento": "Ciudad de México",
  "fechaNacimiento": "1990-05-14",
  "escolaridad": "Licenciatura",
  "dependientes": "2",
  "ingresos": "18000",
  "otrosIngresos": "Renta, $2,000.00 mensuales",
  "gastosFijos": "6000",
  "domicilio": "Calle Falsa 123, Col. Centro",
  "antiguedadDomicilio": "5 años",
  "plazo": "12",
  "uso": "Consolidación de deudas",
  "aceptaPrivacidad": true,
  "captchaToken": "0.abc123...",
  "origen": "landing-tanda-miranda",
  "enviadoEn": "2026-07-23T18:20:00.000Z"
}
```

### Referencia de campos

| Campo                 | Tipo    | Notas                                                        |
| --------------------- | ------- | ------------------------------------------------------------ |
| `nombre`              | string  | **Requerido**. Nombre completo.                              |
| `telefono`            | string  | **Requerido**. Validado en frontend a 10 dígitos.           |
| `correo`              | string  | Opcional; si se envía, se valida el formato de email.       |
| `empresa`             | string  | Empresa / fuente de ingresos.                                |
| `producto`            | string  | Producto de crédito seleccionado.                            |
| `monto`               | string  | Monto solicitado (texto, sin formato garantizado).          |
| `apellidoPaterno`     | string  |                                                              |
| `apellidoMaterno`     | string  |                                                              |
| `entidadNacimiento`   | string  |                                                              |
| `fechaNacimiento`     | string  |                                                              |
| `escolaridad`         | string  |                                                              |
| `dependientes`        | string  |                                                              |
| `ingresos`            | string  |                                                              |
| `otrosIngresos`       | string  |                                                              |
| `gastosFijos`         | string  |                                                              |
| `domicilio`           | string  |                                                              |
| `antiguedadDomicilio` | string  |                                                              |
| `plazo`               | string  | Plazo del préstamo.                                          |
| `uso`                 | string  | Uso del recurso.                                             |
| `aceptaPrivacidad`    | boolean | `true` si aceptó el Aviso de Privacidad.                     |
| `captchaToken`        | string  | Token de Cloudflare Turnstile (vacío si Turnstile desactivado). Ver §4. |
| `origen`              | string  | Constante `"landing-tanda-miranda"`.                        |
| `enviadoEn`           | string  | Timestamp ISO-8601 generado en el navegador.                |

> Los campos numéricos viajan como **string** (tal cual los captura el
> usuario). Normalízalos en el servidor si necesitas números.

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

En el script al final del `<body>` de `index.html`:

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

## Notas sobre el archivo

`index.html` es una página autocontenida exportada desde un artifact de diseño;
el formulario es una app de componentes reactivos, por eso la lógica de envío
(`submitSol`) vive dentro del script del componente y no en un `<form>` HTML
nativo. La validación nativa HTML5 se sustituyó por validación equivalente en
JavaScript.
