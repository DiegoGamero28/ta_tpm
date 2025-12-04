# SYSFORUM – Documentación paso a paso por semanas (9–14)

Este documento describe, semana a semana, cómo se implementan en **SYSFORUM** las funcionalidades relacionadas con hardware, GPS, menús, servicios de Google/Firebase, mapas, almacenamiento de imágenes y tiempo real. Cada semana se apoya en código real del proyecto y, cuando un tema no está completamente implementado, se plantea como mejora o ejercicio basado en la app.

> Nota: Las rutas de archivos se indican relativas al módulo `app` salvo que se especifique lo contrario.

---

## Semana 9 y 10: Hardware, sonido y manejo de GPS

### 9–10.1. Permisos de hardware y ubicación

Para acceder a recursos de hardware (cámara, almacenamiento, ubicación, notificaciones) SYSFORUM declara los permisos necesarios en el manifiesto.

Archivo: `app/src/main/AndroidManifest.xml`

Permisos relevantes:

- Red y conectividad:
  - `android.permission.INTERNET`
  - `android.permission.ACCESS_NETWORK_STATE`
  - `android.permission.ACCESS_WIFI_STATE`
- Cámara y almacenamiento:
  - `android.permission.CAMERA`
  - `android.permission.READ_EXTERNAL_STORAGE`
  - `android.permission.WRITE_EXTERNAL_STORAGE`
- Ubicación (GPS):
  - `android.permission.ACCESS_FINE_LOCATION`
  - `android.permission.ACCESS_COARSE_LOCATION`
- Notificaciones (Android 13+):
  - `android.permission.POST_NOTIFICATIONS`

Estos permisos permiten:

- Capturar/seleccionar imágenes para perfil y preguntas.
- Obtener la ubicación actual del usuario al crear una pregunta.
- Mostrar notificaciones (a través de `NotificationService`).

**Imagen sugerida**: captura del editor de `AndroidManifest.xml` en Android Studio, resaltando el bloque `<uses-permission>`.

---

### 9–10.2. Manejo de GPS y ubicación al crear una pregunta

SYSFORUM permite asociar una ubicación (latitud/longitud) a una pregunta desde la pantalla de creación.

Archivo principal: `app/src/main/java/com/sysforum/app/CreateQuestionActivity.kt`

Elementos clave:

- Cliente de ubicación de Google Play Services:

```kotlin
private lateinit var fusedLocationClient: FusedLocationProviderClient
```

- Inicialización del cliente en `onCreate`:

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.activity_create_question)

    fusedLocationClient = LocationServices.getFusedLocationProviderClient(this)

    initViews()
    setupToolbar()
    setupListeners()
}
```

- Comprobación de permisos de ubicación:

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

- Obtención de la última ubicación conocida:

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
        .addOnFailureListener { e ->
            locationStatusText.text = getString(R.string.location_error)
            Toast.makeText(this, getString(R.string.location_error), Toast.LENGTH_SHORT).show()
        }
}
```

- Uso de la ubicación al guardar la pregunta (`saveQuestion()`):

```kotlin
val question = Question(
    id = "",
    title = title,
    description = description,
    category = selectedCategory,
    authorId = currentUser.uid,
    authorName = currentUser.displayName ?: "Usuario",
    authorEmail = currentUser.email ?: "",
    createdAt = Date(),
    latitude = currentLatitude,
    longitude = currentLongitude,
    imageUrl = imageBase64,
    likes = 0,
    dislikes = 0,
    answers = 0
)
```

Flujo completo:

1. Usuario pulsa un botón tipo “Agregar ubicación” en `activity_create_question.xml`.
2. Se llama a `checkLocationPermission()`.
3. Si el permiso ya está concedido, se ejecuta `getCurrentLocation()`.
4. Si se obtiene ubicación, se guarda en `currentLatitude` y `currentLongitude`.
5. Al guardar la pregunta con `saveQuestion()`, se incluyen estos campos en el objeto `Question` guardado en Firestore (y replicado en Realtime Database).

**Imágenes sugeridas**:

- Captura de `CreateQuestionActivity` mostrando el botón de ubicación y el texto con las coordenadas seleccionadas.
- Captura del mensaje (“No se pudo obtener la ubicación”) cuando `location` es `null`.

---

### 9–10.3. Uso de cámara y galería para imágenes

SYSFORUM permite seleccionar o capturar imágenes para el **perfil de usuario** y para las **preguntas**.

#### 9–10.3.1. Imagen de perfil en `ProfileSettingsActivity`

