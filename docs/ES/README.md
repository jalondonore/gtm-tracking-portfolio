# Portafolio GTM — Documentación Completa (Español)

> **Contexto:** Todos los proyectos fueron implementados para grupos de clínicas dentales con múltiples sedes. El desafío central en cada caso fue capturar eventos de envío de formularios e interacciones del usuario en páginas y widgets que viven fuera del dominio propio del cliente, o que son renderizados dinámicamente por frameworks de JavaScript de terceros, lo cual hace que los triggers estándar de GTM sean poco confiables o simplemente no disponibles.

---

## Proyecto 01 — Seguimiento de Formularios Multi-Sede con Nombres de Eventos Dinámicos

### Contexto de Negocio

Un grupo dental con múltiples sedes clínicas utilizaba un único sitio web con un formulario de citas compartido y un formulario de contacto. Ambos formularios incluían un **selector desplegable de ubicación**, permitiendo a los pacientes elegir su clínica preferida antes de enviar el formulario. El objetivo era identificar desde qué sede provenía cada envío, para poder segmentar los reportes de Google Ads y GA4 por clínica sin necesidad de crear propiedades o vistas separadas.

### Desafío Técnico

Los formularios fueron construidos en Webflow y se renderizan dentro de un **modal que carga de forma asíncrona** — es decir, el elemento del formulario no existe en el DOM cuando GTM hace su carga inicial. Los triggers estándar de Envío de Formulario y de Clic de GTM no podían apuntar al formulario de forma confiable porque:

1. El elemento del formulario (`#wf-form-Appointment-Form`) se inyecta en el DOM solo cuando el usuario abre el modal
2. El clic en el botón de envío se dispara antes de que los datos del formulario estén disponibles para leer
3. El valor del selector de ubicación necesitaba ser sanitizado antes de usarse en el nombre del evento

### Solución: Polling del DOM + Listener de Submit + Sanitización del Valor

Se desplegaron dos tags de HTML Personalizado en GTM — uno para el formulario de citas, otro para el formulario de contacto. Ambos siguen el mismo patrón.

---

#### Script: Rastreador del Formulario de Citas

```javascript
(function() {

  // PASO 1 — Definir la función de inicialización del formulario
  // Esta función solo se llama una vez que se confirma que el formulario
  // existe en el DOM
  function initFormTracking() {
    var form = document.getElementById('wf-form-Appointment-Form');
    if (!form) return;

    // PASO 2 — Adjuntar un listener de evento 'submit' al elemento del formulario
    // Se dispara cuando el usuario envía el formulario, antes de que la página navegue
    form.addEventListener('submit', function(e) {

      // PASO 3 — Leer la ubicación seleccionada desde el desplegable
      var locationSelect = document.getElementById('Location');
      if (!locationSelect) return;

      var rawCity = locationSelect.value || 'unknown';

      // PASO 4 — Sanitizar el valor crudo del desplegable
      // Eliminar comillas, colapsar espacios, quitar guiones, convertir a minúsculas
      // Resultado: "East Cobb" → "eastcobb", "Johns-Creek" → "johnscreek"
      var cleanCity = rawCity
        .replace(/['"]/g, '')    // eliminar comillas simples y dobles
        .replace(/\s+/g, '')     // eliminar todos los espacios
        .replace(/-/g, '')       // eliminar guiones
        .toLowerCase();          // normalizar a minúsculas

      // PASO 5 — Construir un nombre de evento dinámico usando la ubicación sanitizada
      // Patrón: form_submit_Appointment_{ubicación}
      // Ejemplo: form_submit_Appointment_eastcobb
      var eventName = 'form_submit_Appointment_' + cleanCity;

      // PASO 6 — Enviar el evento al dataLayer
      // Se envían dos variables: el valor limpio (para el nombre del evento) y el valor
      // crudo (para depuración o como referencia del valor original)
      window.dataLayer = window.dataLayer || [];
      window.dataLayer.push({
        event: eventName,
        dlv_form_appointment_Location: cleanCity,
        dlv_form_appointment_Location_raw: rawCity
      });
    });
  }

  // PASO 7 — Hacer polling al DOM cada 500ms hasta que aparezca el formulario
  // Esto maneja el caso donde el formulario está dentro de un modal o se carga
  // de forma lazy después del render inicial de la página
  var checkInterval = setInterval(function() {
    var form = document.getElementById('wf-form-Appointment-Form');
    if (form) {
      clearInterval(checkInterval); // detener el polling al encontrar el formulario
      initFormTracking();           // inicializar el seguimiento
    }
  }, 500);

})();
```

**Cómo se conecta con GTM:**
- Un trigger de Evento Personalizado en GTM escucha eventos que coincidan con `form_submit_Appointment_.*` (regex) o cada nombre de evento específico
- Un tag de Evento GA4 se dispara en ese trigger, enviando `dlv_form_appointment_Location` como parámetro del evento
- Los tags de conversión de Google Ads pueden estar limitados por sede usando triggers separados para cada variante del nombre del evento

