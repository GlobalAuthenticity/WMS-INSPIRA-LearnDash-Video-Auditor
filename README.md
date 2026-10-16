# LearnDash Video Auditor

**Versión:** 2.8.2 · **Requiere:** WordPress 5.8+ · PHP 7.4+ · LearnDash · **Text domain:** `ld-video-auditor`

> Audita videos de YouTube, Vimeo y Bunny en lecciones de LearnDash: edita, exporta, firma embeds de Bunny y exige verlos antes de completar.

## ¿Qué es?

Es un plugin para sitios con **LearnDash** que hace cuatro cosas:

1. **Auditar:** recorre los cursos y sus lecciones y localiza todos los videos de **YouTube**, **Vimeo** y **Bunny Stream**, estén donde estén guardados.
2. **Gestionar:** permite añadir, editar y quitar videos de una lección desde una tabla, sin abrir el editor.
3. **Firmar los embeds de Bunny Stream:** genera el token que exige la opción *Embed View Token Authentication*. El token se recalcula en cada visita, así que no caduca.
4. **Exigir ver los videos:** el botón «Marcar completado» de una lección queda bloqueado, en el navegador y en el servidor, hasta que el alumno termina todos los videos de Bunny de esa lección.

La pantalla está en **LearnDash LMS → Auditor de videos** y solo la ven los usuarios con `manage_options`.

---

## Funcionalidades

### 1. Escaneo de videos
- Se elige un curso y el plugin obtiene sus lecciones mediante `learndash_course_get_steps_by_type()`. Si esa función no está disponible, prueba con `learndash_get_course_steps()` y, por último, con los metadatos de curso de cada lección.
- En cada lección revisa:
  - El contenido (`post_content`).
  - **Todos los metadatos de la lección**, incluidos los serializados (`_sfwd-lessons` y la URL de *Video Progression* de LearnDash).
  - Los campos propios del plugin: `_ldva_video_url`, `_ldva_video_embed` y `_ldva_video_shortcode`. Los shortcodes se ejecutan para ver qué iframe generan.
- Detecta los videos en enlaces directos, iframes y shortcodes que contengan URL.
- Plataformas que reconoce:
  - YouTube: `youtube.com`, `youtu.be` y `youtube-nocookie.com`.
  - Vimeo: `vimeo.com`.
  - Bunny Stream: `player.mediadelivery.net`, `iframe.mediadelivery.net` y `dash.bunny.net`.
- **Evita duplicados:** si la URL y el iframe son del mismo video (mismo ID de YouTube o Vimeo, o misma biblioteca e ID de Bunny), aparecen en una sola fila.
- Ignora los scripts y enlaces auxiliares de Vimeo que no son videos.
- Las lecciones sin video aparecen con el estado **«No encontrado»**.

### 2. Tabla de resultados
- La tabla se carga por AJAX (`ldva_scan`).
- Columnas: **curso, lección, plataforma, URL original, HTML embebido, shortcode, estado y acciones**.
- Filtros por **plataforma** (YouTube, Vimeo o Bunny Stream) y por **estado** (Encontrado o No encontrado), y un buscador por curso, lección o URL.
- Un botón para **copiar** la URL, el HTML o el shortcode de cada fila.
- El menú **«Acciones»** de cada fila tiene tres opciones:
  - **Ver lección:** abre la lección con su enlace dentro del curso.
  - **Añadir o editar video:** abre un modal.
  - **Remover video.**
- En móvil y tablet, la tabla se muestra como tarjetas.
- En esta pantalla se ocultan los avisos de otros plugins.

### 3. Añadir y editar videos
- El modal admite tres formas de indicar el video: una **URL**, un **HTML embebido** (solo se permite un iframe, filtrado con `wp_kses`) o un **shortcode**.
- Al editar, sustituye el video original en el sitio donde estaba (el contenido o la URL de *Video Progression*), buscándolo por su ID aunque el iframe tenga otros atributos. Así no se crea una fila nueva.
- **Videos de Bunny:** siempre se guardan como el shortcode `[bunnycdn_stream_video library="…" id="…"]`, nunca como un iframe suelto. Si no se puede sacar la biblioteca y el ID de lo que se pegó, aparece un error en lugar de guardar un iframe sin firmar.
- Si la lección no tiene video en el contenido, el iframe o el shortcode guardados por el plugin se **añaden al final del contenido** (filtro `the_content`), con proporción 16:9.

### 4. Quitar videos
- Quita el video del contenido y de **todos los metadatos de la lección**, incluidos los serializados.
- Si el video se repite en varios sitios de la lección, borra todas las copias.