Archivo: `app/src/main/java/com/sysforum/app/ProfileSettingsActivity.kt`

Elementos clave:

- Vistas y constantes:

```kotlin
private lateinit var profileImageView: ImageView

private val PICK_IMAGE_REQUEST = 1
private val CAMERA_REQUEST = 2
```

- Listener para cambiar la imagen de perfil (en `setupListeners()`):

```kotlin
profileImageView.setOnClickListener {
    showImagePickerDialog()
}
```

- Diálogo para elegir entre cámara y galería (`showImagePickerDialog()`):

```kotlin
private fun showImagePickerDialog() {
    val options = arrayOf("Tomar foto", "Elegir de la galería")

    AlertDialog.Builder(this)
        .setTitle("Seleccionar imagen de perfil")
        .setItems(options) { _, which ->
            when (which) {
                0 -> openCamera()
                1 -> openGallery()
            }
        }
        .show()
}
```

- Apertura de cámara y galería (métodos `openCamera()` y `openGallery()`):
  - **Galería**: `Intent(Intent.ACTION_PICK, MediaStore.Images.Media.EXTERNAL_CONTENT_URI)`.
  - **Cámara**: `Intent(MediaStore.ACTION_IMAGE_CAPTURE)` con un archivo temporal y `FileProvider` (según configuración de la app).

- Manejo del resultado en `onActivityResult(...)` y conversión a archivo local:

```kotlin
private fun convertContentUriToLocalFile(contentUri: Uri, userId: String) {
    val inputStream = contentResolver.openInputStream(contentUri)
    val fileName = "profile_${userId}.jpg"
    val file = File(filesDir, fileName)

    inputStream?.use { input ->
        file.outputStream().use { output ->
            input.copyTo(output)
        }
    }

    val userData = mapOf(
        "localImageUri" to file.absolutePath,
        "updatedAt" to System.currentTimeMillis()
    )

    firestore.collection("users")
        .document(userId)
        .update(userData)
}
```

**Imágenes sugeridas**:

- Pantalla de edición de perfil con la foto de perfil clickable.
- Diálogo con opciones “Tomar foto” y “Elegir de la galería”.
- Estructura de archivos en `filesDir` mostrando `profile_<uid>.jpg`.

#### 9–10.3.2. Imagen de pregunta en `CreateQuestionActivity`

Archivo: `CreateQuestionActivity.kt`

- Registro de resultado para selección de imagen de galería:

```kotlin
private val getContent = registerForActivityResult(ActivityResultContracts.GetContent()) { uri: Uri? ->
    if (uri != null) {
        selectedImageUri = uri
        imagePreview.setImageURI(uri)
        imagePreview.visibility = View.VISIBLE
    }
}
```

- Listener en `setupListeners()` para el botón de imagen:

```kotlin
addImageButton.setOnClickListener {
    getContent.launch("image/*")
}
```

- En el flujo de guardado (`saveQuestion()`), si hay imagen seleccionada, se codifica a Base64:

```kotlin
val imageBase64 = selectedImageUri?.let { encodeImageToBase64(it) }
```

**Imagen sugerida**: captura de la pantalla “Nueva pregunta” con una imagen ya seleccionada en el `imagePreview`.

---

### 9–10.4. Sonido de feedback al publicar una pregunta

Aunque la app no reproduce música compleja, utiliza una API sencilla de sonido para dar feedback cuando se publica una pregunta con éxito.

Archivo: `CreateQuestionActivity.kt`

- Importación de APIs de audio:

```kotlin
import android.media.AudioManager
import android.media.ToneGenerator
```

- En el `addOnSuccessListener` de la operación de guardado de pregunta en Firestore (dentro de `saveQuestion()`):

```kotlin
firestore.collection("questions")
    .add(question)
    .addOnSuccessListener { documentReference ->
        // ... lógica de éxito (RTDB, puntos, limpiar formulario, etc.) ...

        // Reproducir un tono corto de confirmación
        val toneG = ToneGenerator(AudioManager.STREAM_MUSIC, 100)
        toneG.startTone(ToneGenerator.TONE_PROP_BEEP, 150)
    }
```

Explicación:

- `ToneGenerator` se crea sobre el stream de música (`AudioManager.STREAM_MUSIC`).
- Se reproduce un tono (`TONE_PROP_BEEP`) durante 150 ms.
- Esto actúa como confirmación audible de que la operación fue exitosa.

**Propuesta de mejora didáctica**:

