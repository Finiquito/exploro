# Exploro: estructura del proyecto

Documento base para "Rutas de Chile by WOM" (nombre interno: Exploro). Define dónde va cada cosa, qué datos tiene una ruta y cómo funcionan los hitos, los desafíos y los auspicios. Este documento se escribe **antes** del código, y lo que se decida acá manda.

Estado: **borrador para discutir**. Las decisiones pendientes están al final.

---

## 1. Piezas del sistema

Hay cuatro piezas. Tres viven dentro de TypeDock y una es la app del usuario.

| Pieza | Para quién | Dónde vive | Tecnología |
|---|---|---|---|
| **Admin** | Nosotros: contenido, rutas, auspicios, reportes | `/admin/exploro/...` | Plugin TypeDock (PHP + Latte), igual que el portal de clientes |
| **Panel de tienda** | Vendedores de WOM que entregan chapitas | `/tienda` | El mismo plugin, con un login aparte (como `/equipo` en el portal) |
| **API** | Lo usa la app | `/api/exploro/...` | El mismo plugin, con rutas que responden JSON |
| **App** | Excursionistas, familias y turistas | `/app/` | PWA en JS (Vite + MapLibre), archivos estáticos |

Además, cada ruta publicada genera un **paquete de archivos estáticos** (JSON, mapa, audios e imágenes) que la app descarga para funcionar sin señal.

```
                 ┌──────────── TypeDock + plugin "exploro" ────────────┐
  Equipo   ───►  │  Admin            Panel tienda         API JSON      │
  WOM tienda ──► │  (rutas, hitos,   (canje de           (login, avance, │
                 │   auspicios)       chapitas)           ranking)       │
                 └───────┬─────────────────────────────────────▲───────┘
                         │ "Publicar ruta"                     │ avance
                         ▼                                     │ (cuando hay señal)
               /contenido/rutas/<ruta>/v<n>/   ──descarga──►  App PWA (/app/)
               ruta.json · mapa.pmtiles · audios               funciona offline
```

**Regla de oro:** la app nunca depende de PHP para *mostrar* una ruta. PHP solo recibe el avance y entrega lo personal (puntos, chapitas, ranking). Así un peak de lanzamiento casi no toca el servidor.

---

## 2. En el servidor de pruebas (Hostinger, plan Empresa)

No hace falta nada raro:

- **PHP 8.2+ y MySQL/MariaDB:** TypeDock y el plugin, igual que el portal.
- **Node.js:** solo para *compilar* la app (`npm run build`), y eso se puede hacer en tu computador. El servidor no necesita correr Node.
- **SSH:** cómodo para subir con `rsync` o `git pull`, pero no obligatorio. También sirve el administrador de archivos.
- **HTTPS:** obligatorio, porque sin él no funciona el modo offline (service worker) ni el GPS. Hostinger trae SSL gratis.
- **Cron:** para la cola de correos y recalcular rankings. Se configura en hPanel.
- Una línea en `.htaccess` para que el servidor entregue bien los archivos de mapa:
  `AddType application/vnd.pmtiles .pmtiles`

```
public_html/                 (exploro.richgt.com)
  (TypeDock)                 núcleo, temas, plugins/
  plugins/exploro/           ← el plugin (se sube como zip desde el admin)
  app/                       ← la PWA compilada (index.html, js, css, sw.js)
  contenido/rutas/           ← paquetes publicados (los escribe el plugin)
    cerro-chena/
      v3/ ruta.json · mapa.pmtiles · audio/ · img/
```

Para producción con WOM (abril) se evalúa si basta Hostinger con su CDN o si conviene mover a AWS. La estructura es la misma en ambos casos.

---

## 3. Estructura del repositorio

Sigue el patrón del portal de clientes, para que el equipo se mueva en terreno conocido.

