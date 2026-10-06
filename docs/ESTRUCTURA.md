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
public_html/
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
      ChapitaService.php        reglas de chapitas y premios por km
      CanjeService.php          canje en tienda (QR rotativo)
      ComodinService.php        saldo mensual y códigos WOM
      GrupoService.php          grupos de hasta 10
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

Sí, se puede. Se pensó para WOM ahora y otros auspiciadores después, con la regla de la propuesta: otros auspicios solo si no compiten con WOM.

### Auspiciador

Nombre, logo (versión clara y oscura), color, enlace, rubro, tipo (**principal** o **secundario**) y vigencia (desde/hasta).

### Ubicaciones de banner

Cada auspicio se asigna a una o más **ubicaciones**, con un **alcance** (todas las rutas, una categoría o una ruta específica) y fechas.

| Ubicación | Qué es | Ejemplo |
|---|---|---|
| **Presentación de ruta** | Pantalla completa breve al partir, después de validar el inicio | "Esta ruta te la trae WOM" |
| **Ficha de ruta** | Sello chico en la tarjeta y en la ficha | "Presentada por WOM" |
| **Durante la ruta** | Franja discreta en la navegación | Sin tapar el mapa |
| **Hito auspiciado** | Un hito puntual con mención | "Este mirador te lo trae..." |
| **Final de ruta** | Junto al resumen y la chapita | "Retira tu chapita en tiendas WOM" |

Reglas:
- **Un solo auspiciador principal** por ruta a la vez. Por defecto, WOM en todas.
- Los secundarios no pueden ser del mismo rubro que el principal (no otra telco, por ejemplo).
- Las creatividades se cargan en tamaños fijos (para no romper el diseño) y en español e inglés.
- Los banners viajan **dentro del paquete de la ruta**, así se ven sin señal.
- Cada vez que se muestra un banner se cuenta, y el conteo se envía al sincronizar. Sirve para el reporte de resultados a WOM.

---

## 8. Chapitas, premios y canje

- **Chapitas:** 3 categorías (naturaleza, historia, ciudad) × 3 niveles (Cachorro, Explorador, Baqueano). Cada una tiene su animal ilustrado y su ecosistema.
- **Regla de cada chapita**, configurable en el admin. Ejemplo: "Naturaleza · Cachorro = completar 1 ruta de naturaleza".
- **Premios por kilómetros:** a los 10 km, 25 km y más. Pueden ser GB, recargas o comodines.
- Una chapita ganada queda **digital** en la app y, si es física, **canjeable** en tienda.
- **Canje en tienda:** la app muestra un QR que cambia cada 30 segundos (para que no sirva un pantallazo). El vendedor lo escanea desde `/tienda` y lo entrega. Queda registrado quién, dónde y cuándo.
- **Stock por tienda:** el admin ve cuántas quedan en cada tienda (4 a 6 en el piloto).

---

## 9. Usuarios y grupos

- **Login:** código de 6 dígitos por correo, igual que el portal. El **teléfono** se pide para validar en tienda y asociar comodines WOM.
- **Perfil:** nombre o apodo (para el ranking), idioma, comuna (opcional) y consentimientos.
- **Menores de edad:** no se crean cuenta solos. Van como compañeros de sendero de un adulto (ver grupos). Esto se alinea con la Ley 21.719 de datos personales, que entra en vigencia en diciembre de 2026.
- **Grupos de hasta 10:**
  - el **guía** crea la salida e invita;
  - hasta **4 acompañantes** con su propia cuenta y teléfono, que validan por GPS y son **excursionistas**;
  - el resto son **compañeros de sendero**: solo un nombre, sin cuenta ni GPS, y el recorrido queda registrado en el guía.
- **Ranking:** personas y grupos, por mes y total, separado por categoría y dificultad.

---

## 10. Datos (tablas)

Prefijo `exploro_`. IDs como texto (UUID), igual que el portal.

| Tabla | Qué guarda |
|---|---|
| `exploro_rutas` | La ruta (sección 4) |
| `exploro_hitos` | Hitos de cada ruta |
| `exploro_desafios` | Un desafío por hito: `tipo` y su configuración en JSON |
| `exploro_rutas_versiones` | Copia congelada de cada publicación |
| `exploro_auspiciadores` | Marcas |
| `exploro_auspicios` | Ubicación, alcance, fechas y creatividad |
| `exploro_usuarios` | Excursionistas |
| `exploro_usuarios_codigos` | Códigos de login (patrón del portal) |
| `exploro_grupos`, `exploro_grupos_miembros` | Grupos |
| `exploro_recorridos` | Cada vez que alguien hace una ruta: versión, estado, puntos, km |
| `exploro_recorridos_participantes` | Excursionistas y compañeros de sendero de un recorrido |
| `exploro_eventos` | Lo que manda la app: llegada a hito, respuesta, comodín, banner visto. Cada evento tiene un id único para no contarlo dos veces |
| `exploro_chapitas`, `exploro_chapitas_reglas` | Catálogo de chapitas y reglas |
| `exploro_logros` | Chapitas y premios ganados por cada usuario |
| `exploro_tiendas`, `exploro_tiendas_usuarios` | Tiendas WOM y sus vendedores |
| `exploro_canjes` | Entregas en tienda |
| `exploro_stock` | Stock de chapitas por tienda |
| `exploro_comodines_movimientos` | +3 al mes, +N por código, −1 al usar |
| `exploro_codigos_comodin` | Lotes de códigos que reparte WOM |
| `exploro_ranking` | Ranking ya calculado (lo actualiza el cron) |
| `exploro_ajustes` | Ajustes generales |

---

## 11. Recorrido del usuario en la app

1. **Inicio:** rutas cercanas ordenadas por distancia, con filtros por categoría, dificultad y duración.
2. **Ficha de ruta:** foto, duración, distancia, cómo llegar, recomendaciones y sello del auspiciador. Botón **"Preparar ruta"**, que descarga el paquete para usarlo sin señal.
3. **Validar inicio:** QR o santo y seña.
4. **Presentación del auspiciador.**
5. **Navegación:** mapa simple con una flecha, una línea al siguiente hito y la distancia que falta.
6. **Llegada al hito:** texto, audio y desafío.
7. Se repiten los pasos 5 y 6 hasta el último hito.
8. **Final:** aciertos, puntos, km, chapita desbloqueada y dónde retirarla. Banner final.
9. **Perfil:** mis chapitas (álbum), mis km, comodines, ranking y grupo.

---

## 12. Decisiones pendientes

1. ¿Los desafíos del piloto quedan en los 5 tipos de la sección 6, o sumamos alguno?
2. ¿El auspiciador principal es fijo (WOM en todo) o se elige por ruta desde el inicio?
3. ¿Las chapitas se ganan por ruta completada, por cantidad de rutas o por ambas?
4. ¿Login solo con correo, o también con SMS al teléfono desde el piloto? El SMS tiene costo, salvo que WOM lo provea.
5. ¿Los códigos comodín los genera WOM o los generamos nosotros y se los entregamos?
6. ¿Dominio del piloto? Por ejemplo `exploro.cl` o un subdominio de prueba.
