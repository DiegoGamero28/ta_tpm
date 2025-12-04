# SYSFORUM – Memoria global del proyecto

Esta memoria presenta una visión holística de **SYSFORUM**: qué problema resuelve, cómo está estructurado el proyecto, cómo se implementaron sus funcionalidades principales y por qué se tomaron ciertas decisiones de diseño y de código.

Se hace referencia a los temas trabajados en clase. Se profundiza especialmente en los contenidos de las **semanas 9 a 14** (hardware, sonido, GPS, menús, Firebase, Maps, Storage, tiempo real), y se da una vista más breve de las **semanas 1 a 6** (estructura de proyecto, Activities, layouts, RecyclerView, etc.).

---

## 1. Visión general del proyecto

SYSFORUM es una aplicación Android tipo foro académico donde los usuarios pueden:

- Registrarse e iniciar sesión con correo/contraseña (Firebase Authentication).
- Publicar preguntas clasificadas por categoría, con texto, imagen y ubicación.
- Ver un feed en tiempo real de preguntas recientes (Firestore + RecyclerView).
- Consultar el detalle de cada pregunta, reaccionar (likes/dislikes), comentar y ver la ubicación en Google Maps.
- Ver y editar su perfil (avatar, biografía), acumular puntos por actividad.
- Configurar opciones de la app (notificaciones, tema, seguridad de la cuenta).

A nivel técnico, es un proyecto Android **nativo** en Kotlin, con fuerte integración de **Firebase** y uso de componentes **Material Design**.

---

## 2. Estructura de proyecto (resumen semanas 1–2)

El módulo principal es `app/`. La estructura clave es:

- `app/src/main/AndroidManifest.xml` – define Activities, permisos, tema de la aplicación y servicios.
- `app/src/main/java/com/sysforum/app/` – código Kotlin:
  - `MainActivity`, `CreateQuestionActivity`, `QuestionDetailsActivity`, `SettingsActivity`, `UserProfileActivity`, `ProfileSettingsActivity`, `FullScreenImageActivity`, `NotificationService`.
  - Subpaquetes:
    - `auth/` – `LoginActivity`, `RegisterActivity`, `SplashActivity`.
    - `adapters/` – `QuestionAdapter`, `CommentAdapter`.
    - `models/` – `Question`, `Comment`.
    - `utils/` – `RankUtils`.
- `app/src/main/res/layout/` – layouts XML de Activities, ítems de lista, diálogos y estados.
- `app/build.gradle` – dependencias (Material, Firebase, etc.) y configuración Android.

La navegación principal es **basada en Activities**, sin Fragments. `SplashActivity` actúa como gateway: revisa si hay sesión iniciada y envía al usuario a `MainActivity` o `LoginActivity`.

En las semanas 1–2 se cubren:

- Conceptos de `Activity` y su ciclo de vida (`onCreate` como punto principal de inicialización en casi todas las pantallas).
- Relación Activity ↔ layout ↔ responsabilidades.
- Navegación con Intents explícitos.

---

## 3. Diseño de UI y listas (resumen semanas 3–6)

En semanas 3–6 se construye la base visual y de interacción:

- **ConstraintLayout** (`activity_main.xml`, pantallas de login/registro): permite posicionar toolbar, contenedor principal, y botones flotantes.
- **Controles de entrada**: `TextInputLayout` + `TextInputEditText` para formularios (login, registro, crear pregunta), `Spinner` o componentes similares para categoría.
- **Eventos de clic**: `setOnClickListener` en botones, FAB, elementos de lista, íconos de menú contextual.
- **RecyclerView**:
  - `MainActivity` muestra el feed de preguntas con `RecyclerView + QuestionAdapter`.
  - `UserProfileActivity` muestra preguntas del usuario actual.
  - `QuestionDetailsActivity` muestra comentarios con `RecyclerView + CommentAdapter`.
- **Material Design**: `MaterialToolbar`, `FloatingActionButton`, `MaterialButton`, `ChipGroup`, `MaterialCardView` para presentar tarjetas de preguntas y comentarios.

Estas capas son el soporte visual sobre el que en semanas 9–14 se añaden hardware, GPS, servicios y tiempo real.

