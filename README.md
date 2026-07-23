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
| `direccion`           | object  | Dirección desglosada: `estado`, `ciudad`, `cp`, `colonia`, `calle`, `numero` (todas string). La llenan **ambos** formularios; si algún campo no se captura llega como `""`. |
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

## Estado de cuenta (`edo-cuenta.html`)

`edo-cuenta.html` es una página autocontenida que **muestra un estado de cuenta
de crédito** (datos del cliente, del crédito, saldos, tabla de pagos,
comisiones y aclaraciones). A diferencia de los formularios, esta página **no
envía datos: los recibe y los despliega**.

### Cómo funciona

Al cargar, la página busca los datos de **dos** formas (la que ocurra primero
gana, y `window.setEstadoCuenta` puede sobrescribir en cualquier momento):

1. **Fetch de un archivo JSON.** Hace `fetch("./data.json")` (misma carpeta) y
   renderiza el resultado. La ruta es configurable con el parámetro
   `fuenteDatos`. Si falla, muestra un aviso: *"No se pudo cargar … "*.
2. **Inyección directa por JavaScript.** Expone una función global:

   ```js
   window.setEstadoCuenta(objetoJSON);   // pinta el estado de cuenta al instante
   ```

   Útil si sirves la página dentro de otro sistema (por ejemplo un portal) y
   ya tienes el JSON en memoria, o para evitar el fetch.

> **Importante (CORS / same-origin):** el `fetch` de `./data.json` funciona
> cuando la página se sirve por **http(s)** con el JSON en el mismo origen.
> Abriéndola con `file://` el navegador suele bloquear el fetch — en ese caso
> usa `window.setEstadoCuenta(...)`.

### Parámetros de configuración

| Parámetro           | Tipo    | Default       | Qué hace                                              |
| ------------------- | ------- | ------------- | ---------------------------------------------------- |
| `fuenteDatos`       | string  | `"data.json"` | Ruta/URL del JSON a cargar.                          |
| `mostrarCalendario` | boolean | `true`        | Muestra/oculta el calendario de pagos.              |
| `mostrarGrafica`    | boolean | `true`        | Muestra/oculta la gráfica.                          |

### Estructura del JSON (lista para desplegar)

El JSON debe traer **la información ya calculada y con el formato final** que
se va a mostrar (esta página solo presenta; no hace cálculos de negocio).
Estructura completa:

```json
{
  "cliente": {
    "nombre": "María López García",
    "noCliente": "CL-000123",
    "noCredito": "CR-778812",
    "rfc": "LOGM900514AB1",
    "telefono": "8112345678",
    "direccion": "Av. Insurgentes Sur 1234, Col. Del Valle, CDMX, 03100"
  },
  "credito": {
    "monto": 77000,
    "plazoMeses": 24,
    "tasaOrdinariaAnual": 24.9,
    "tasaMoratoriaAnual": 37.35,
    "cat": 32.9,
    "tipoInteres": "Fijo",
    "tipoMoneda": "MXN",
    "tipoPago": "Quincenal",
    "fechaApertura": "05/Mar/2026"
  },
  "saldos": {
    "saldoAnterior": 77000,
    "anticipoCapital": 0,
    "saldoFinal": 72500.50
  },
  "documento": {
    "producto": "Crédito personal",
    "periodoInicio": "01/Abr/2026",
    "periodoFin": "30/Abr/2026",
    "diasPeriodo": 30,
    "tipoEnvio": "Correo electrónico"
  },
  "pagos": [
    {
      "numero": 1,
      "fechaLimite": "15/Abr/2026",
      "fechaPago": "14/Abr/2026",
      "monto": 3625.00,
      "capital": 2900.00,
      "interesOrdinario": 625.00,
      "interesesMoratorios": 0,
      "comision": 0,
      "iva": 100.00,
      "ivaComision": 0,
      "ivaMoratorios": 0,
      "pagado": true,
      "tipoPago": "Quincenal"
    }
  ],
  "comisiones": [
    { "concepto": "Comisión por apertura", "monto": 1925.00, "iva": 308.00, "moneda": "MXN" }
  ],
  "aclaraciones": [
    {
      "folio": "ACL-0091",
      "fecha": "10/Abr/2026",
      "montoReclamado": 500.00,
      "montoDevuelto": 500.00,
      "estatus": "Resuelta",
      "fechaResolucion": "18/Abr/2026"
    }
  ]
}
```

### Referencia de campos