- Como ejercicio, se podría reemplazar el tono por un sonido propio usando `MediaPlayer`:
  - Añadir `res/raw/success.mp3`.
  - Crear un método `playSuccessSound()` que instancie un `MediaPlayer` desde ese recurso y lo reproduzca.

**Imágenes sugeridas**:

- Captura del formulario tras publicar (con Toast de éxito visible).
- Diagrama simple: botón “Publicar” → validación → Firestore/RTDB → aumento de puntos → sonido corto.

---

## Semana 11: Menú de opciones y Toolbars

### 11.1. Toolbars en Activities principales

SYSFORUM usa `MaterialToolbar` como barra de acciones en varias pantallas.

#### 11.1.1. Toolbar en `MainActivity`

Archivo: `app/src/main/java/com/sysforum/app/MainActivity.kt`

- Vista declarada en el layout `activity_main.xml`:

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

- En Kotlin, se asocia al `ActionBar` en `setupToolbar()`:

```kotlin
private fun setupToolbar() {
    setSupportActionBar(toolbar)
    supportActionBar?.title = "SYSFORUM"
    supportActionBar?.setDisplayHomeAsUpEnabled(false)
}
```

#### 11.1.2. Toolbars en otras Activities

Patrón similar se repite en:

- `CreateQuestionActivity.kt` → `setupToolbar()`.
- `QuestionDetailsActivity.kt` → `setupToolbar()` con botón de navegación atrás.
- `ProfileSettingsActivity.kt` → toolbar con título “Perfil”.
- `SettingsActivity.kt`, `UserProfileActivity.kt` → toolbars con título descriptivo y botón de back.

**Imagen sugerida**: captura de la pantalla principal donde se vea la `MaterialToolbar` con el título “SYSFORUM” y el menú de overflow a la derecha.

---

### 11.2. Menú de opciones en `MainActivity`

La pantalla principal tiene un menú superior (`main_menu`) que incluye opciones de búsqueda, perfil, ajustes, información y logout.

Archivo: `MainActivity.kt`