---

## 4. Semanas 9–10: Hardware, sonido y GPS

En estas semanas el foco es el acceso a recursos físicos del dispositivo y el uso de servicios de ubicación y sonido.

### 4.1. Permisos de hardware y ubicación

Toda interacción con cámara, almacenamiento, ubicación o notificaciones debe declararse en el manifiesto:

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
<uses-permission android:name="android.permission.ACCESS_WIFI_STATE" />
<uses-permission android:name="android.permission.CAMERA" />
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
```

**Por qué se colocan estos permisos**:

- **Cámara** y **almacenamiento**: para permitir que el usuario capture o seleccione imágenes (foto de perfil, imagen de pregunta) y que la app pueda leerlas/guardarlas.
- **Ubicación**: para asociar coordenadas a una pregunta y luego mostrarlas en un mapa.
- **Internet y estado de red**: Firebase necesita conexión; además se pueden mostrar mensajes específicos cuando no hay red.
- **Notificaciones** (Android 13+): algunas acciones pueden disparar notificaciones locales; el permiso es obligatorio a partir de esa versión.

> Imagen sugerida: editor del `AndroidManifest.xml` con el bloque `<uses-permission>` resaltado.

---

### 4.2. Manejo de GPS en `CreateQuestionActivity`

El objetivo es permitir que una pregunta tenga coordenadas geográficas (latitud y longitud). Para ello se usa el **Fused Location Provider** de Google Play Services.

Fragmentos clave en `CreateQuestionActivity`:

1. **Cliente de ubicación**:

```kotlin
private lateinit var fusedLocationClient: FusedLocationProviderClient
```

2. **Inicialización en `onCreate`**:

```kotlin
fusedLocationClient = LocationServices.getFusedLocationProviderClient(this)
```

3. **Comprobación de permisos**:

```kotlin
private fun checkLocationPermission() {
    if (ContextCompat.checkSelfPermission(
            this,
            Manifest.permission.ACCESS_FINE_LOCATION
        ) != PackageManager.PERMISSION_GRANTED
    ) {
        ActivityCompat.requestPermissions(
            this,
            arrayOf(Manifest.permission.ACCESS_FINE_LOCATION),
            LOCATION_PERMISSION_REQUEST_CODE
        )
    } else {
        getCurrentLocation()
    }
}
```

4. **Obtención de la última ubicación conocida**:

```kotlin
@SuppressLint("MissingPermission")
private fun getCurrentLocation() {
    fusedLocationClient.lastLocation
        .addOnSuccessListener { location ->
            if (location != null) {
                currentLatitude = location.latitude
                currentLongitude = location.longitude
                locationStatusText.text = getString(
                    R.string.location_selected,
                    currentLatitude,
                    currentLongitude
                )
            } else {
                locationStatusText.text = getString(R.string.location_not_available)
                Toast.makeText(this, R.string.location_not_available, Toast.LENGTH_SHORT).show()
            }
        }
        .addOnFailureListener {
            locationStatusText.text = getString(R.string.location_error)
            Toast.makeText(this, getString(R.string.location_error), Toast.LENGTH_SHORT).show()
        }
}
```

5. **Uso de las coordenadas al guardar la pregunta**:

```kotlin
val question = Question(
    // ...otros campos...
    latitude = currentLatitude,
    longitude = currentLongitude,
    imageUrl = imageBase64,
    likes = 0,
    dislikes = 0,
    answers = 0
)
```

**Por qué se implementa así**:

- Se usa `lastLocation` porque es rápido y suficiente para asociar una ubicación aproximada al momento de crear la pregunta (no hace falta un tracking continuo).
- Se encapsula la lógica de permisos en un método dedicado (`checkLocationPermission`) para mantener el código limpio y reutilizable.
- Las coordenadas se guardan dentro del modelo `Question`, lo que permite usarlas más tarde tanto en Firestore como en Realtime Database y en Google Maps.

> Imágenes sugeridas: pantalla de “Nueva pregunta” con el botón de ubicación y texto con las coordenadas; mensaje de error cuando la ubicación no está disponible.

---

### 4.3. Cámara y galería

Se cubren dos casos: **foto de perfil** y **imagen de pregunta**.

#### 4.3.1. Foto de perfil (`ProfileSettingsActivity`)

- **Por qué**: una foto de perfil mejora la identificación visual de los usuarios en el foro.
- **Cómo**:
  - Al hacer clic en la imagen de perfil, se muestra un diálogo para elegir entre “Tomar foto” (cámara) o “Elegir de la galería”.
  - Si el usuario elige galería, se abre un `Intent.ACTION_PICK` sobre `MediaStore.Images.Media.EXTERNAL_CONTENT_URI`.
  - Si elige cámara, se lanza `MediaStore.ACTION_IMAGE_CAPTURE` con un archivo temporal mediante `FileProvider`.
  - Tras recibir el resultado, se copia la imagen a un archivo local (`filesDir/profile_<uid>.jpg`) y se guarda la ruta en el documento del usuario en Firestore (`localImageUri`).

**Motivación técnica**:

- Guardar la imagen como archivo local evita problemas de permisos de contenido cuando la app se vuelve a abrir.
- `localImageUri` en Firestore permite a otras partes de la app (por ejemplo, el feed) conocer dónde está el archivo en el dispositivo y cargarlo con Glide.

#### 4.3.2. Imagen de pregunta (`CreateQuestionActivity`)

- Se usa `ActivityResultContracts.GetContent` para abrir un selector moderno de contenido (`image/*`).
- Al elegir una imagen, se muestra en un `ImageView` de preview.
- Antes de guardar la pregunta, la imagen se codifica a Base64 para almacenarla en Firestore (ver sección 6.2).

**Razón pedagógica**:

- `GetContent` simplifica el manejo de resultados comparado con `onActivityResult` clásico.
- Trabajar con URIs y codificación Base64 introduce al estudiante en el manejo de imágenes y optimización de tamaño.

---

### 4.4. Sonido de feedback

Para reforzar la UX, al publicar una pregunta se reproduce un **tono corto**.

En `CreateQuestionActivity`:

```kotlin
import android.media.AudioManager
import android.media.ToneGenerator
```

```kotlin
firestore.collection("questions")
    .add(question)
    .addOnSuccessListener { documentReference ->
        // ... lógica de éxito ...

        val toneG = ToneGenerator(AudioManager.STREAM_MUSIC, 100)
        toneG.startTone(ToneGenerator.TONE_PROP_BEEP, 150)
    }
```

**Por qué así y no con `MediaPlayer`**:

- `ToneGenerator` es inmediato, no requiere archivos en `res/raw` ni manejo de recursos externos.
- Es perfecto para un “beep” corto de confirmación.
- Deja espacio para proponer como ejercicio una versión más compleja con sonidos personalizados usando `MediaPlayer`.

---

## 5. Semana 11: Menús de opciones y Toolbars

Los menús son la puerta de entrada a acciones globales (perfil, ajustes, logout) y contextuales (bookmark, compartir, reportar).

### 5.1. Toolbar principal (`MainActivity`)

En `activity_main.xml` se define un `MaterialToolbar` que actúa como barra de acciones:

```xml
<com.google.android.material.appbar.MaterialToolbar
    android:id="@+id/toolbar"
    android:layout_width="match_parent"
    android:layout_height="?attr/actionBarSize"
    android:background="?attr/colorPrimary"
    app:title="@string/app_name"
    app:titleTextColor="?attr/colorOnPrimary"
    app:navigationIcon="@null"
    app:menu="@menu/main_menu" />
```

En Kotlin se conecta con el sistema de menús de `AppCompat`:

```kotlin
private fun setupToolbar() {
    setSupportActionBar(toolbar)
    supportActionBar?.title = "SYSFORUM"
    supportActionBar?.setDisplayHomeAsUpEnabled(false)
}
```

**Por qué se hace así**:

- `setSupportActionBar(toolbar)` permite usar los callbacks estándar `onCreateOptionsMenu` y `onOptionsItemSelected`.
- Se mantiene cohesión con el resto del framework de soporte.

---

### 5.2. Menú de opciones en `MainActivity`

Se infla el menú `main_menu` y se configura la barra de búsqueda (`SearchView`):

```kotlin
override fun onCreateOptionsMenu(menu: Menu?): Boolean {
    menuInflater.inflate(R.menu.main_menu, menu)

    val searchItem = menu?.findItem(R.id.action_search)
    val searchView = searchItem?.actionView as? SearchView

    searchView?.setOnQueryTextListener(object : SearchView.OnQueryTextListener {
        override fun onQueryTextSubmit(query: String?): Boolean {
            currentSearchQuery = query ?: ""
            filterQuestions()
            searchView.clearFocus()
            return true
        }

        override fun onQueryTextChange(newText: String?): Boolean {
            currentSearchQuery = newText ?: ""
            filterQuestions()
            return true
        }
    })

    return true
}
```

Y se gestionan las acciones de menú:

```kotlin
override fun onOptionsItemSelected(item: MenuItem): Boolean {
    return when (item.itemId) {
        R.id.action_profile -> {
            startActivity(Intent(this, ProfileSettingsActivity::class.java))
            true
        }
        R.id.action_settings -> {
            startActivity(Intent(this, SettingsActivity::class.java))
            true
        }
        R.id.action_about -> {
            Toast.makeText(this, "SYSFORUM v1.0", Toast.LENGTH_SHORT).show()
            true
        }
        R.id.action_logout -> {
            auth.signOut()
            startActivity(Intent(this, LoginActivity::class.java))
            finish()
            true
        }
        else -> super.onOptionsItemSelected(item)
    }
}
```

**Decisiones de diseño**:

- El menú concentra acciones globales de la app: perfil, configuración, información y cierre de sesión.
- La búsqueda forma parte del menú y se integra con el filtrado de preguntas (`filterQuestions()`), combinando texto y categoría.

---

### 5.3. Menú contextual en `QuestionDetailsActivity`

En la pantalla de detalle de pregunta el menú es específico de ese contexto:

- `bookmark`: guarda/retira la pregunta de los favoritos del usuario (`users/{uid}/bookmarks/{questionId}`).
- `share`: genera un `Intent.ACTION_SEND` con el contenido de la pregunta.
- `report`: muestra un diálogo para reportar contenido inapropiado.

Se usa el mismo patrón `onCreateOptionsMenu` + `onOptionsItemSelected`, pero la lógica ahora interactúa directamente con **Firestore** y el sistema de compartición de Android.

**Enfoque pedagógico**:

- Mostrar la diferencia entre menús globales (MainActivity) y menús contextuales (QuestionDetailsActivity).
- Introducir subcolecciones de Firestore (`bookmarks`) ligadas al usuario actual.

---

## 6. Semana 12: Servicios de Google, Firebase y Authentication

En esta etapa se consolida la integración con Firebase y servicios de Google.

### 6.1. Configuración en Gradle

En `app/build.gradle` se añaden el plugin y las dependencias:

```gradle
plugins {
    id 'com.android.application'
    id 'org.jetbrains.kotlin.android'
    id 'com.google.gms.google-services'
}

dependencies {
    implementation platform('com.google.firebase:firebase-bom:32.2.0')
    implementation 'com.google.firebase:firebase-auth-ktx'
    implementation 'com.google.firebase:firebase-firestore-ktx'
    implementation 'com.google.firebase:firebase-database-ktx'
    implementation 'com.google.firebase:firebase-storage-ktx'
    // ...otras dependencias...
}
```

**Por qué usar BOM y plugins**:

- El **BOM** (Bill of Materials) garantiza versiones compatibles entre los distintos módulos de Firebase.
- El plugin `com.google.gms.google-services` automatiza la lectura de `google-services.json` y configuración interna del SDK.

---

### 6.2. Autenticación con Firebase Auth

`LoginActivity` y `RegisterActivity` muestran cómo implementar un flujo clásico de email/contraseña.

- `FirebaseAuth.getInstance()` se inicializa en cada Activity.
- Login: `signInWithEmailAndPassword(email, password)`.
- Registro: `createUserWithEmailAndPassword(email, password)` + registro de datos adicionales en `users` (Firestore).

**Decisiones importantes**:

- Separar validación de campos (`validateFields`) de la llamada a Firebase, para mantener el código legible.
- Mapear los códigos de error de Firebase a mensajes claros al usuario.
- Almacenar metadatos del usuario (nombre, puntos, bio, rutas de imagen) en Firestore, no en Auth, para mayor flexibilidad.

---

### 6.3. Firestore como base de datos principal

Se usan tres colecciones principales:

- `users` – datos de perfil, puntos, uri de imagen, etc.
- `questions` – preguntas del foro.
- `comments` – comentarios asociados a preguntas.

La lectura se hace con **listeners en tiempo real** (`addSnapshotListener`), lo que permite que el feed y los comentarios se actualicen automáticamente sin refrescar la app.

**Motivo**:

- Firestore ofrece consultas con filtros y ordenamientos (por ejemplo, `orderBy("createdAt", DESCENDING)`), ajustándose bien al patrón de feed.

---

## 7. Semana 13: Google Maps, almacenamiento de imágenes y Cloud Storage

### 7.1. Google Maps vía Intents

En lugar de integrar un mapa embebido, SYSFORUM delega la visualización a la aplicación nativa de Google Maps:

```kotlin
viewLocationButton.setOnClickListener {
    currentQuestion?.let { question ->
        if (question.latitude != null && question.longitude != null) {
            val uri = Uri.parse("geo:${question.latitude},${question.longitude}?q=${question.latitude},${question.longitude}(${question.title})")
            val mapIntent = Intent(Intent.ACTION_VIEW, uri)
            mapIntent.setPackage("com.google.android.apps.maps")
            if (mapIntent.resolveActivity(packageManager) != null) {
                startActivity(mapIntent)
            } else {
                startActivity(Intent(Intent.ACTION_VIEW, uri))
            }
        }
    }
}
```

**Por qué usar Intents y no directamente el SDK de Maps**:

- Reduce complejidad y dependencias.
- Evita gestionar claves de API de Maps en el código.
- Es suficiente para el objetivo académico: demostrar cómo una app puede invocar otra app de mapas con parámetros.

Aun así, se deja abierta la extensión para usar `MapView`/`SupportMapFragment` en una actividad dedicada (`QuestionMapActivity`).

---

### 7.2. Estrategia actual de almacenamiento de imágenes

En lugar de usar directamente Cloud Storage, SYSFORUM opta por **codificar las imágenes a Base64** y almacenarlas como strings en Firestore.

#### 7.2.1. Codificación a Base64

```kotlin
private fun encodeImageToBase64(imageUri: Uri): String? {
    return try {
        val inputStream = contentResolver.openInputStream(imageUri)
        val bitmap = BitmapFactory.decodeStream(inputStream)
        val scaledBitmap = scaleBitmapDown(bitmap, 800)
        val outputStream = ByteArrayOutputStream()
        scaledBitmap.compress(Bitmap.CompressFormat.JPEG, 80, outputStream)
        val imageBytes = outputStream.toByteArray()
        Base64.encodeToString(imageBytes, Base64.NO_WRAP)
    } catch (e: Exception) {
        e.printStackTrace()
        null
    }
}
```

**Por qué se eligió Base64 inicialmente**:

- Evita la curva de aprendizaje adicional de Cloud Storage en fases tempranas.
- Permite trabajar con una única tecnología (Firestore) para datos y archivos.
- Hace visible una limitación real: el tamaño máximo de documentos en Firestore, lo que obliga a comprimir y escalar las imágenes.

#### 7.2.2. Lectura de la imagen

En `QuestionDetailsActivity` se detecta si `imageUrl` es una URL HTTP o Base64 y se carga con Glide en consecuencia.

**Ventajas/desventajas**:

- Ventaja: simplicidad inicial.
- Desventaja: documentos más grandes, potencialmente más costosos de transferir y almacenar; no es la estrategia recomendada en producción.

---

### 7.3. Cloud Storage como siguiente paso natural

Aunque el código ya integra `firebase-storage-ktx` y referencias a `FirebaseStorage`, el flujo principal actual no sube imágenes a Storage.

El diseño de la app deja, sin embargo, un camino claro para hacerlo:

1. Subir la imagen seleccionada a Storage (`/questions/{questionId}/image.jpg`).
2. Obtener la URL pública de descarga.
3. Guardar esa URL en Firestore dentro del documento de la pregunta.
4. Cargar siempre las imágenes usando Glide con esa URL.

Este planteamiento aparece en los comentarios y en el diseño de clases (`storageReference` en `CreateQuestionActivity`, `storage` en `ProfileSettingsActivity`), y se documenta como **mejora recomendada** para etapas posteriores.

---

## 8. Semana 14: Tiempo real con Firestore y Realtime Database

### 8.1. Tiempo real con Cloud Firestore

El comportamiento “en vivo” del feed y de los comentarios se basa en `addSnapshotListener`:

- `MainActivity.loadQuestions()` escucha cambios en la colección `questions`.
- `QuestionDetailsActivity.loadComments()` escucha cambios en `comments` filtrados por `questionId`.

**Por qué Firestore**:

- Ofrece consultas flexibles (filtros, ordenación) con actualizaciones en tiempo real.
- Se adapta bien al patrón de feed con filtros por categoría y búsqueda.

---

### 8.2. Realtime Database como espejo

Cada vez que se crea una pregunta o un comentario, se guarda también en **Firebase Realtime Database**:

```kotlin
val database = FirebaseDatabase.getInstance("https://sysforumuntels-default-rtdb.firebaseio.com")
val ref = database.getReference("questions")

val questionWithId = question.copy(id = questionId)

ref.child(questionId).setValue(questionWithId)
```

y, para comentarios:

```kotlin
val ref = database.getReference("comments")
ref.child(commentId).setValue(comment)
```

Actualmente la app **no lee** desde RTDB, pero este patrón tiene valor docente:

- Muestra cómo una misma entidad puede existir en Firestore y en RTDB.
- Permite comparar dos modelos de base de datos de Firebase:
  - Firestore (documentos/colecciones, consultas ricas).
  - RTDB (árbol JSON, muy rápido y simple para sincronización).

> Imagen sugerida: capturas de la consola de Firebase mostrando colecciones en Firestore y nodos equivalentes en Realtime Database.

---

### 8.3. Cloud Storage y tiempo real

Aunque Storage no es tiempo real, se combina conceptualmente con Firestore/RTDB:

- La subida de una imagen (perfil o pregunta) genera una URL.
- Esa URL se guarda en Firestore.
- Los listeners de Firestore hacen que la UI reaccione al cambio (mostrando la nueva imagen sin intervención manual del usuario).

Este flujo encaja con el patrón **reactivo** de la app: la base de datos informa a la UI cuando hay cambios relevantes.

---

## 9. Conclusión

SYSFORUM es más que un ejemplo de CRUD; es un caso de estudio completo que integra:

- **Fundamentos Android**: Activities, ciclo de vida, Intents, layouts, RecyclerView (semanas 1–6).
- **Diseño de interfaz moderna**: Material Design, Toolbars, FAB, CardView, ChipGroup.
- **Hardware y servicios del dispositivo**: cámara, almacenamiento, GPS, sonido (semanas 9–10).
- **Menús y navegación avanzada**: menús globales y contextuales integrados con la lógica de negocio (semana 11).
- **Backend como servicio**: Firebase Authentication, Firestore, Realtime Database, (preparación para Storage) (semana 12–14).
- **Integración con otras apps y servicios de Google**: Google Maps mediante Intents.

Las decisiones tomadas en el código (uso de Firestore para lecturas en tiempo real, RTDB como espejo, Base64 como solución inicial de imágenes, Intents para Maps, ToneGenerator para sonido) están guiadas tanto por objetivos pedagógicos como por simplicidad incremental:

1. Primero se introduce un concepto con la herramienta más directa (por ejemplo, Firestore + Base64).
2. Luego se señala el camino hacia una solución más profesional (Cloud Storage, Maps SDK, RTDB para flujos muy live).

Esta memoria, junto con los documentos `paso_a_paso.md` y `paso_a_paso_2.md`, conforma una documentación completa del proyecto, válida tanto como **memoria de curso** como guía de referencia para futuros desarrollos o mejoras sobre SYSFORUM.
