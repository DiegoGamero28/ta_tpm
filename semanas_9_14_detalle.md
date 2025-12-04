# SYSFORUM – Explicación sencilla de las semanas 9 a 14

En este documento se explican **solo** los contenidos relacionados con las semanas **9 a 14** del curso, usando el proyecto SYSFORUM como ejemplo.

La idea es que cualquier persona que lea esto, incluso sin ser experta en Android, entienda:

- Qué hace cada funcionalidad.
- Cómo está implementada a grandes rasgos.
- Sobre todo: **por qué** se hizo así y no de otra forma.

Temas cubiertos:

- Semana 9–10: Hardware (cámara, almacenamiento), sonido y GPS.
- Semana 11: Menú de opciones y Toolbars.
- Semana 12: Servicios de Google, Firebase y Authentication.
- Semana 13: Google Maps y almacenamiento de imágenes (Cloud Storage como idea futura).
- Semana 14: Tiempo real con Firestore y Realtime Database.

---

## Semana 9 y 10 – Hardware, sonido y GPS

En estas semanas se busca aprovechar capacidades **físicas** del dispositivo (cámara, ubicación, sonido) para enriquecer la app.

### 1. Permisos: la puerta de entrada al hardware

Antes de usar algo como la cámara o el GPS, Android obliga a pedir **permiso**. Esto protege la privacidad del usuario.

En `AndroidManifest.xml` se declaran los permisos:

```xml
<uses-permission android:name="android.permission.CAMERA" />
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
```

**¿Por qué se ponen estos permisos?**

- **CAMERA**: porque el usuario puede tomar una foto de perfil con la cámara.
- **READ/WRITE EXTERNAL STORAGE**: porque puede elegir imágenes desde la galería, y la app necesita leerlas y copiarlas.
- **ACCESS_FINE/COARSE_LOCATION**: porque al crear una pregunta, el usuario puede asociar su ubicación.
- **POST_NOTIFICATIONS**: porque en versiones recientes de Android las notificaciones también necesitan permiso explícito.

> Si estos permisos no se declaran, al intentar usar la cámara o la ubicación, la app fallaría o Android lo bloquearía.

Además de declararlos en el manifest, algunos (como ubicación o notificaciones) hay que pedirlos **en tiempo de ejecución** con `requestPermissions`, para que el usuario acepte o rechace.

---

### 2. GPS: guardar dónde se hizo la pregunta

En SYSFORUM, cuando creas una pregunta puedes guardar **dónde** la hiciste (latitud y longitud). Esto luego sirve para abrir un mapa en el detalle de la pregunta.

Todo esto ocurre en `CreateQuestionActivity`.

#### 2.1. ¿Qué herramienta se usa?

Se usa el **Fused Location Provider** (`FusedLocationProviderClient`) de Google Play Services. Es la forma recomendada de obtener la ubicación, porque combina GPS, WiFi y redes móviles de manera eficiente.

```kotlin
private lateinit var fusedLocationClient: FusedLocationProviderClient
```

Se inicializa en `onCreate`:

```kotlin
fusedLocationClient = LocationServices.getFusedLocationProviderClient(this)
```

#### 2.2. Flujo básico para obtener la ubicación

1. El usuario pulsa un botón tipo "Obtener ubicación".
2. Se llama a un método que verifica permisos:

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

3. Si el permiso ya está concedido, se llama a `getCurrentLocation()`:

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

#### 2.3. ¿Por qué se usa `lastLocation` y no actualizaciones constantes?

- `lastLocation` devuelve la última ubicación conocida del dispositivo.
- Para este caso (solo queremos saber dónde estaba el usuario cuando creó la pregunta) es **suficiente**.
- Pedir actualizaciones constantes de ubicación gastaría más batería y complicaría el código.

**Idea clave**: solo usamos lo que necesitamos. Si solo necesitamos una ubicación puntual, `lastLocation` es ideal.