```
exploro/
  plugin/                     el plugin, tal como se instala (zip → admin TypeDock)
    plugin.json
    src/                      servicios y controladores PHP
      ExploroPlugin.php         registra rutas de admin, tienda y API
      RutaService.php           rutas, hitos, versiones
      PublicadorRuta.php        arma el paquete estático al publicar
      DesafioService.php        tipos de desafío y su validación
      AuspicioService.php       auspiciadores y ubicaciones de banners
      RecorridoService.php      recibe el avance y valida (GPS, QR, tiempos)
      PuntajeService.php        puntos, km, ranking
      PremioService.php         premios por ruta, cantidad de rutas y km
      CanjeService.php          canje en tienda (QR rotativo)
      ComodinService.php        saldo, tramos por compra, boletas y códigos
      SalidaService.php         salidas en grupo (guía, acompañantes, invitados)
      UsuarioAuthService.php    login por código (reusa el patrón del portal)
      TiendaAuthService.php
      Schema.php                columnas nuevas sin borrar datos
    templates/
      admin/                  Latte del admin
      tienda/                 Latte del panel de tienda
    migrations/               SQL
    assets/                   CSS y JS del admin (incluye el editor de mapa)
    assets-src/
  app/                        la PWA (fuente)
    src/
      pantallas/              inicio, ficha de ruta, navegación, hito, final, perfil...
      mapa/                   MapLibre, flecha, distancia al hito
      desafios/               un módulo por tipo de desafío (ver sección 6)
      offline/                descarga de paquetes, cola de avance, sincronización
      auspicios/              cómo se muestran los banners
      i18n/                   español e inglés
    public/
    vite.config.js
  dev/                        núcleo TypeDock simulado (copiado del portal)
  tests/                      pruebas PHP (SQLite y MariaDB) y de la app
  scripts/
    zip.sh                    arma el zip del plugin
    build-app.sh              compila la app y la deja lista para subir
    mapa-ruta.sh              recorta el mapa de una ruta (.pmtiles)
  docs/
    ESTRUCTURA.md             este documento
    formato-paquete-ruta.md   el JSON que lee la app, con ejemplo
```

---

## 4. Qué tiene una ruta

### Ruta

| Campo | Ejemplo | Nota |
|---|---|---|
| Nombre (es / en) | "Cerro Chena: el Pucará" | |
| Slug | `cerro-chena` | va en la URL |
| Categoría | naturaleza · historia · ciudad | define qué chapitas suma |
| Dificultad | iniciado · avanzado | |
| Modo | a pie · bicicleta · ambos | |
| Duración estimada | 90 min | 45 min a 3 h |
| Distancia | 4,2 km | se calcula desde la línea dibujada |
| Desnivel | 180 m | opcional |
| Región y comuna | RM, San Bernardo | para filtrar y "cerca de ti" |
| Punto de inicio | lat/lng | |
| Cómo llegar (es / en) | "Metro Lo Ovalle y bus..." | para turistas |
| Validación de inicio | QR · santo y seña · ambos | ver sección 5 |
| Línea del recorrido | dibujada en el mapa del admin | se guarda como GeoJSON |
| Portada e imágenes | | |
| Recomendaciones | agua, zapatillas, horario, sin sombra | |
| Seguridad | zonas con señal, teléfono de ayuda, punto de salida | |
| Estado | borrador · publicada · pausada | "pausada" la oculta sin borrarla (ej. sendero cerrado) |
| Versión publicada | v3 | ver "versiones" abajo |

### Hito (5 a 12 por ruta)

| Campo | Nota |
|---|---|
| Orden | 1, 2, 3... |
| Nombre (es / en) | "La muralla del Pucará" |
| Ubicación | lat/lng, se pone en el mapa del admin |
| Radio de llegada | 30 m por defecto. Se agranda en quebradas o bosque, donde el GPS falla más |
| Texto (es / en) | lo que se lee al llegar |
| Audio (es / en) | audioguía, opcional |
| Imagen | opcional |
| Desafío | opcional (ver sección 6) |
| Puntos | por llegar al hito, más los del desafío |

### Versiones

Al apretar **Publicar**, el plugin congela una copia de la ruta (v1, v2, v3...) y arma su paquete. Si mañana el equipo de terreno mueve un hito, se publica la v4, pero quien ya descargó la v3 y va caminando sin señal termina tranquilo con la v3. El servidor acepta el avance de ambas versiones.

---

## 5. Cómo se valida que estás ahí