---

#### Script: Rastreador del Formulario de Contacto

Patrón idéntico al rastreador del formulario de citas. Diferencias:
- Apunta a `#wf-form-Contact-Form` y al desplegable `#location-3`
- Patrón del nombre del evento: `form_submit_Contact_{ubicación}`

---

### Resumen de Configuración GTM

| Componente | Nombre | Propósito |
|-----------|------|---------|
| Tag HTML Personalizado | Appointment Form | Hace polling, adjunta listener, envía evento dinámico |
| Tag HTML Personalizado | Contact Form | Mismo patrón para el formulario de contacto |
| Trigger Evento Personalizado | Envío Formulario Citas | Escucha eventos `form_submit_Appointment_*` |
| Trigger Evento Personalizado | Envío Formulario Contacto | Escucha eventos `form_submit_Contact_*` |
| Variable DataLayer | `dlv_form_appointment_Location` | Captura el valor de ubicación limpio |
| Variable DataLayer | `dlv_form_appointment_Location_raw` | Captura el valor original del desplegable |
| Tag Evento GA4 | GA4 - Envío Formulario por Sede | Se dispara en el envío, envía la ubicación |

---

---

## Proyecto 02 — Seguimiento de Pasos en Formulario SPA de SmileVirtual (v1 — Flags Booleanos)

### Contexto de Negocio

SmileVirtual es una plataforma de terceros para consultas virtuales de odontología, renderizada como una **aplicación Next.js (React) independiente** alojada en el dominio del proveedor. Las clínicas dentales embeben o enlazan a este formulario como su flujo principal de consulta virtual. El formulario es de múltiples pasos (3 pasos), y cada paso renderiza contenido diferente dinámicamente sin navegación de página tradicional.

El objetivo era rastrear **hasta qué punto avanzaban los usuarios** en el formulario de 3 pasos, permitiendo análisis de embudo en GA4 y triggers de retargeting en Meta Pixel.

### Desafío Técnico

Dado que el formulario de SmileVirtual es una SPA en React:
- **No hay vistas de página** entre pasos — la URL no cambia
- Los triggers estándar de visibilidad o clic de GTM no pueden detectar los cambios de paso
- El DOM es manipulado por la reconciliación del DOM virtual de React — los elementos aparecen y desaparecen sin eventos de carga tradicionales
- El texto indicador de paso (`STEP 1`, `STEP 2`, `STEP 3`) se renderiza dentro de elementos profundamente anidados con nombres de clase CSS generados dinámicamente (clases hash de JSX)

### Solución: MutationObserver + Switch/Case con Flags Booleanos

```javascript
// PASO 1 — Declarar flags booleanos para rastrear qué pasos ya se han disparado
// Esto previene eventos duplicados si el observer se activa varias veces
// en el mismo paso (React puede re-renderizar sin cambiar realmente el paso)
var step1 = false;
var step2 = false;
var step3 = false;

// PASO 2 — Definir la función de detección de pasos
function detectSteps() {

  // PASO 3 — Definir múltiples selectores CSS para apuntar al elemento indicador de paso
  // Se proporcionan dos selectores porque la SPA renderiza diferentes layouts
  // dependiendo del dispositivo/breakpoint — ambos apuntan al elemento que
  // muestra el texto "STEP 1", "STEP 2" o "STEP 3"
  var selectors = [
    '#__next > div > main > form > div > div.sign-up-column > div.sign-up-content.my-auto > div.text-center > small',
    '#__next > div > main > form > div > div:nth-child(1) > div.sign-up-content > div > p'
  ];

  // PASO 4 — Probar cada selector hasta que uno retorne un elemento visible
  for (var i = 0; i < selectors.length; i++) {
    var el = document.querySelector(selectors[i]);
    if (el) {
      var text = el.innerText.trim();

      // PASO 5 — Evaluar qué paso es visible y enviar al dataLayer
      // Los flags booleanos aseguran que cada evento de paso solo se dispare una vez por sesión
      switch (text) {
        case 'STEP 1':
          if (step1 === false) {
            window.dataLayer = window.dataLayer || [];
            window.dataLayer.push({
              event: 'formStep',
              formStepText: 'step1'
            });
            step1 = true; // marcado como disparado, no se disparará de nuevo
          }
          break;
        case 'STEP 2':
          if (step2 === false) {
            window.dataLayer = window.dataLayer || [];
            window.dataLayer.push({
              event: 'formStep',
              formStepText: 'step2'
            });
            step2 = true;
          }
          break;
        case 'STEP 3':
          if (step3 === false) {
            window.dataLayer = window.dataLayer || [];
            window.dataLayer.push({
              event: 'formStep',
              formStepText: 'step3'
            });
            step3 = true;
          }
          break;
      }
    }
  }
}

// PASO 6 — Configurar un MutationObserver sobre document.body
// Observa CUALQUIER cambio en el DOM (adición de hijos, mutaciones en el subárbol)
// y llama a detectSteps() cada vez que React actualiza el DOM
var observer = new MutationObserver(function() {
  detectSteps();
});

// PASO 7 — Comenzar la observación después de un retardo de 1 segundo
// El retardo asegura que React haya terminado su render inicial antes
// de que el observer comience a vigilar los cambios
setTimeout(function() {
  observer.observe(document.body, {
    childList: true,  // observar nodos hijos añadidos/eliminados
    subtree: true     // observar todos los descendientes, no solo los hijos directos
  });
  detectSteps(); // también ejecutar inmediatamente por si ya estamos en un paso
}, 1000);
```