#### 2.4. Guardar la ubicación en la pregunta

Cuando se construye el objeto `Question`, se añaden la latitud y longitud:

```kotlin
val question = Question(
    // ...
    latitude = currentLatitude,
    longitude = currentLongitude,
    // ...
)
```

Más adelante, en el detalle de la pregunta (`QuestionDetailsActivity`), estos campos permiten construir un Intent para abrir Google Maps (ver Semana 13).

---

### 3. Cámara y galería: imágenes de perfil y de pregunta

SYSFORUM usa imágenes en dos contextos:

1. Foto de perfil del usuario.
2. Imagen asociada a una pregunta.

La lógica es parecida en ambos casos: **dejar que el usuario elija o capture una imagen** y luego usarla dentro de la app.

#### 3.1. Foto de perfil en `ProfileSettingsActivity`

- Al pulsar la imagen de perfil, se abre un diálogo con dos opciones: “Tomar foto” (cámara) o “Elegir de la galería”.
- Si elige tomar foto, se lanza la cámara con un `Intent` (`MediaStore.ACTION_IMAGE_CAPTURE`).
- Si elige galería, se abre un selector de imágenes (`Intent.ACTION_PICK`).

Después, se recibe el resultado (URI de la imagen o bitmap) y se guarda en un archivo local dentro de la app, algo como `filesDir/profile_<uid>.jpg`. Ese **path** se guarda en Firestore en el documento del usuario (`localImageUri`).

**¿Por qué guardar una copia local?**

- Si solo usáramos la URI de contenido original, podríamos perder acceso si esa imagen se mueve o si cambian permisos.
- Copiar la imagen al almacenamiento interno de la app (`filesDir`) garantiza que la app seguirá teniendo acceso.
- Guardar la ruta en Firestore permite que, si después queremos mostrar el avatar en el feed (`QuestionAdapter`), sabremos dónde buscarlo.

#### 3.2. Imagen de pregunta en `CreateQuestionActivity`

Aquí se usa un mecanismo más moderno: `ActivityResultContracts.GetContent`.

- Se configura un launcher que abre un selector de archivos de tipo `image/*`.
- Cuando el usuario selecciona una imagen, recibimos su `Uri` y la mostramos en un `ImageView` de preview.
- Antes de guardar la pregunta, se convierte la imagen a una cadena Base64 (ver Semana 13 para más detalle).

**¿Por qué `GetContent`?**

- Es la forma recomendada con la API de resultados de actividad moderna, más segura y simple que el viejo `onActivityResult`.
- Encaja bien con el flujo de “solo seleccionar” (no necesitamos permisos especiales de cámara si solo usamos galería).

---

### 4. Sonido: pequeño beep de confirmación

Cuando el usuario publica una pregunta con éxito, la app reproduce un pequeño **beep**.

Se usa la clase `ToneGenerator`:

```kotlin
val toneG = ToneGenerator(AudioManager.STREAM_MUSIC, 100)
toneG.startTone(ToneGenerator.TONE_PROP_BEEP, 150)
```

**¿Por qué este enfoque tan simple?**

- No hace falta crear un archivo de sonido ni gestionar un `MediaPlayer`.
- `ToneGenerator` ya trae tonos estándar; ideal para un “feedback rápido”.
- Pedagógicamente es una forma ligera de introducir audio sin complicar el proyecto.

Más adelante, si se quisiera, se podría mejorar usando:

- Un archivo `success.mp3` en `res/raw`.
- `MediaPlayer` para reproducirlo.

Pero para el objetivo de la semana, este beep es suficiente y fácil de mantener.

---

## Semana 11 – Menú de opciones y Toolbars

En esta semana la idea es ordenar bien las **acciones** de la app: que el usuario sepa dónde ir para ver su perfil, cambiar ajustes, cerrar sesión, etc.

### 1. Toolbar: la barra superior