#### 11.2.1. Inflado del menú y configuración de SearchView

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

    // Restaurar estado de búsqueda si es necesario
    if (currentSearchQuery.isNotEmpty()) {
        searchItem?.expandActionView()
        searchView?.setQuery(currentSearchQuery, false)
        searchView?.clearFocus()
    }

    return true
}
```

- `menuInflater.inflate(R.menu.main_menu, menu)` asocia el XML del menú a la toolbar.
- Se localiza el ítem `action_search` y se obtiene su `SearchView`.
- Cada cambio en el texto actualiza `currentSearchQuery` y llama a `filterQuestions()` para filtrar la lista localmente.

#### 11.2.2. Manejo de selección de ítems de menú

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

Relación con la navegación:

- Menú → Perfil: `ProfileSettingsActivity`.
- Menú → Ajustes: `SettingsActivity`.
- Menú → Acerca de: muestra un `Toast` con la versión.
- Menú → Logout: cierra sesión en Firebase Auth y regresa a `LoginActivity`.

**Imágenes sugeridas**:

- Menú desplegado en la toolbar con cada opción marcada.
- Pantalla de ajustes (`SettingsActivity`) abierta desde el menú.

---

### 11.3. Menú de opciones en `QuestionDetailsActivity`

En el detalle de una pregunta existe un menú de opciones específico para esa pantalla (guardar, compartir, reportar).

Archivo: `app/src/main/java/com/sysforum/app/QuestionDetailsActivity.kt`

#### 11.3.1. Inflado del menú y comprobación de marcadores

```kotlin
override fun onCreateOptionsMenu(menu: Menu?): Boolean {
    menuInflater.inflate(R.menu.menu_question_details, menu)
    checkBookmarkStatus(menu)
    return true
}
```

El método `checkBookmarkStatus(menu)` consulta Firestore para saber si la pregunta actual está guardada (bookmark) por el usuario actual y actualiza el icono del menú en consecuencia.

#### 11.3.2. Selección de ítems del menú

```kotlin
override fun onOptionsItemSelected(item: MenuItem): Boolean {
    return when (item.itemId) {
        android.R.id.home -> {
            finish()
            true
        }
        R.id.action_bookmark -> {
            toggleBookmark(item)
            true
        }
        R.id.action_share -> {
            shareQuestion()
            true
        }
        R.id.action_report -> {
            showReportDialog()
            true
        }
        else -> super.onOptionsItemSelected(item)
    }
}
```

- `toggleBookmark(item)` guarda/elimina un documento en `users/{uid}/bookmarks/{questionId}` en Firestore.
- `shareQuestion()` crea un `Intent.ACTION_SEND` con el contenido de la pregunta.
- `showReportDialog()` muestra un diálogo para reportar la pregunta.

**Imágenes sugeridas**:

- Menú de detalle de pregunta mostrando íconos de “Guardar”, “Compartir” y “Reportar”.
- Captura del documento `bookmarks` en la subcolección del usuario en Firestore.

---

### 11.4. Búsqueda y filtrado desde el menú

El menú de búsqueda está fuertemente ligado al filtrado de la lista de preguntas.

Archivo: `MainActivity.kt`

Método de filtrado:

```kotlin
private fun filterQuestions() {
    questions.clear()

    val filtered = allQuestions.filter { question ->
        val matchesCategory = currentCategory == "Todas" || question.category == currentCategory
        val matchesSearch = currentSearchQuery.isEmpty() ||
            question.title.contains(currentSearchQuery, ignoreCase = true) ||
            question.description.contains(currentSearchQuery, ignoreCase = true)
        matchesCategory && matchesSearch
    }

    questions.addAll(filtered)
    questionAdapter.notifyDataSetChanged()

    if (questions.isEmpty()) {
        showEmptyState()
    } else {
        showQuestionsState()
    }
}
```

- `currentCategory` se controla con un `ChipGroup` (filtro por categoría).
- `currentSearchQuery` proviene de la `SearchView` del menú.
- La lista mostrada (`questions`) es una versión filtrada de `allQuestions`.

**Imagen sugerida**: secuencia de dos capturas, antes y después de escribir en el campo de búsqueda, mostrando cómo cambia la lista.

---

## Semana 12: Servicios de Google, Firebase y Authentication

### 12.1. Configuración de Firebase en el proyecto

SYSFORUM utiliza varios productos de Firebase:

- Authentication
- Cloud Firestore
- Realtime Database
- Cloud Storage (integración parcial)

#### 12.1.1. Configuración de Gradle

Archivo: `app/build.gradle`

Elementos clave:

- Plugin de Google Services:

```gradle
plugins {
    id 'com.android.application'
    id 'org.jetbrains.kotlin.android'
    id 'com.google.gms.google-services'
}
```

- Dependencias Firebase con BOM:

```gradle
implementation platform('com.google.firebase:firebase-bom:32.2.0')
implementation 'com.google.firebase:firebase-auth-ktx'
implementation 'com.google.firebase:firebase-firestore-ktx'
implementation 'com.google.firebase:firebase-database-ktx'
implementation 'com.google.firebase:firebase-storage-ktx'
```

- Archivo de configuración `google-services.json` ubicado en `app/`.

**Imágenes sugeridas**:

- Captura de la sección de dependencias Firebase en `build.gradle`.
- Captura de la consola de Firebase mostrando el proyecto vinculado.

---

### 12.2. Firebase Authentication: Login y Registro

#### 12.2.1. Inicio de sesión (`LoginActivity`)

Archivo: `app/src/main/java/com/sysforum/app/auth/LoginActivity.kt`

Elementos clave:

- Inicialización de Firebase Auth:

```kotlin
private lateinit var auth: FirebaseAuth

override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.activity_login)

    auth = FirebaseAuth.getInstance()
    initViews()
    setupListeners()
}
```

- Proceso de login:

```kotlin
private fun loginUser() {
    val email = emailEditText.text.toString().trim()
    val password = passwordEditText.text.toString().trim()

    if (!validateFields(email, password)) {
        return
    }

    loginButton.isEnabled = false
    loginButton.text = getString(R.string.loading)

    auth.signInWithEmailAndPassword(email, password)
        .addOnCompleteListener(this) { task ->
            loginButton.isEnabled = true
            loginButton.text = getString(R.string.login_button)

            if (task.isSuccessful) {
                Toast.makeText(this, getString(R.string.success_login), Toast.LENGTH_SHORT).show()
                startActivity(Intent(this, MainActivity::class.java))
                finish()
            } else {
                Toast.makeText(this, getString(R.string.error_login_failed), Toast.LENGTH_SHORT).show()
            }
        }
}
```

#### 12.2.2. Registro de usuario (`RegisterActivity`)

Archivo: `app/src/main/java/com/sysforum/app/auth/RegisterActivity.kt`

- Inicialización:

```kotlin
private lateinit var auth: FirebaseAuth
private lateinit var firestore: FirebaseFirestore