---

---

## Proyecto 03 — Seguimiento de Pasos en Formulario SPA de SmileVirtual (v2 — Selector Semántico + Captura de Título)

### Contexto de Negocio

Una segunda clínica dental implementó la misma integración de SmileVirtual pero requirió un script de seguimiento más robusto e informativo. El enfoque v1 tenía dos limitaciones en este contexto:

1. Los selectores profundamente anidados con hashes de JSX eran frágiles — una actualización de la plataforma podía romperlos
2. No se capturaba el título del paso del formulario, limitando la utilidad de los datos del embudo

### Solución: Selectores CSS Semánticos + Título del Paso + Deduplicación con `lastStep`

```javascript
(function() {

  // PASO 1 — Rastrear el último paso visto para evitar eventos duplicados
  // A diferencia de los flags booleanos de v1, este enfoque permite re-detectar
  // si un usuario navega hacia atrás y luego hacia adelante en el formulario
  var lastStep = null;

  function detectStep() {

    // PASO 2 — Usar un selector CSS semántico e independiente del framework
    // En lugar de una ruta completa con hashes de clase generados por JSX,
    // esto apunta a cualquier <p> dentro del formulario con estas tres clases
    // utilitarias estables, que difícilmente cambiarán con actualizaciones menores de React
    var stepEl = document.querySelector(
      '#__next form p.text-primary.text-uppercase.font-weight-bold'
    );

    // PASO 3 — También capturar el título del paso (elemento h1 que sigue al indicador)
    // El combinador de hermano adyacente CSS (+) selecciona el h1 que
    // inmediatamente sigue al párrafo con la etiqueta del paso
    var titleEl = document.querySelector(
      '#__next form p.text-primary.text-uppercase.font-weight-bold + h1'
    );

    // PASO 4 — Salir si alguno de los elementos no se encuentra en el estado actual del DOM
    if (!stepEl || !titleEl) return;

    var stepText = stepEl.innerText.trim();   // ej: "STEP 1", "STEP 2"
    var titleText = titleEl.innerText.trim(); // ej: "Cuéntanos sobre ti"

    // PASO 5 — Solo disparar si el paso realmente cambió desde la última detección
    // Esta es la compuerta de deduplicación — previene envíos redundantes en
    // re-renders de React que no cambian el paso visible
    if (stepText && stepText !== lastStep) {

      // PASO 6 — Enviar paso y título al dataLayer
      window.dataLayer = window.dataLayer || [];
      window.dataLayer.push({
        event: 'formStep',
        formStepText: stepText,       // "STEP 1", "STEP 2", "STEP 3"
        dl_formStepTitle: titleText   // título legible del paso actual
      });

      lastStep = stepText; // actualizar la referencia para la próxima comparación
    }
  }

  // PASO 7 — Observar mutaciones del DOM (igual que v1, pero envuelto en IIFE)
  var observer = new MutationObserver(detectStep);
  observer.observe(document.body, {
    childList: true,
    subtree: true
  });

  // PASO 8 — Ejecutar una vez al inicio con un pequeño retardo para el render inicial
  setTimeout(detectStep, 500);

})();
```

**Mejoras clave respecto a v1:**

| Aspecto | v1 | v2 |
|--------|----|----|
| Estrategia de selectores | Ruta completa con hashes de clase JSX | Solo clases utilitarias semánticas |
| Deduplicación | Flags booleanos (un solo sentido) | Comparación de cadena `lastStep` (permite navegación hacia atrás) |
| Título del paso | No capturado | Capturado en `dl_formStepTitle` |
| Aislamiento de alcance | Variables globales | IIFE — todas las variables tienen alcance local |
| Espera de render inicial | Timeout de 1000ms | Timeout de 500ms |

---

---

## Proyecto 04 — Arquitectura GA4 + Google Ads Multi-Sede (Sin JavaScript Personalizado)