- **Inicio:** QR pegado en el lugar, que la app lee con la cámara. Si no hay QR, un **santo y seña**: una pregunta que solo se responde estando ahí ("¿qué animal está pintado en el letrero de la entrada?").
- **Cada hito:** llegada por GPS dentro del radio. Si el GPS no da, el santo y seña del hito sirve de respaldo.
- **Antitrampas:** el teléfono valida al tiro para que la experiencia sea fluida, pero los puntos y chapitas **se confirman en el servidor** al sincronizar. El servidor revisa que los tiempos y distancias entre hitos tengan sentido (nadie camina 2 km en 1 minuto). Además, hay un canje por ruta por persona.
- **En grupo:** cada acompañante valida con su propio GPS. Los invitados van con el guía (sección 9).

---

## 6. Desafíos: la gamificación de cada hito

Respuesta corta: **sí, la sopa de letras es posible y es una buena idea**. La propuesta es que cada hito tenga un desafío de **un tipo a elegir** en el admin, no solo preguntas.

Cada tipo de desafío es un **módulo independiente** en la app (`app/src/desafios/<tipo>/`). Todos funcionan igual por fuera:
- **reciben** su configuración (palabras, opciones, imágenes);
- **entregan** un resultado: completado sí/no, aciertos, intentos, tiempo y si usó comodín.

Agregar un tipo nuevo después no obliga a tocar el resto de la app. Todos funcionan **sin señal**, porque su contenido viene dentro del paquete de la ruta.

### Para el piloto (febrero)

| Tipo | Cómo se juega | Qué se carga en el admin |
|---|---|---|
| **Pregunta** | 3 o 4 alternativas | Pregunta, alternativas, cuál es la correcta y una explicación que se muestra al responder |
| **Observa y responde** | Respuesta escrita o número: "¿cuántos arcos tiene la fachada?" | Pregunta y respuestas aceptadas (sin importar tildes ni mayúsculas) |
| **Sopa de letras** | Encontrar 4 a 8 palabras del lugar (flora, fauna, historia) | Palabras y tamaño de la grilla. La grilla la arma la app sola, siempre igual para la misma ruta |
| **Ordena** | Arrastrar para ordenar: hechos históricos, capas de roca, etapas | Elementos en el orden correcto |
| **Solo llegada** | Sin juego: audio, texto y puntos por llegar | Nada extra |

### Para después del piloto

| Tipo | Idea |
|---|---|
| **Memorice** | Parejas de fauna y flora local, con las ilustraciones de las chapitas |
| **Une con una línea** | Animal con su huella, planta con su nombre |
| **Rompecabezas** | Foto del hito en piezas |
| **Ahorcado / palabra escondida** | Una palabra clave del lugar |
| **Brújula** | "Apunta hacia el volcán": usa la orientación del teléfono (en iPhone pide permiso) |
| **Foto reto** | "Encuentra y fotografía un quillay". Es por honor, y la foto queda solo en tu teléfono por privacidad |
| **Búsqueda del tesoro** | El hito no se marca en el mapa: hay que encontrarlo con pistas. Sirve para la "gran búsqueda del tesoro" del año 2 |

### Puntaje y comodines

- Cada desafío da puntos según aciertos e intentos. Hay un **% de errores permitido** por ruta.
- Un **comodín** sirve para saltar un desafío difícil o recuperarte si te pasaste del % de errores. Queda registrado (cuenta para métricas).
- Dentro de los juegos no se cobra ni se compra nada: los comodines se tienen antes de salir, como dice la propuesta.

---

## 7. Auspicios y banners

Hay dos niveles:

- **Auspiciador:** la marca. Nombre, logo (versión clara y oscura), color, enlace y rubro.
- **Auspicio:** lo que esa marca pone en una ruta. Trae todo su material: la pantalla de presentación, los banners y los textos. Una marca puede tener varios auspicios (por ejemplo "WOM verano" y "WOM invierno").

**Cada ruta elige su auspicio** en un selector. Si existe uno solo, viene elegido por defecto. Si se cambia, se aplica desde la siguiente versión publicada de la ruta.

### Lo que trae un auspicio