| Bloque         | Campo                 | Tipo    | Notas                                                        |
| -------------- | --------------------- | ------- | ------------------------------------------------------------ |
| `cliente`      | `nombre`              | string  | Nombre del titular.                                         |
|                | `noCliente`           | string  | Número de cliente.                                         |
|                | `noCredito`           | string  | Número de crédito.                                         |
|                | `rfc`                 | string  |                                                              |
|                | `telefono`            | string  |                                                              |
|                | `direccion`          | string  | Domicilio en una línea.                                    |
| `credito`      | `monto`               | number  | Monto del crédito.                                         |
|                | `plazoMeses`          | number  | Plazo en meses.                                            |
|                | `tasaOrdinariaAnual`  | number  | % anual (p. ej. `24.9`).                                   |
|                | `tasaMoratoriaAnual`  | number  | % anual.                                                   |
|                | `cat`                 | number  | CAT % (p. ej. `32.9`).                                     |
|                | `tipoInteres`         | string  | P. ej. `"Fijo"`.                                           |
|                | `tipoMoneda`          | string  | P. ej. `"MXN"`.                                            |
|                | `tipoPago`            | string  | P. ej. `"Quincenal"`.                                      |
|                | `fechaApertura`       | string  | Fecha (ver formato abajo).                                 |
| `saldos`       | `saldoAnterior`       | number  |                                                              |
|                | `anticipoCapital`     | number  |                                                              |
|                | `saldoFinal`          | number  | Saldo al cierre del periodo.                               |
| `documento`    | `producto`            | string  |                                                              |
|                | `periodoInicio`       | string  | Fecha inicio del periodo.                                  |
|                | `periodoFin`          | string  | Fecha fin del periodo.                                     |
|                | `diasPeriodo`         | number  | Días del periodo.                                          |
|                | `tipoEnvio`           | string  | P. ej. `"Correo electrónico"`.                            |
| `pagos[]`      | `numero`              | number  | Número de pago/amortización.                              |
|                | `fechaLimite`         | string  | Fecha límite de pago.                                      |
|                | `fechaPago`           | string  | Fecha en que se pagó (vacío si no).                       |
|                | `monto`               | number  | Monto total del pago.                                      |
|                | `capital`             | number  | Parte a capital.                                           |
|                | `interesOrdinario`    | number  |                                                              |
|                | `interesesMoratorios` | number  |                                                              |
|                | `comision`            | number  |                                                              |
|                | `iva`                 | number  | IVA del interés.                                           |
|                | `ivaComision`         | number  |                                                              |
|                | `ivaMoratorios`       | number  |                                                              |
|                | `pagado`              | boolean | `true` = realizado, `false` = pendiente.                   |
|                | `tipoPago`            | string  |                                                              |
| `comisiones[]` | `concepto`            | string  |                                                              |
|                | `monto`               | number  |                                                              |
|                | `iva`                 | number  |                                                              |
|                | `moneda`              | string  |                                                              |
| `aclaraciones[]`| `folio`              | string  |                                                              |
|                | `fecha`               | string  | Fecha de la aclaración.                                    |
|                | `montoReclamado`      | number  |                                                              |
|                | `montoDevuelto`       | number  |                                                              |
|                | `estatus`             | string  | P. ej. `"Resuelta"`, `"En proceso"`.                      |
|                | `fechaResolucion`     | string  |                                                              |

**Notas de formato:**

- **Montos:** números (no strings). La página los formatea a moneda es-MX.
- **Fechas:** strings en formato **`DD/Mmm/AAAA`** con mes abreviado en español
  (`Ene, Feb, Mar, Abr, May, Jun, Jul, Ago, Sep, Oct, Nov, Dic`) — p. ej.
  `05/Mar/2026`. También acepta mes numérico (`05/03/2026`). El calendario usa
  estas fechas, así que respétalas.
- **Arreglos** (`pagos`, `comisiones`, `aclaraciones`): pueden ir vacíos (`[]`)
  y la sección correspondiente se muestra sin filas.
- Todos los bloques son opcionales a nivel de robustez (si falta uno, se toma
  como vacío), pero para un estado de cuenta completo envía todos.

---

## Notas sobre los archivos

`index.html`, `form.html` y `edo-cuenta.html` son páginas autocontenidas
exportadas desde artifacts de diseño (apps de componentes reactivos). En los
formularios la lógica de envío vive dentro del script del componente
(`submitSol` en `index.html`, `goNext` en `form.html`) y no en un `<form>` HTML
nativo; la validación nativa HTML5 se sustituyó por validación equivalente en
JavaScript. `edo-cuenta.html` es de solo lectura: carga su JSON con `fetch`
(o `window.setEstadoCuenta(...)`) y lo despliega — ver la sección
[Estado de cuenta](#estado-de-cuenta-edo-cuentahtml).