### 5. Exportación
| Botón | Formato | Contenido |
|---|---|---|
| Exportar CSV | `.csv` | Curso actual con los filtros aplicados: curso, lección, plataforma, URL, HTML embebido, shortcode y estado |
| Exportar Excel | `.xlsx` | Lo mismo que el CSV, en Excel |
| Excel de todos los cursos | `.xlsx` | Todos los cursos, **solo las lecciones con video**: curso, lección, plataforma y enlace |

- El CSV está protegido contra **inyección de fórmulas**: los valores que empiezan por `=`, `+`, `-` o `@` no se ejecutan al abrir el archivo en Excel.

### 6. Firma de embeds de Bunny Stream
- La sección **«Autenticación por token (Bunny Stream)»** guarda, para cada biblioteca, su clave de *Embed View Token Authentication* y el tiempo de expiración del token. La casilla **«No expira»** equivale a 10 años, porque Bunny siempre exige una fecha de expiración.
- Los datos se guardan en la opción `ldva_bunny_tokens`. Cada biblioteca se puede borrar al momento con el icono de papelera.
- Cálculo del token: `token = SHA256(clave + video_id + expires)`.
- Se firma **en cada carga de página**, de dos formas:
  - Al mostrar el shortcode `[bunnycdn_stream_video]`. El plugin lo registra como respaldo solo si ningún otro plugin lo ha registrado ya.
  - Revisando el HTML final de la lección (filtro `the_content`) y firmando **cualquier iframe de Bunny**, aunque se haya pegado a mano.
- El atributo `tokenAuth="false"` desactiva la firma para un video concreto.
- Si la biblioteca no tiene clave configurada, el video se muestra sin firmar.

### 7. Ver los videos de Bunny antes de completar la lección
- **En el navegador** (`bunny-progression.js`):
  - Detecta los iframes de Bunny de la lección y escucha el evento de fin de reproducción.
  - Bloquea el botón «Marcar completado» y muestra un aviso debajo del video. El aviso desaparece cuando se han visto todos.
  - Guarda en el navegador qué videos se han visto, para que no se pierda al recargar la página.
- **En el servidor:**
  - El navegador envía cada video terminado a `ldva_mark_video_watched`, con nonce. El servidor comprueba que el video **pertenece de verdad a esa lección**.
  - El progreso de cada alumno se guarda en el usermeta `_ldva_bunny_watched_{lesson_id}`.
  - El filtro `learndash_process_mark_complete` **rechaza la finalización** mientras falte algún video, aunque alguien intente forzarla desde el navegador.
  - Los administradores no tienen este bloqueo. Se puede desactivar con el filtro `ldva_enforce_bunny_progression`.

---

## Datos que guarda

| Tipo | Clave | Uso |
|---|---|---|
| postmeta | `_ldva_video_url` | URL del video guardada por el plugin |
| postmeta | `_ldva_video_embed` | Iframe guardado por el plugin |
| postmeta | `_ldva_video_shortcode` | Shortcode guardado (Bunny) |
| usermeta | `_ldva_bunny_watched_{lesson_id}` | Videos de Bunny que el alumno ha terminado en cada lección |
| option | `ldva_bunny_tokens` | Clave y expiración del token de cada biblioteca de Bunny |

Al **desinstalar** el plugin se borran todos estos datos (`uninstall.php`).

## Endpoints AJAX
| Acción | Quién puede usarla | Qué hace |
|---|---|---|
| `ldva_scan` | `manage_options` | Escanea un curso |
| `ldva_save_video` | `manage_options` | Añade o edita un video |
| `ldva_remove_video` | `manage_options` | Quita un video |
| `ldva_save_bunny_tokens` / `ldva_delete_bunny_token` | `manage_options` | Guarda o borra la configuración de tokens de Bunny |
| `ldva_mark_video_watched` | Usuario con sesión iniciada | Registra que el alumno terminó un video de Bunny |
| `admin_post_ldva_export` | `manage_options` | Descarga el CSV o el Excel |

Todas las acciones comprueban un nonce.

## Arquitectura
```
learndash-video-auditor/
├── learndash-video-auditor.php    # Bootstrap
├── includes/
│   ├── class-ldva-scanner.php     # Descubrimiento de lecciones y extracción de videos
│   ├── class-ldva-admin.php       # Pantalla, AJAX, edición, tokens Bunny, progresión
│   └── class-ldva-exporter.php    # CSV y XLSX
├── views/admin-page.php           # Plantilla de la pantalla
├── assets/                        # admin.js/css, public.css, bunny-progression.js
└── uninstall.php                  # Limpieza de metadatos y opciones
```

## Limitaciones conocidas
- Las lecciones construidas **solo con Elementor** guardan su contenido en `_elementor_data`. Ese contenido puede no pasar por `the_content`, y entonces sus iframes de Bunny podrían quedar sin firmar.
- El bloqueo de «Marcar completado» solo se aplica a los videos de **Bunny Stream**. YouTube y Vimeo se auditan, pero no bloquean la lección.