| Ubicación | Qué es | Ejemplo |
|---|---|---|
| **Presentación de ruta** | Pantalla completa breve al partir, después de validar el inicio | "Esta ruta te la trae WOM" |
| **Ficha de ruta** | Sello chico en la tarjeta y en la ficha | "Presentada por WOM" |
| **Durante la ruta** | Franja discreta en la navegación | Sin tapar el mapa |
| **Hito auspiciado** | Mención en un hito puntual (opcional) | "Este mirador te lo trae..." |
| **Final de ruta** | Junto al resumen y la chapita | "Retira tu chapita en tiendas WOM" |

Cada ubicación es opcional: si el auspicio no la trae, no se muestra nada ahí.

Reglas:
- Las creatividades se cargan en tamaños fijos (para no romper el diseño) y en español e inglés.
- Los banners viajan **dentro del paquete de la ruta**, así se ven sin señal.
- Cada vez que se muestra un banner se cuenta, y el conteo se envía al sincronizar. Sirve para el reporte de resultados al auspiciador.
- Otros auspiciadores, solo si no compiten con WOM (dato de rubro, revisado en el admin).

---

## 8. Chapitas, premios y canje

### Qué se gana y cómo

Todo se gana con **rutas terminadas**. La categoría y la dificultad de cada ruta (sección 4) determinan qué suma.

| Premio | Se gana por | Ejemplo |
|---|---|---|
| **Chapita de ruta** | Terminar esa ruta | Chapita del zorrito al terminar Cerro Chena |
| **Chapita de experiencia** | N rutas terminadas de una categoría | 3 rutas de naturaleza = Raíces |
| **Chapita de kilómetros** | Km acumulados en rutas terminadas, de cualquier categoría | 10 km = Cachorro |

### Chapitas de kilómetros

| Km | Chapita |
|---|---|
| 10 | Cachorro |
| 25 | Sabueso |
| 50 | *(nombre por definir)* |

### Chapitas de experiencia (por categoría)

| Rutas terminadas | Naturaleza | Historia | Ciudad |
|---|---|---|---|
| 3 | Raíces | Descubridor | *(por definir)* |
| 5 | Árbol | Escriba | *(por definir)* |
| 10 | Gran Árbol | Gran Maestre | *(por definir)* |

Los nombres son provisorios y se cambian en el admin sin tocar código. La dificultad queda como filtro opcional de cada premio, por si más adelante se quiere, por ejemplo, una chapita especial por 3 rutas avanzadas.

- Cada premio se configura en el admin: **condición** (ruta, cantidad o km), **filtro** (categoría y/o dificultad), **qué se entrega** (chapita física, GB, recarga, comodines o solo digital) y **vigencia**.
- Todo premio queda **digital** en el álbum de la app. Si es físico, además es **canjeable** en tienda.

### Canje en tienda

- La app muestra un QR que cambia cada 30 segundos (para que no sirva un pantallazo). El vendedor lo escanea desde `/tienda`, ve qué corresponde entregar y lo marca como entregado. Queda registrado quién, dónde y cuándo.
- El QR del **guía** incluye sus premios **y los de sus invitados** (sección 9). El vendedor ve el detalle: "Juan (guía) + 3 invitados: Sofía 8 años, Tomás 11 años...".
- **Stock por tienda:** el admin ve cuántas quedan en cada tienda (4 a 6 en el piloto).
- Un canje por ruta por persona.

---

## 9. Usuarios, grupos y salidas

### Cuentas

- **Login:** código de 6 dígitos por correo, igual que el portal (el SMS se explica más abajo).
- **Perfil:** nombre o apodo (para el ranking), teléfono, idioma, comuna (opcional) y consentimientos.

### Salidas en grupo

Cada vez que alguien parte una ruta se crea una **salida**. Quien la crea es el **guía**. Máximo **10 personas** por salida:

| | Máximo |
|---|---|
| Guía | 1 |
| Acompañantes con cuenta | 5 |
| Invitados sin cuenta | 4 |
| **Total** | **10** |

| Rol | Quién es | Valida | Responde desafíos | Recibe premios |
|---|---|---|---|---|
| **Guía** | Quien inscribe la salida | Con su celular: inicio y GPS en cada hito | **Sí, por todo el grupo** | Sí, en su cuenta |
| **Acompañante** | Persona con cuenta propia que se une a la salida | Con su propio celular: inicio y GPS en cada hito, igual que el guía | No hace falta: participa en la dinámica que lleva el guía | Sí, en su cuenta, igual que el guía |
| **Invitado** | Persona sin cuenta, agregada por el guía con **nombre y edad** | No valida: va con el guía | No | Sí, los retira el guía en tienda |