En casi todas las pantallas hay una barra superior (`MaterialToolbar`) que muestra:

- El título de la pantalla (por ejemplo, "SYSFORUM" o "Detalles de la pregunta").
- Iconos de menú (buscar, ajustes, etc.).

En `activity_main.xml` se define así:

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

En el código Kotlin (`MainActivity`):

```kotlin
private fun setupToolbar() {
    setSupportActionBar(toolbar)
    supportActionBar?.title = "SYSFORUM"
    supportActionBar?.setDisplayHomeAsUpEnabled(false)
}
```

**¿Por qué hacer `setSupportActionBar`?**

- Para que Android trate esta toolbar como la barra oficial de la Activity y podamos usar los métodos estándar `onCreateOptionsMenu` y `onOptionsItemSelected`.

---

### 2. Menú principal en `MainActivity`

El menú superior (`main_menu`) incluye acciones como:

- Buscar preguntas.
- Ir a configuración de perfil.
- Abrir ajustes generales.
- Ver información de la app.
- Cerrar sesión.

En `onCreateOptionsMenu` se infla el menú y se configura la búsqueda:

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

**¿Por qué usar `SearchView` aquí?**

- Porque la búsqueda afecta **a toda la lista** de preguntas.
- Es lógico que esté en la barra superior, como en muchas apps (Gmail, Play Store, etc.).

En `onOptionsItemSelected` se manejan las demás acciones:

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

**¿Por qué estas opciones están en el menú y no como botones en la pantalla?**

- Son acciones **globales** de la app (no dependen de una sola pregunta).
- El usuario ya está acostumbrado a encontrarlas en el menú de la barra superior.
- Evita llenar la UI de botones y mantiene la pantalla del feed más limpia.

---

### 3. Menú específico en el detalle de pregunta

En `QuestionDetailsActivity` hay un menú diferente, con acciones relacionadas **solo con esa pregunta**:

- Guardar como favorito (bookmark).
- Compartir.
- Reportar.

Esto se hace con otro archivo de menú (`menu_question_details`) y se maneja en `onOptionsItemSelected` de esa Activity.

**Idea importante**:

- Usar menús diferentes según la pantalla refuerza el concepto de **acciones globales vs acciones contextuales**.

---

## Semana 12 – Servicios de Google, Firebase y Authentication

Aquí el enfoque está en conectar la app con servicios externos (Firebase) para no tener que programar todo el backend desde cero.

### 1. Configuración de Firebase en Gradle

En `app/build.gradle` se declara el uso de Firebase:

```gradle
plugins {
    id 'com.android.application'
    id 'org.jetbrains.kotlin.android'
    id 'com.google.gms.google-services'
}

// ...

dependencies {
    implementation platform('com.google.firebase:firebase-bom:32.2.0')
    implementation 'com.google.firebase:firebase-auth-ktx'
    implementation 'com.google.firebase:firebase-firestore-ktx'
    implementation 'com.google.firebase:firebase-database-ktx'
    implementation 'com.google.firebase:firebase-storage-ktx'
}
```

**¿Por qué usar Firebase?**

- Permite tener autenticación, base de datos y almacenamiento en la nube **sin** montar un servidor propio.
- Ahorra tiempo de desarrollo.

**¿Y el BOM (`firebase-bom`)?**

- Garantiza que todas las librerías de Firebase usen versiones compatibles entre sí.

---

### 2. Autenticación de usuarios

SYSFORUM usa **Firebase Authentication** para el login y registro.

- `LoginActivity`: pide email y contraseña, valida, y llama a `signInWithEmailAndPassword`.
- `RegisterActivity`: valida campos, llama a `createUserWithEmailAndPassword` y luego guarda datos del usuario en la colección `users` de Firestore.

**¿Por qué separar Auth y Firestore?**