override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.activity_register)

    auth = FirebaseAuth.getInstance()
    firestore = FirebaseFirestore.getInstance()

    initViews()
    setupListeners()
}
```

- Proceso de registro (resumen):

```kotlin
private fun registerUser() {
    val name = nameEditText.text.toString().trim()
    val email = emailEditText.text.toString().trim()
    val password = passwordEditText.text.toString().trim()
    val confirmPassword = confirmPasswordEditText.text.toString().trim()

    if (!validateFields(name, email, password, confirmPassword)) {
        return
    }

    if (!isNetworkAvailable()) {
        Toast.makeText(this, R.string.error_no_internet, Toast.LENGTH_SHORT).show()
        return
    }

    registerButton.isEnabled = false
    registerButton.text = getString(R.string.loading)

    auth.createUserWithEmailAndPassword(email, password)
        .addOnCompleteListener(this) { task ->
            registerButton.isEnabled = true
            registerButton.text = getString(R.string.register_button)

            if (task.isSuccessful) {
                val user = auth.currentUser
                if (user != null) {
                    saveUserToFirestore(user, name)
                }
            } else {
                // Manejo de distintos códigos de error
                handleRegistrationError(task.exception)
            }
        }
}
```

#### 12.2.3. Control de sesión y navegación

- `SplashActivity` (no mostrado aquí en código) decide si navegar a `MainActivity` o `LoginActivity` según `FirebaseAuth.getInstance().currentUser`.
- `MainActivity` vuelve a `LoginActivity` si el usuario es `null`:

```kotlin
val currentUser = auth.currentUser
if (currentUser == null) {
    startActivity(Intent(this, LoginActivity::class.java))
    finish()
    return
}
```

**Imágenes sugeridas**:

- Pantallas de login y registro completas.
- Consola de Firebase → sección Authentication con uno o varios usuarios creados.

---

### 12.3. Firestore: preguntas, comentarios y usuarios

SYSFORUM usa **Cloud Firestore** como base de datos principal.

#### 12.3.1. Colección `questions`

- Creación de una pregunta en `CreateQuestionActivity` (método `saveQuestion()`):

```kotlin
firestore.collection("questions")
    .add(question)
    .addOnSuccessListener { documentReference ->
        val questionId = documentReference.id
        // Actualizar la pregunta con su ID y guardar también en RTDB
    }
```

- Lectura en tiempo real en `MainActivity` (`loadQuestions()`):

```kotlin
firestore.collection("questions")
    .orderBy("createdAt", Query.Direction.DESCENDING)
    .addSnapshotListener { snapshot, exception ->
        isLoading = false
        swipeRefreshLayout.isRefreshing = false

        if (exception != null) {
            showErrorState()
            return@addSnapshotListener
        }

        if (snapshot != null) {
            allQuestions.clear()
            val newQuestions = snapshot.documents.mapNotNull { document ->
                document.toObject(Question::class.java)?.copy(id = document.id)
            }
            allQuestions.addAll(newQuestions)
            filterQuestions()
        }
    }
```

#### 12.3.2. Colección `comments`

- Creación de comentario en `QuestionDetailsActivity` (`postComment()`):

```kotlin
val comment = Comment(
    questionId = question.id,
    content = content,
    authorId = currentUser.uid,
    authorName = currentUser.displayName ?: "Usuario",
    authorEmail = currentUser.email ?: "",
    createdAt = Date()
)

firestore.collection("comments")
    .add(comment)
    .addOnSuccessListener { documentReference ->
        // Limpia campo, Toast de éxito, RTDB, suma de puntos, etc.
    }
```

- Lectura en tiempo real de comentarios (`loadComments()`):

```kotlin
firestore.collection("comments")
    .whereEqualTo("questionId", question.id)
    .addSnapshotListener { snapshot, exception ->
        if (exception != null) {
            return@addSnapshotListener
        }

        if (snapshot != null) {
            comments.clear()
            val newComments = snapshot.documents.mapNotNull { document ->
                document.toObject(Comment::class.java)
            }.sortedBy { it.createdAt }
            comments.addAll(newComments)
            commentAdapter.notifyDataSetChanged()
        }
    }