Cómo funciona:
1. El guía crea la salida y agrega a los invitados (nombre y edad).
2. Los acompañantes se unen escaneando un **QR de la salida** en el celular del guía. Cada uno necesita su cuenta.
3. El guía valida el inicio. Cada acompañante también lo valida en su celular.
4. En cada hito, el guía lee el texto y lanza el desafío. El grupo participa y el guía responde. Cada acompañante debe pasar por el hito con su GPS.
5. Al terminar:
   - **guía y acompañantes** que pasaron por todos los hitos reciben la ruta terminada, los puntos, los km y sus premios en sus propias cuentas;
   - **los invitados** reciben sus premios asociados al guía, que los retira en tienda.

Ventaja: nadie tiene que ir contestando en su celular, y aun así cada persona recibe su chapita.

**Antiabuso de invitados** (porque las chapitas son físicas):
- los invitados de una salida deben tener nombres distintos;
- tope por guía de **40 personas al mes** sumando todas sus salidas, de las cuales **máximo 16** pueden ser acompañantes con cuenta (ambos números se ajustan en el admin). Pensado para un grupo que sale una vez cada fin de semana;
- el vendedor ve la lista completa al canjear.

**Datos de menores:** a los invitados solo se les pide nombre (o apodo) y edad. Así se cumple con lo mínimo que exige la Ley 21.719 de datos personales, vigente desde diciembre de 2026.

### Login con correo y SMS

**Correo (piloto):** es lo mismo que ya hace el portal de clientes (`CodigoAccesoService`).
1. La persona escribe su correo.
2. El servidor genera un código de 6 dígitos, lo guarda con vencimiento (10 min) y lo envía.
3. La persona escribe el código y queda con la sesión iniciada.

Ya tiene límite de intentos y protección CSRF.

**Para que no lleguen a spam:** los correos se envían con **Brevo** por SMTP. El plan gratis alcanza para las pruebas, con unos 300 correos al día, y el portal ya soporta SMTP propio. Lo importante es configurar en el DNS de `richgt.com` (en Hostinger) los registros **SPF, DKIM y DMARC** que entrega Brevo. Esos registros le prueban a Gmail y Outlook que el correo es legítimo. Alternativa: Resend (gratis hasta unos 100 al día).

**SMS (después del piloto):** es el mismo flujo, pero el código va por SMS en vez de correo. PHP llama a un **proveedor de SMS** (por ejemplo Twilio, Amazon SNS o un proveedor chileno), que le envía el mensaje al teléfono. Cada SMS tiene un costo que hay que cotizar. WOM, al ser telco, podría proveer el envío.

**Propuesta:**
- **Piloto:** login por correo. El teléfono se pide en el perfil sin verificar. En tienda, el QR rotativo solo sale de una cuenta con sesión iniciada, así que eso valida a la persona.
- **Año completo:** se agrega la verificación del teléfono por SMS una sola vez (al primer canje o al activar la carga automática de comodines por compras), así se mandan pocos SMS.

### Ranking

Personas y grupos, por mes y total, separado por categoría y dificultad.

---

## 10. Comodines: se ganan comprando en WOM

### De dónde salen

| Origen | Cómo | Ejemplo |
|---|---|---|
| **Mensuales** | Automático, el día 1 | 3 gratis al mes para todos |
| **Compras en WOM** | Recarga, plan o compra en tienda, según una tabla de tramos | Recarga de $10.000 = 1, de $30.000 = 4 |
| **Códigos de campaña** | Lotes que generamos en el admin (eventos, prensa, alianzas) | `EXP-7K3M-Q9TD` = 2 comodines |
| **Premios** | Un premio puede entregar comodines (sección 8) | 25 km = 3 comodines |

### Tabla de tramos (editable en el admin)

Cada **tipo de compra** tiene sus tramos. Son tramos y no una proporción fija, para premiar más a quien compra más.