- Auth solo gestiona **identidad** (correo, UID, contraseña).
- Firestore guarda **información extra** del usuario (nombre, puntos, bio, foto, etc.).
- Esta separación da flexibilidad: se pueden añadir campos al usuario sin tocar el sistema de login.

Además, se controla la sesión:

- Si no hay usuario logueado (`auth.currentUser == null`), `MainActivity` redirige a `LoginActivity`.
- `SplashActivity` decide si ir directamente a `MainActivity` o pedir login.

---

### 3. Firestore: preguntas, comentarios y usuarios

SYSFORUM usa **Cloud Firestore** como base de datos principal.

Colecciones principales:

- `users`: datos de perfil y puntos.
- `questions`: información de cada pregunta.
- `comments`: comentarios asociados a preguntas.

Lectura en tiempo real:

- `MainActivity.loadQuestions()` usa `addSnapshotListener` para escuchar cambios en `questions`.
- `QuestionDetailsActivity.loadComments()` hace lo mismo con `comments`.

**¿Por qué Firestore y no solo Realtime Database?**

- Firestore permite consultas más ricas: filtros, orden por fecha, etc.
- El modelo de documentos/colecciones es muy cómodo para representar usuarios, preguntas y comentarios.

Realtime Database se reserva para otros usos (ver Semana 14).

---

## Semana 13 – Google Maps e imágenes (Cloud Storage)

En esta semana se conectan dos ideas:

- Ubicación + Google Maps.
- Imágenes + almacenamiento (primero Firestore con Base64, luego la idea de Storage).

### 1. Mostrar la ubicación con Google Maps

En el detalle de una pregunta (`QuestionDetailsActivity`), si la pregunta tiene latitud y longitud, se muestra un botón "Ver ubicación".

Al pulsarlo se crea un Intent para abrir Google Maps:

```kotlin
viewLocationButton.setOnClickListener {
    currentQuestion?.let { question ->
        if (question.latitude != null && question.longitude != null) {
            val uri = Uri.parse(
                "geo:${question.latitude},${question.longitude}?q=${question.latitude},${question.longitude}(${question.title})"
            )
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

**¿Por qué abrir la app de Maps y no usar un mapa dentro de SYSFORUM?**

- Es mucho más simple de implementar.
- No requiere gestionar claves de API de Maps ni un nuevo componente (`MapView`).
- Para mostrar una ubicación puntual, abrir Google Maps es suficiente y familiar para el usuario.

Más adelante, si se quisiera, se podría añadir una pantalla propia con `MapFragment`.

---

### 2. Imágenes de preguntas: Base64 primero, Storage después

Actualmente, las imágenes de preguntas se guardan de forma **simplificada**: se codifican como texto Base64 y se almacenan en Firestore dentro del campo `imageUrl`.

En `CreateQuestionActivity` se hace algo así:

```kotlin
val imageBase64 = selectedImageUri?.let { encodeImageToBase64(it) }