```

#### 12.3.3. Colección `users`

- Guardado de usuario desde `RegisterActivity` (en `saveUserToFirestore(...)`).
- Actualización de perfil desde `ProfileSettingsActivity` (`saveProfile()` y `loadUserData()`).
- Suma de puntos por actividad (publicar pregunta y comentar) desde `CreateQuestionActivity` y `QuestionDetailsActivity`.

**Imágenes sugeridas**:

- Capturas de Firestore con las colecciones `users`, `questions`, `comments` visibles.
- Documento de usuario con campos de perfil y puntos.

---

## Semana 13: Google Maps, almacenamiento de imágenes y Cloud Storage

### 13.1. Google Maps: ver ubicación de una pregunta

SYSFORUM no integra un mapa embebido, pero utiliza **Intents** para abrir la app de Google Maps y mostrar la ubicación asociada a una pregunta.

Archivo: `QuestionDetailsActivity.kt`

Botón de ver ubicación en `setupListeners()`:

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

- Construye un URI `geo:` con coordenadas y el título de la pregunta como texto asociado.
- Intenta abrir específicamente la app de Google Maps; si no está disponible, delega en cualquier app que pueda manejar el intent.

**Imágenes sugeridas**:

- Detalle de pregunta con el botón “Ver ubicación” visible.
- Pantalla de Google Maps abierta mostrando el pin con el título de la pregunta.

**Propuesta didáctica** (no implementada, solo para el documento):

- Crear un `QuestionMapActivity` que use el SDK de **Google Maps** (`com.google.android.gms:play-services-maps`) y un `MapView` o `SupportMapFragment` para mostrar la ubicación directamente en la app.

---

### 13.2. Almacenamiento de imágenes en Firestore (Base64)

En la versión actual, SYSFORUM **no sube imágenes a Cloud Storage**, sino que opta por codificarlas en Base64 y almacenarlas directamente en Firestore.

#### 13.2.1. Codificación de imágenes en `CreateQuestionActivity`

Archivo: `CreateQuestionActivity.kt`

- Método para codificar la imagen seleccionada a Base64:

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

- Método de escalado de la imagen para reducir tamaño (`scaleBitmapDown`).
- Validación del tamaño (ejemplo): si el Base64 resultante supera cierto tamaño, se muestra un error y no se guarda.

#### 13.2.2. Uso de `imageUrl` en el modelo `Question`

Archivo: `Question.kt`

- Campo `imageUrl: String?` almacena ya sea una URL remota o un string Base64.

Lectura y mostrado en `QuestionDetailsActivity`:

```kotlin
if (!question.imageUrl.isNullOrEmpty()) {
    questionImage.visibility = View.VISIBLE
    val imageToLoad = question.imageUrl

    questionImage.setOnClickListener {
        val intent = Intent(this, FullScreenImageActivity::class.java)
        intent.putExtra("imageUrl", imageToLoad)
        startActivity(intent)
    }
    try {
        if (imageToLoad.startsWith("http")) {
            Glide.with(this).load(imageToLoad).into(questionImage)
        } else {
            val decodedString = Base64.decode(imageToLoad, Base64.DEFAULT)
            val decodedByte = BitmapFactory.decodeByteArray(decodedString, 0, decodedString.size)
            Glide.with(this).load(decodedByte).into(questionImage)
        }
    } catch (e: Exception) {
        e.printStackTrace()
        questionImage.visibility = View.GONE
    }
} else {
    questionImage.visibility = View.GONE
}
```

**Imágenes sugeridas**:

- Documento de Firestore de una pregunta con el campo `imageUrl` (Base64 largo).
- Pantalla de detalle de pregunta mostrando la imagen.

---

### 13.3. Integración parcial de Cloud Storage

El proyecto incluye la dependencia de Cloud Storage y campos preparados para usarlo, aunque el flujo principal actual usa Base64.

#### 13.3.1. Configuración

Archivo: `app/build.gradle`

```gradle
implementation 'com.google.firebase:firebase-storage-ktx'
```

Archivo: `CreateQuestionActivity.kt`

```kotlin
private lateinit var storageReference: StorageReference

override fun onCreate(savedInstanceState: Bundle?) {
    // ...
    storageReference = FirebaseStorage.getInstance().reference
}
```

Archivo: `ProfileSettingsActivity.kt`

```kotlin
private lateinit var storage: FirebaseStorage