### Contexto de Negocio

Un grupo dental con tres sedes (servidas por un único contenedor GTM) necesitaba que la página de cada sede disparara **tags de conversión separados de Google Ads** para llamadas telefónicas, clics en citas, clics en mapa/direcciones y clics en botones de SMS — sin escribir JavaScript personalizado.

### Enfoque: Segmentación de Triggers por URL

El sitio web usaba rutas de URL distintas por sede. Los triggers nativos de **Clic en Enlace** y **Clic** de GTM fueron configurados con condiciones de ruta de URL, creando una matriz de triggers y tags:

**Patrón de trigger por sede:**
- Clic en enlace de llamada → `href` comienza con `tel:` Y URL de página contiene `/[slug-de-sede]/`
- Clic en agendador de citas → `href` contiene la URL de la plataforma de agendamiento Y URL de página contiene `/[slug-de-sede]/`
- Clic en Google Maps → `href` contiene `maps.google.com` Y URL de página contiene `/[slug-de-sede]/`
- Clic en SMS → `href` comienza con `sms:` Y URL de página contiene `/[slug-de-sede]/`

**Matriz de tags (3 sedes × 4 tipos de acción = 12 tags de conversión de Google Ads + 12 tags de evento GA4)**

### Puntos Destacados de la Arquitectura GTM

- Las 3 sedes comparten un único tag de Configuración GA4 y un Conversion Linker
- La variable `Enhanced Conversions` recopila datos del usuario con hash en el momento de la conversión
- Las acciones de conversión de Google Ads están definidas por sede para permitir la optimización de estrategia de puja a nivel de ubicación
- Microsoft Clarity se carga en todas las páginas para grabación de sesiones y mapas de calor

---

---

## Proyecto 05 — Integración del Widget de Reservas de Terceros NexHealth

### Contexto de Negocio

NexHealth es una plataforma de gestión de consultas dentales que provee un **widget de reservas en línea** embebido en los sitios web de las clínicas mediante un iframe o una página alojada. El widget está completamente controlado por NexHealth y envía sus propios eventos a `window.dataLayer` cuando los pacientes completan pasos de reserva.

El objetivo era capturar estos eventos en GTM sin modificar el código de NexHealth, y enriquecerlos con datos de atribución de campaña provenientes de parámetros URL.

### Cómo NexHealth Envía Datos

El widget de NexHealth dispara un evento personalizado al dataLayer cuando ocurre una acción de reserva. El payload del evento incluye campos como:

```javascript
window.dataLayer.push({
  event: 'nh_booking_event',
  location_id: '...',         // qué sede clínica
  patient_type: '...',        // paciente nuevo o existente
  has_insurance: '...',       // indicador de seguro
  booking_for: '...',         // para sí mismo o para otra persona
  appointment_type: '...'     // tipo de cita seleccionada
});
```

### Configuración GTM

**Trigger de Evento Personalizado:**  
Se dispara cuando se detecta el evento de reserva de NexHealth en el dataLayer.

**Variables DataLayer (capturan los campos del payload de NexHealth):**

| Nombre de Variable | Clave DataLayer | Propósito |
|---------------|---------------|---------|
| NH Event | `event` | Identifica el tipo de evento |
| Location ID | `location_id` | Qué sede fue reservada |
| Patient Type | `patient_type` | Paciente nuevo vs existente |
| Has Insurance | `has_insurance` | Booleano de seguro |
| Booking For | `booking_for` | Para sí mismo o dependiente |
| Appointment Type | `appointment_type` | Tipo de servicio seleccionado |

**Variables de Parámetros URL (capturan atribución de campaña):**

| Nombre de Variable | Parámetro URL | Propósito |
|---------------|--------------|---------|
| Raw URL Param - nh | `?nh=` | Identificador de enlace generado por NexHealth |
| Raw URL Param - ocb | `?ocb=` | Flag de reserva con un clic |
| Campaign Type | `?ct=` | Tipo de campaña de la plataforma publicitaria |
| Campaign ID | `?cid=` | Identificador de campaña |
| Campaign Medium | `?cm=` | Canal/medio |

**Variables de Coincidencia Inteligente:**

Dos variables de Macro JavaScript evalúan si el usuario llegó mediante un enlace especial:

- **Is NH-Generated Link:** Retorna `true` si el parámetro `?nh=` está presente, `false` en caso contrario
- **Is One-Click Booking Link:** Retorna `true` si el parámetro `?ocb=` está presente, `false` en caso contrario

Estas permiten que los tags de GA4 y Ads segmenten las reservas según el tipo de enlace que el paciente usó para llegar a la página de reservas — distinguiendo visitas orgánicas de enlaces de campaña de email o enlaces de reserva con un clic.

---

*Buenas prácticas y recomendaciones → [buenas-practicas.md](buenas-practicas.md)*