| Tipo de compra | Desde | Comodines |
|---|---|---|
| Recarga | $10.000 | 1 |
| Recarga | $20.000 | 2 |
| Recarga | $30.000 | 4 |
| Plan (pago de cuenta) | cualquier monto | 2 |
| Compra en tienda (accesorios, equipos) | $20.000 | 1 |
| Compra en tienda | $50.000 | 2 |

Los montos son de ejemplo; WOM define los reales. Se aplica el tramo más alto que alcance la compra. Topes:
- una boleta se usa **una sola vez**, por una sola persona;
- máximo de comodines por compras al mes por persona (configurable);
- la boleta debe ser de los **últimos 30 días**.

### Cómo se valida la compra

No tenemos acceso a los sistemas de WOM, así que hay que elegir cómo comprobar que la boleta existe. Como un comodín solo ayuda en el juego y no entrega premios físicos, la validación puede ser más liviana que la de las chapitas.

| Vía | Cómo funciona | Qué necesita de WOM | Cuándo |
|---|---|---|---|
| **A. En tienda** | Al pagar, el vendedor escanea el QR de la app desde `/tienda`, elige el tipo de compra, escribe el monto y el n° de boleta. Los comodines se cargan al instante | Nada: solo que el vendedor lo haga | **Piloto** (4 a 6 tiendas) |
| **B. La persona ingresa su boleta** | En la app: tipo, n° de boleta, monto y fecha (puede escanear el código de barras de la boleta). Queda **"por confirmar"** | Un archivo periódico (CSV semanal) con las boletas: n°, monto, fecha y tipo. El admin lo sube y el sistema confirma las que coinciden | **Piloto**, si WOM entrega el archivo |
| **C. Automático por teléfono** (más adelante) | WOM envía cada día las recargas y pagos por número de teléfono. Se cargan solos a quien tenga ese teléfono verificado | Archivo diario o API, y un acuerdo de datos. Requiere verificar el teléfono por SMS | **Año completo** |

Sobre la vía B:
- mientras está "por confirmar", la persona ve el comodín como pendiente y no lo puede usar;
- si en 15 días no aparece en el archivo, se rechaza con un aviso;
- si WOM no puede entregar el archivo, se puede leer el **timbre electrónico** de la boleta (el código de barras del SII), que trae el RUT de quien emite, el folio, el monto y la fecha. Así se comprueba al menos que es una boleta de WOM. Es más trabajo, por eso queda como alternativa.

### Boletas de prueba (piloto)

Mientras WOM no entregue su archivo, el admin genera un **lote de boletas de prueba**: X números con tipo de compra y monto. El sistema los trata como si vinieran del archivo de WOM. Así se prueba la vía B completa con los usuarios de prueba. Las boletas de prueba quedan marcadas y se desactivan antes de abrir al público.

### Códigos de campaña

Los generamos nosotros desde el admin:
- **Lote:** nombre ("Lanzamiento Cerro Chena"), cantidad de códigos, comodines que da cada código y vigencia.
- **Formato:** fácil de dictar y sin letras confusas (sin 0/O ni 1/I). Ejemplo: `EXP-7K3M-Q9TD`.
- Cada código se usa **una sola vez**. El lote se exporta en CSV.

---

## 11. Datos (tablas)

Prefijo `exploro_`. IDs como texto (UUID), igual que el portal.