override fun onCreate(savedInstanceState: Bundle?) {
    // ...
    storage = FirebaseStorage.getInstance()
}
```

En el código de `CreateQuestionActivity`, se deja explícito que por ahora no se suben imágenes a Storage, sino que se guarda Base64 en Firestore por simplicidad (esto se puede citar como comentario si está presente).

#### 13.3.2. Propuesta de uso de Cloud Storage (teórico)

Como parte de la documentación de esta semana, se puede plantear el flujo sugerido para **migrar a Cloud Storage**:

1. Al seleccionar una imagen en `CreateQuestionActivity`, en lugar de codificar a Base64:
   - Subir el archivo a Storage:

   ```kotlin
   val questionImageRef = storageReference.child("questions/$questionId/image.jpg")
   questionImageRef.putFile(selectedImageUri!!)
       .addOnSuccessListener { taskSnapshot ->
           questionImageRef.downloadUrl.addOnSuccessListener { uri ->
               val imageUrl = uri.toString()
               // Guardar imageUrl en Firestore en la pregunta
           }
       }
   ```

2. Guardar en Firestore solo la URL de descarga.
3. En `QuestionDetailsActivity` y `QuestionAdapter`, tratar siempre `imageUrl` como URL HTTP y cargarla con Glide.

**Imágenes sugeridas**:

- Mockup de la estructura de Storage en la consola de Firebase con carpetas `/questions` y `/users`.
- Pantalla de reglas de Storage mostrando acceso restringido a usuarios autenticados.

---

### 13.4. Relación Maps – Imágenes – Ubicación

Al final de esta semana, el alumno ve cómo se combinan varios elementos en una sola pantalla:

- Texto (título, descripción) de la pregunta.
- Imagen (Base64/URL renderizada con Glide).
- Ubicación (coordenadas) visualizada a través de Google Maps.

**Imagen sugerida**: montaje o captura anotada de `QuestionDetailsActivity` donde se ve:

1. Card con texto e imagen.
2. Botones de like/dislike/comentar.
3. Botón “Ver ubicación” que lleva a Maps.

---

## Semana 14: Tiempo real usando Realtime Database y Cloud Storage

### 14.1. Firestore en tiempo real: feed y comentarios

Aunque el título de la semana menciona Realtime Database, en SYSFORUM el comportamiento “en tiempo real” principal está implementado con **Cloud Firestore**.

#### 14.1.1. Feed de preguntas en tiempo real

Archivo: `MainActivity.kt`, método `loadQuestions()` (ya citado en Semana 12):

- Uso de `addSnapshotListener` sobre la colección `questions`.
- Cada cambio en la colección (alta, baja, actualización) dispara el listener y la UI se refresca.

**Imagen sugerida**: demostración (en vídeo o secuencia de capturas) de dos dispositivos donde uno crea una pregunta y ésta aparece sin refrescar en el otro.

#### 14.1.2. Comentarios en tiempo real

Archivo: `QuestionDetailsActivity.kt`, método `loadComments()` (ya citado):

- Listener `addSnapshotListener` sobre la colección `comments` filtrando por `questionId`.
- Al publicar un nuevo comentario, se actualiza la lista en pantalla automáticamente.

**Imagen sugerida**: captura antes y después de enviar un comentario, donde se vea cómo aparece al instante.

---

### 14.2. Realtime Database (RTDB) como espejo de datos

Además de Firestore, la app escribe en **Firebase Realtime Database** para ciertas colecciones, aunque actualmente no se leen desde allí en la UI.

#### 14.2.1. Espejo de preguntas en RTDB

Archivo: `CreateQuestionActivity.kt`, dentro de `saveQuestion()`:

```kotlin
val database = FirebaseDatabase.getInstance("https://sysforumuntels-default-rtdb.firebaseio.com")
val ref = database.getReference("questions")

val questionWithId = question.copy(id = questionId)

ref.child(questionId).setValue(questionWithId)
    .addOnSuccessListener {
        Log.d("CreateQuestion", "Question saved to RTDB")
    }
    .addOnFailureListener { e ->
        Log.e("CreateQuestion", "Error saving to RTDB", e)
    }
```

- Tras guardar la pregunta en Firestore y obtener su ID, se crea una entrada equivalente en Realtime Database bajo `questions/{questionId}`.

#### 14.2.2. Espejo de comentarios en RTDB

Archivo: `QuestionDetailsActivity.kt`, dentro de `postComment()`:

```kotlin
val database = FirebaseDatabase.getInstance("https://sysforumuntels-default-rtdb.firebaseio.com")
val ref = database.getReference("comments")
val commentId = documentReference.id

ref.child(commentId).setValue(comment)
    .addOnSuccessListener {
        Log.d("QuestionDetails", "Comment saved to RTDB")
    }
    .addOnFailureListener { e ->
        Log.e("QuestionDetails", "Error saving comment to RTDB", e)
    }