val question = Question(
    // ...
    imageUrl = imageBase64,
    // ...
)
```

**Ventajas de esta aproximación al principio**:

- No hace falta aprender Cloud Storage desde el primer día.
- Toda la información de la pregunta (texto + imagen) está en un solo documento.

**Inconvenientes**:

- Los documentos en Firestore se vuelven más pesados.
- Hay un límite de tamaño de documento; por eso se escala y comprime la imagen.

En `QuestionDetailsActivity`, para mostrarla, se comprueba si `imageUrl` es una URL HTTP o Base64, y se carga con Glide.

**Paso siguiente natural (Cloud Storage)**:

- Guardar las imágenes en Cloud Storage (por ejemplo, `/questions/{questionId}/image.jpg`).
- Guardar en Firestore solo la URL pública.
- Ganancia: documentos Firestore más ligeros, flujo más profesional.

SYSFORUM ya tiene las dependencias y referencias a `FirebaseStorage` preparadas, por lo que es fácil evolucionar hacia este modelo.

---

## Semana 14 – Tiempo real con Firestore y Realtime Database

Esta semana se centra en entender **lo que significa "tiempo real"** desde el punto de vista de la app y cómo Firebase ayuda a conseguirlo.

### 1. Tiempo real con Firestore

Cuando se usa `addSnapshotListener` en una colección, Firestore avisa a la app cada vez que cambia algo (alta, baja, modificación).

SYSFORUM lo usa en:

- `MainActivity` para la colección `questions`.
- `QuestionDetailsActivity` para la colección `comments`.

**¿Qué efecto ve el usuario?**

- Si una nueva pregunta se publica, aparece en la lista sin que el usuario tenga que recargar.
- Si alguien comenta una pregunta, el comentario aparece automáticamente en la pantalla de detalle.

**Ventaja pedagógica**: se ve claramente el concepto de "app conectada" donde los cambios del servidor se reflejan en la UI al instante.

---

### 2. Realtime Database como espejo de datos

Además de Firestore, SYSFORUM también guarda preguntas y comentarios en **Firebase Realtime Database** (RTDB), aunque la app no los lea desde allí todavía.

Ejemplo en `CreateQuestionActivity` (preguntas):

```kotlin
val database = FirebaseDatabase.getInstance("https://sysforumuntels-default-rtdb.firebaseio.com")
val ref = database.getReference("questions")

val questionWithId = question.copy(id = questionId)

ref.child(questionId).setValue(questionWithId)
```

Y en `QuestionDetailsActivity` (comentarios):

```kotlin
val ref = database.getReference("comments")
ref.child(commentId).setValue(comment)
```

**¿Por qué escribir también en RTDB si ya tenemos Firestore?**

- Para mostrar en la consola de Firebase otro modelo de base de datos (árbol JSON) con los mismos datos.
- Para tener la posibilidad de, en el futuro, usar RTDB para alguna funcionalidad muy "live" (por ejemplo, contadores de usuarios conectados, indicadores de "escribiendo...", etc.).

**Comparación simple**:

- **Firestore**: documentos dentro de colecciones, consultas sofisticadas, listeners en tiempo real.
- **Realtime Database**: árbol JSON, muy rápido y sencillo para sincronizar estados.

Ver ambos modelos en un mismo proyecto ayuda a entender mejor las diferencias.

---

### 3. Cloud Storage y tiempo real

Aunque Cloud Storage **no** envía actualizaciones en tiempo real como tal, se puede combinar con Firestore:

1. Subimos una imagen al Storage.
2. Guardamos la URL en Firestore.
3. Como la pantalla escucha Firestore, en cuanto cambie la URL, la imagen en la UI puede actualizarse.

Es decir, el "tiempo real" se consigue usando Firestore como "anunciador" de cambios, y Storage como "lugar donde realmente viven los archivos".

---

## Resumen final

En las semanas 9 a 14, SYSFORUM sirve para entender, de forma práctica y sencilla:

- Cómo pedir permisos y usar hardware del dispositivo (cámara, almacenamiento, GPS).
- Cómo dar feedback sonoro sin complicar el código.
- Cómo organizar acciones globales y contextuales con menús y toolbars.
- Cómo delegar la parte de backend a servicios como Firebase (Auth, Firestore, Realtime Database, Storage).
- Cómo aprovechar Google Maps sin necesidad de integrar el SDK completo.
- Cómo construir experiencias "en tiempo real" donde los cambios en la nube aparecen enseguida en la app.

Lo más importante es que **cada decisión tiene un porqué pedagógico**:

- Se empieza por soluciones sencillas (Base64 en Firestore, Intents a Maps, ToneGenerator).
- Se deja claro el camino para evolucionar a soluciones más profesionales (Cloud Storage, Maps SDK, RTDB para funcionalidades específicas).

De esta forma, SYSFORUM no solo es una app funcional, sino también una herramienta didáctica para entender paso a paso los temas avanzados del curso.