| Tabla | Qué guarda |
|---|---|
| `exploro_rutas` | La ruta (sección 4) |
| `exploro_hitos` | Hitos de cada ruta |
| `exploro_desafios` | Un desafío por hito: `tipo` y su configuración en JSON |
| `exploro_rutas_versiones` | Copia congelada de cada publicación |
| `exploro_auspiciadores` | Marcas |
| `exploro_auspicios` | Material de cada auspicio: pantalla, banners y textos por ubicación |
| `exploro_usuarios` | Excursionistas |
| `exploro_usuarios_codigos` | Códigos de login (patrón del portal) |
| `exploro_salidas` | Cada vez que un guía parte una ruta: versión, estado, código QR para unirse |
| `exploro_salidas_participantes` | Guía, acompañantes (con cuenta) e invitados (nombre y edad); si validó y si terminó |
| `exploro_eventos` | Lo que manda la app: llegada a hito, respuesta, comodín, banner visto. Cada evento tiene un id único para no contarlo dos veces |
| `exploro_recorridos` | Resultado de cada persona con cuenta en una salida: puntos, km, aciertos |
| `exploro_premios` | Catálogo: chapitas, GB, recargas, comodines. Condición (ruta, cantidad o km), categoría, dificultad y vigencia |
| `exploro_logros` | Premios ganados por cada participante (cuenta propia o invitado del guía) |
| `exploro_tiendas`, `exploro_tiendas_usuarios` | Tiendas WOM y sus vendedores |
| `exploro_canjes` | Entregas en tienda |
| `exploro_stock` | Stock de chapitas por tienda |
| `exploro_comodines_movimientos` | Saldo: +3 al mes, +N por compra, código o premio, −1 al usar |
| `exploro_comodin_tramos` | Tabla de tramos por tipo de compra |
| `exploro_compras` | Boletas registradas: tipo, n°, monto, fecha, vía (tienda, persona, archivo), estado (por confirmar, confirmada, rechazada) |
| `exploro_compras_importaciones` | Archivos de boletas que entrega WOM y su resultado |
| `exploro_comodin_lotes`, `exploro_comodin_codigos` | Códigos de campaña que generamos |
| `exploro_ranking` | Ranking ya calculado (lo actualiza el cron) |
| `exploro_ajustes` | Ajustes generales |

---

## 12. Recorrido del usuario en la app

1. **Inicio:** rutas cercanas ordenadas por distancia, con filtros por categoría, dificultad y duración.
2. **Ficha de ruta:** foto, duración, distancia, cómo llegar, recomendaciones y sello del auspiciador. Botón **"Preparar ruta"**, que descarga el paquete para usarlo sin señal.
3. **Armar la salida (opcional):** agregar invitados y mostrar el QR para que se unan los acompañantes.
4. **Validar inicio:** QR o santo y seña.
5. **Presentación del auspiciador.**
6. **Navegación:** mapa simple con una flecha, una línea al siguiente hito y la distancia que falta.
7. **Llegada al hito:** texto, audio y desafío.
8. Se repiten los pasos 6 y 7 hasta el último hito.
9. **Final:** aciertos, puntos, km, chapita desbloqueada y dónde retirarla. Banner final.
10. **Perfil:** mis chapitas (álbum), mis km, comodines, ranking y grupo.

---

## 13. Decisiones

### Tomadas

| # | Tema | Decisión |
|---|---|---|
| 1 | Desafíos del piloto | Pregunta, observa y responde, sopa de letras, ordena y solo llegada |
| 2 | Auspicios | Cada ruta elige un auspicio; si hay uno solo, viene por defecto. El auspicio trae su pantalla, banners y textos |
| 3 | Premios | Por ruta terminada, por cantidad de rutas y por km, filtrados por categoría y dificultad. Salidas en grupo de hasta 10 (guía, acompañantes con cuenta e invitados con nombre y edad) |
| 4 | Login | Piloto con código por correo, enviado con Brevo (SPF, DKIM y DMARC configurados); SMS después, solo para verificar el teléfono una vez |
| 5 | Comodines | 3 al mes, más los que se ganan comprando en WOM según la tabla de tramos. Vías A (en tienda, en el mismo panel de las chapitas) y B (boleta + archivo de WOM). La vía C queda para después. Para el piloto, lote de boletas de prueba |
| 7 | Chapitas | Por km (10 Cachorro, 25 Sabueso, 50 por definir) y por experiencia: 3, 5 y 10 rutas por categoría (Naturaleza: Raíces, Árbol, Gran Árbol; Historia: Descubridor, Escriba, Gran Maestre) |
| 8 | Grupos | Guía + hasta 5 con cuenta + hasta 4 sin cuenta = 10. Tope por guía: 40 personas al mes, máximo 16 con cuenta |
| 6 | Dominio de pruebas | `exploro.richgt.com` (Hostinger) |

### Pendientes

1. Nombre de la chapita de 50 km y nombres de experiencia para Ciudad.
2. Montos reales de los tramos de comodines (los define WOM).
3. ¿WOM puede entregar un archivo periódico con las boletas?