```

- Se guarda una copia del comentario bajo `comments/{commentId}`.

**Imágenes sugeridas**:

- Consola de Firebase → Realtime Database mostrando `questions` y `comments` con datos replicados.
- Comparativa visual entre la estructura de Firestore y la de RTDB para las mismas entidades.

---

### 14.3. Comparación Firestore vs Realtime Database en SYSFORUM

En la implementación actual:

- **Lecturas en tiempo real** para la UI se hacen con **Firestore**:
  - `MainActivity` (preguntas).
  - `QuestionDetailsActivity` (comentarios).
- **Escrituras duplicadas** se envían también a **Realtime Database**:
  - `CreateQuestionActivity` (preguntas).
  - `QuestionDetailsActivity` (comentarios).

Esto permite ilustrar en clase:

- Firestore como base de datos de documentos con consultas ricas y listeners por colección.
- RTDB como base de datos en árbol JSON, muy eficiente para ciertos flujos en tiempo real.

**Propuesta didáctica**:

- Definir un caso de uso real para RTDB, por ejemplo:
  - Un contador de usuarios conectados.
  - Un indicador “escribiendo…” en los comentarios.
- Implementar listeners en la app para leer directamente de RTDB, ilustrando las diferencias con Firestore.

**Imágenes sugeridas**:

- Tabla en el propio `.md` comparando características de Firestore y RTDB.
- Capturas de reglas de seguridad de Firestore vs. Realtime Database.

---

### 14.4. Cloud Storage en el contexto de tiempo real

Aunque Cloud Storage no es un servicio “tiempo real” en sí mismo, se puede combinar con Firestore y/o RTDB para efectos inmediatos en la UI:

- Flujo propuesto (basado en la arquitectura actual):
  1. Usuario cambia su foto de perfil.
  2. La app sube la nueva imagen a Cloud Storage (`/users/{uid}/profile.jpg`).
  3. Al finalizar la subida, se obtiene la URL de descarga y se guarda en el documento `users/{uid}` en Firestore.
  4. Gracias a que `MainActivity` y `QuestionAdapter` recargan datos de usuarios (o usan `updateExistingProfileImages()`), los nuevos avatares aparecen pronto en el feed.

Esto permite explicar en clase:

- Cómo combinar Storage (archivos binarios) + Firestore (metadatos y URLs) + listeners para actualizar la UI en cuanto cambian los datos.

**Imágenes sugeridas**:

- Captura de las reglas de Cloud Storage garantizando que solo usuarios autenticados pueden subir/leer ciertas rutas.
- Captura de Firestore mostrando la nueva URL de imagen de perfil.

---

### 14.5. Resumen arquitectónico de tiempo real en SYSFORUM

Para cerrar la semana, se puede presentar un diagrama general:

- **App Android**:
  - `MainActivity` escucha Firestore → lista de preguntas.
  - `QuestionDetailsActivity` escucha Firestore → lista de comentarios.
  - `CreateQuestionActivity` y `QuestionDetailsActivity` escriben en Firestore y RTDB.
  - `ProfileSettingsActivity` y `MainActivity` gestionan imágenes de perfil (local/Storage futuro).
- **Firebase**:
  - **Auth**: gestiona identidad de usuario.
  - **Firestore**: colecciones `users`, `questions`, `comments`, subcolección `bookmarks`.
  - **Realtime Database**: nodos `questions`, `comments` reflejando Firestore.
  - **Storage**: integrado a nivel de dependencias, con flujo propuesto para imágenes.

**Imagen sugerida**: diagrama de arquitectura donde se visualicen todas las flechas entre Activities ↔ Firestore ↔ RTDB ↔ Storage ↔ Servicios de Google (Location/Maps).

---

## Cierre general (Semanas 9–14)

En estas semanas, SYSFORUM se convierte en un caso de estudio completo para temas avanzados de desarrollo móvil:

- Uso de hardware (cámara, almacenamiento, GPS) con permisos en tiempo de ejecución.
- Feedback sonoro sencillo al completar acciones.
- Diseño de menús, toolbars y menús contextuales ligados a la navegación y a Firestore.
- Integración con Firebase Authentication, Firestore, Realtime Database y (de forma parcial) Cloud Storage.
- Interacción con Google Maps mediante Intents y propuesta de integración con el SDK de mapas.
- Gestión de datos en tiempo real (lectura con Firestore, escritura duplicada en RTDB) y reflexión sobre cuándo usar uno u otro.

El código de SYSFORUM, junto con las capturas y diagramas sugeridos, permite documentar y evaluar cada tema con ejemplos concretos y extensibles a nuevos escenarios.
