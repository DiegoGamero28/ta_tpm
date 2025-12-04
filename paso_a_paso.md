# SYSFORUM – Documentación paso a paso por semanas

Este documento describe, semana a semana, cómo se han ido implementando las funcionalidades principales de la aplicación **SYSFORUM**. Cada semana se apoya en código y pantallas reales del proyecto, y cuando un tema no está directamente implementado se proponen extensiones o ejercicios basados en la app.

> Nota: Las rutas de archivos se indican relativas al módulo `app` salvo que se especifique lo contrario.

---

## Semana 1: Estructura de proyecto Android

### 1.1. Estructura general de carpetas

En esta semana se revisa la estructura base de un proyecto Android usando SYSFORUM como ejemplo real.

Estructura principal del proyecto:

- `build.gradle`, `settings.gradle`, `gradle.properties`, `local.properties` – configuración de Gradle y del proyecto.
- Módulo principal: `app/`
  - `app/build.gradle` – configuración específica del módulo Android (dependencias de Firebase, Material Design, etc.).
  - `app/src/main/AndroidManifest.xml` – manifiesto de la aplicación.
  - Código fuente Kotlin:
    - `app/src/main/java/com/sysforum/app/MainActivity.kt`
    - `app/src/main/java/com/sysforum/app/CreateQuestionActivity.kt`
    - `app/src/main/java/com/sysforum/app/QuestionDetailsActivity.kt`
    - `app/src/main/java/com/sysforum/app/UserProfileActivity.kt`
    - `app/src/main/java/com/sysforum/app/ProfileSettingsActivity.kt`
    - `app/src/main/java/com/sysforum/app/SettingsActivity.kt`
    - `app/src/main/java/com/sysforum/app/FullScreenImageActivity.kt`
    - `app/src/main/java/com/sysforum/app/NotificationService.kt`
    - `app/src/main/java/com/sysforum/app/auth/LoginActivity.kt`
    - `app/src/main/java/com/sysforum/app/auth/RegisterActivity.kt`
    - `app/src/main/java/com/sysforum/app/auth/SplashActivity.kt`
    - `app/src/main/java/com/sysforum/app/adapters/QuestionAdapter.kt`
    - `app/src/main/java/com/sysforum/app/adapters/CommentAdapter.kt`
    - `app/src/main/java/com/sysforum/app/models/Question.kt`
    - `app/src/main/java/com/sysforum/app/models/Comment.kt`
    - `app/src/main/java/com/sysforum/app/utils/RankUtils.kt`
  - Recursos de interfaz:
    - Layouts: `app/src/main/res/layout/*.xml` (Activities, ítems de lista, diálogos, estados vacíos/carga…)
    - Valores: `app/src/main/res/values/*.xml` (cadenas, colores, estilos…)

**Imagen sugerida**: captura de Android Studio mostrando el árbol de archivos del módulo `app`, resaltando las carpetas `java/` y `res/`.

---

### 1.2. Manifiesto y puntos de entrada

El archivo `app/src/main/AndroidManifest.xml` define los componentes principales de SYSFORUM:

- **Permisos declarados**:
  - `INTERNET`, `ACCESS_NETWORK_STATE`, `ACCESS_WIFI_STATE` – necesarios para conectarse a Firebase y comprobar conectividad.
  - `CAMERA`, `READ_EXTERNAL_STORAGE`, `WRITE_EXTERNAL_STORAGE` – para manejo de imágenes de perfil y de preguntas.
  - `ACCESS_FINE_LOCATION`, `ACCESS_COARSE_LOCATION` – para guardar/mostrar ubicación asociada a una pregunta.
  - `POST_NOTIFICATIONS` – para mostrar notificaciones (Android 13+).
- **Application**:
  - `android:theme="@style/Theme.SYSFORUM"` – tema principal de la app.
- **Activities declaradas**:
  - `.auth.SplashActivity` – *launcher* (pantalla inicial). Tiene el `intent-filter` con `MAIN` y `LAUNCHER`.
  - `.auth.LoginActivity`, `.auth.RegisterActivity` – flujo de autenticación.
  - `.MainActivity` – pantalla principal con el feed de preguntas.
  - `.CreateQuestionActivity` – crear nueva pregunta.
  - `.QuestionDetailsActivity` – detalle de una pregunta.
  - `.UserProfileActivity` – perfil público de usuario.
  - `.ProfileSettingsActivity` – configuración de perfil propio.
  - `.SettingsActivity` – ajustes de la app.
  - `.FullScreenImageActivity` – visualización de imágenes a pantalla completa.
- **Service**:
  - `.NotificationService` – servicio para manejar notificaciones en segundo plano.

**Imagen sugerida**: diagrama de componentes donde se vea `SplashActivity` como punto de entrada y las demás Activities conectadas por flechas.

---

### 1.3. Mapeo de Activities, layouts y responsabilidades

A continuación se mapea cada Activity principal con su layout y su responsabilidad en la app:

| Activity                      | Layout                          | Responsabilidad principal                           |
|------------------------------|---------------------------------|-----------------------------------------------------|
| `SplashActivity`             | `activity_splash.xml`          | Pantalla de bienvenida y comprobación de sesión.   |
| `LoginActivity`              | `activity_login.xml`           | Inicio de sesión con correo y contraseña.          |
| `RegisterActivity`           | `activity_register.xml`        | Registro de nuevo usuario y validación de datos.   |
| `MainActivity`               | `activity_main.xml`            | Lista de preguntas (feed), filtros, FAB.           |
| `CreateQuestionActivity`     | `activity_create_question.xml` | Formulario de creación de pregunta.                |
| `QuestionDetailsActivity`    | `activity_question_details.xml`| Detalle de pregunta, likes/dislikes, comentarios.  |
| `UserProfileActivity`        | `activity_user_profile.xml`    | Perfil público y lista de preguntas del usuario.   |
| `ProfileSettingsActivity`    | `activity_profile_settings.xml`| Edición de datos de perfil.                        |
| `SettingsActivity`           | `activity_settings.xml`        | Preferencias de notificaciones, tema, cuenta.      |
| `FullScreenImageActivity`    | `activity_full_screen_image.xml`| Imagen de pregunta a pantalla completa.          |

**Imagen sugerida**: “mapa de navegación” con las pantallas como cajas y flechas indicando la ruta típica:

- `SplashActivity` → `LoginActivity` / `MainActivity`  
- `MainActivity` → `CreateQuestionActivity`, `QuestionDetailsActivity`, `UserProfileActivity`, `SettingsActivity`  
- `QuestionDetailsActivity` → `FullScreenImageActivity`  
- `UserProfileActivity` → `QuestionDetailsActivity`  

---

## Semana 2: Arquitectura para aplicaciones móviles. Activity. Ciclo de un Activity

### 2.1. Arquitectura general usada en SYSFORUM

SYSFORUM adopta una arquitectura centrada en **Activities** (sin Fragments) y servicios, apoyada en Firebase como backend.

Capas principales:

- **Capa de presentación (UI)**: Activities como `MainActivity`, `QuestionDetailsActivity`, `LoginActivity`, etc.
- **Capa de datos**:
  - `FirebaseAuth` – autenticación de usuarios.
  - `FirebaseFirestore` – base de datos principal de preguntas, usuarios, comentarios.
  - `FirebaseDatabase` (Realtime Database) – replicación de comentarios para fines demostrativos.
- **Modelos**:
  - `Question.kt` – representación de una pregunta.
  - `Comment.kt` – representación de un comentario.
- **Utilidades**:
  - `RankUtils.kt` – utilidades para calcular nombres y colores de rangos según puntos.
- **Servicio**:
  - `NotificationService.kt` – servicio que puede ser iniciado desde `MainActivity`.

**Imagen sugerida**: diagrama en capas donde las Activities consumen modelos y servicios de Firebase.

---

### 2.2. Ciclo de vida de un Activity con `MainActivity`

La clase `MainActivity` (`app/src/main/java/com/sysforum/app/MainActivity.kt`) es un buen ejemplo del ciclo de vida básico de un Activity.

En `onCreate` se realiza la mayor parte de la configuración:

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.activity_main)

    // 1. Inicializar Firebase
    auth = FirebaseAuth.getInstance()
    firestore = FirebaseFirestore.getInstance()

    // 2. Verificar si hay usuario autenticado
    val currentUser = auth.currentUser
    if (currentUser == null) {
        startActivity(Intent(this, LoginActivity::class.java))
        finish()
        return
    }

    // 3. Pedir permiso de notificaciones (Android 13+)
    if (android.os.Build.VERSION.SDK_INT >= android.os.Build.VERSION_CODES.TIRAMISU) {
        // requestPermissions(...)
    }

    // 4. Inicializar vistas y configurar UI
    initViews()
    setupRecyclerView()
    setupCategories()
    setupToolbar()
    setupListeners()

    // 5. Lógica de datos
    updateExistingProfileImages()
    loadQuestions()

    // 6. Iniciar servicio de notificaciones
    startService(Intent(this, NotificationService::class.java))
}
```

Puntos clave del ciclo de vida:

- `onCreate` – se infla el layout, se inicializan Firebase y las vistas, se configura la lógica inicial.
- `onStart` / `onResume` – no se sobreescriben explícitamente, pero aquí la Activity ya está visible y responde a eventos. El listener de Firestore registrado en `loadQuestions()` mantiene el feed actualizado mientras la Activity esté activa.
- `onPause` / `onStop` / `onDestroy` – podrían utilizarse para liberar listeners o detener servicios; en esta versión se mantiene una implementación mínima.

**Imagen sugerida**: diagrama de ciclo de vida de Activity, marcando `onCreate` como punto donde se inicializa casi todo en SYSFORUM.

---

### 2.3. Ciclo de vida de Activities de autenticación

`LoginActivity` (`app/src/main/java/com/sysforum/app/auth/LoginActivity.kt`) muestra un patrón muy típico:

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.activity_login)

    // Inicializar Firebase Auth
    auth = FirebaseAuth.getInstance()

    // Inicializar vistas
    initViews()

    // Configurar listeners
    setupListeners()
}
```

- `initViews()` usa `findViewById` para asociar controles de UI (`emailEditText`, `passwordEditText`, `loginButton`, `registerLink`).
- `setupListeners()` asigna eventos `setOnClickListener` a los botones.
- `loginUser()` encapsula la lógica de negocio: validación y llamada a `FirebaseAuth.signInWithEmailAndPassword`.

De forma similar, `RegisterActivity` define su propio `onCreate`, inicializa vistas y listeners, y luego contiene métodos específicos para la lógica de registro.

**Imagen sugerida**: pantalla de login con una flecha que señale cómo el clic en el botón dispara el método `loginUser()`.

---

### 2.4. Splash y navegación inicial

`SplashActivity` actúa como Activity de transición:

- En `onCreate` muestra un layout simple (`activity_splash.xml`) con el logo.
- Tras un breve retardo, comprueba si hay usuario autenticado y redirige:
  - Si hay sesión iniciada → `MainActivity`.
  - Si no → `LoginActivity`.

Esto ejemplifica cómo se puede usar el ciclo de vida de un Activity para gestionar la navegación inicial de la app.

**Imagen sugerida**: captura de la pantalla de splash con el logo de SYSFORUM.

---

## Semana 3: ConstraintLayout. Controles de entrada. Buttons. Checkboxes con evento clic. RadioButton

### 3.1. ConstraintLayout en pantallas principales

Muchos layouts de SYSFORUM utilizan **ConstraintLayout** para organizar la UI. Un ejemplo claro es `activity_main.xml`:

```xml
<androidx.constraintlayout.widget.ConstraintLayout
    ...
    tools:context=".MainActivity">

    <com.google.android.material.appbar.AppBarLayout
        android:id="@+id/appBarLayout"
        ...
        app:layout_constraintTop_toTopOf="parent">

        <com.google.android.material.appbar.MaterialToolbar
            android:id="@+id/toolbar"
            android:layout_width="match_parent"
            android:layout_height="?attr/actionBarSize"
            app:menu="@menu/main_menu" />

    </com.google.android.material.appbar.AppBarLayout>

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:orientation="vertical"
        app:layout_constraintTop_toBottomOf="@id/appBarLayout"
        app:layout_constraintBottom_toBottomOf="parent">
        <!-- Contenido principal -->
    </LinearLayout>

    <com.google.android.material.floatingactionbutton.FloatingActionButton
        android:id="@+id/addQuestionFab"
        ...
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toEndOf="parent" />

</androidx.constraintlayout.widget.ConstraintLayout>
```

Aquí se ve cómo ConstraintLayout permite:

- Fijar la `AppBarLayout` a la parte superior.
- Anclar el contenido principal (`LinearLayout`) entre la barra y la parte inferior.
- Posicionar el `FloatingActionButton` en la esquina inferior derecha con `layout_constraintBottom_toBottomOf` y `layout_constraintEnd_toEndOf`.

**Imagen sugerida**: vista de diseño de `activity_main.xml` donde se aprecian las líneas de constraints.

---

### 3.2. Controles de entrada y botones en Login y Registro

Las pantallas de autenticación (`activity_login.xml`, `activity_register.xml`) utilizan componentes de **Material Design** para los campos de entrada:

- `TextInputLayout` + `TextInputEditText` para email y contraseña.
- `MaterialButton` para las acciones principales.

En `LoginActivity`:

```kotlin
private fun initViews() {
    emailEditText = findViewById(R.id.emailEditText)
    passwordEditText = findViewById(R.id.passwordEditText)
    emailInputLayout = findViewById(R.id.emailInputLayout)
    passwordInputLayout = findViewById(R.id.passwordInputLayout)
    loginButton = findViewById(R.id.loginButton)
    registerLink = findViewById(R.id.registerLink)
}

private fun setupListeners() {
    loginButton.setOnClickListener {
        loginUser()
    }

    registerLink.setOnClickListener {
        startActivity(Intent(this, RegisterActivity::class.java))
    }
}
```

La validación de campos se hace con utilidades de Android:

```kotlin
private fun validateFields(email: String, password: String): Boolean {
    var isValid = true

    if (TextUtils.isEmpty(email)) {
        emailInputLayout.error = getString(R.string.error_email_required)
        isValid = false
    } else if (!Patterns.EMAIL_ADDRESS.matcher(email).matches()) {
        emailInputLayout.error = getString(R.string.error_invalid_email)
        isValid = false
    } else {
        emailInputLayout.error = null
    }

    if (TextUtils.isEmpty(password)) {
        passwordInputLayout.error = getString(R.string.error_password_required)
        isValid = false
    } else if (password.length < 6) {
        passwordInputLayout.error = getString(R.string.error_weak_password)
        isValid = false
    } else {
        passwordInputLayout.error = null
    }

    return isValid
}
```

**Imagen sugerida**: captura de la pantalla de login, resaltando campos y mensajes de error de `TextInputLayout`.

---

### 3.3. Formulario de creación de preguntas

`CreateQuestionActivity` (layout `activity_create_question.xml`) contiene:

- Campos de texto para título y descripción.
- Un `Spinner` o control similar para elegir la categoría de la pregunta.
- Botones para seleccionar/adjuntar imagen y ubicación.
- Un `MaterialButton` para publicar la pregunta.

En el código (a alto nivel):

- Se validan los campos requeridos.
- Se crea un objeto `Question`.
- Se guarda en Firestore y opcionalmente se asocia una imagen (URL o Base64) y una ubicación (latitud/longitud).

**Imagen sugerida**: captura de la pantalla de “Nueva pregunta” mostrando controles de entrada y botón de publicar.

---

### 3.4. CheckBox y RadioButton – propuesta didáctica

El proyecto SYSFORUM **no utiliza** actualmente `CheckBox` ni `RadioButton`, pero esta semana se puede complementar con un ejercicio práctico sobre la base de la app:

- **Ejemplo de CheckBox**:
  - Añadir en `activity_register.xml` un `CheckBox` con texto “Acepto los términos y condiciones”.
  - En `RegisterActivity`, antes de registrar, comprobar si el CheckBox está marcado; si no, mostrar un `Toast` indicando que debe aceptarlos.
- **Ejemplo de RadioButton**:
  - Añadir un `RadioGroup` con dos `RadioButton`: “Estudiante” y “Docente”.
  - Guardar este rol en el documento de usuario en Firestore y usarlo para mostrar un icono o color distinto en el perfil.

Pseudo-código de manejo de un CheckBox:

```kotlin
if (!termsCheckBox.isChecked) {
    Toast.makeText(this, "Debes aceptar los términos", Toast.LENGTH_SHORT).show()
    return
}
```

**Imagen sugerida**: mockup de la pantalla de registro con el CheckBox de términos y un grupo de RadioButtons para el rol.

---

## Semana 4: Evento Clic. Intents explícitos. ListView

### 4.1. Eventos de clic en SYSFORUM

Los eventos de clic (`setOnClickListener`) son la base de las interacciones en SYSFORUM.

Ejemplos en `MainActivity`:

```kotlin
private fun setupListeners() {
    addQuestionFab.setOnClickListener {
        startActivity(Intent(this, CreateQuestionActivity::class.java))
    }

    swipeRefreshLayout.setOnRefreshListener {
        loadQuestions()
    }
}
```

En el adaptador `QuestionAdapter`, se maneja el clic sobre cada pregunta:

```kotlin
holder.itemView.setOnClickListener {
    onQuestionClick(question)
}

holder.authorText.setOnClickListener {
    onAuthorClick?.invoke(question.authorId)
}

holder.moreOptionsButton.setOnClickListener { view ->
    val popup = PopupMenu(view.context, view)
    popup.menu.add("Eliminar")
    popup.setOnMenuItemClickListener { item ->
        if (item.title == "Eliminar") {
            onDeleteClick?.invoke(question)
            true
        } else false
    }
    popup.show()
}
```

En `LoginActivity` y `RegisterActivity`, los clics en botones y enlaces gestionan la navegación y las acciones de autenticación.

**Imagen sugerida**: captura del FAB de “Nueva pregunta” con una flecha mostrando el método `setupListeners()`.

---

### 4.2. Intents explícitos y navegación entre pantallas

SYSFORUM utiliza **Intents explícitos** para navegar de una Activity a otra:

- De `MainActivity` a `CreateQuestionActivity`:

```kotlin
startActivity(Intent(this, CreateQuestionActivity::class.java))
```

- De `MainActivity` a `QuestionDetailsActivity` pasando un objeto `Question` serializable:

```kotlin
private fun openQuestionDetails(question: Question) {
    val intent = Intent(this, QuestionDetailsActivity::class.java)
    intent.putExtra("question", question)
    startActivity(intent)
}
```

- De `SplashActivity` a `LoginActivity` o `MainActivity` según si hay usuario autenticado.
- De `QuestionDetailsActivity` a `FullScreenImageActivity` pasando la URL/imagen en Base64.
- De `QuestionDetailsActivity` a la aplicación de Google Maps con un Intent implícito `ACTION_VIEW` usando un `geo:` URI:

```kotlin
val uri = Uri.parse("geo:${question.latitude},${question.longitude}?q=${question.latitude},${question.longitude}(${question.title})")
val mapIntent = Intent(Intent.ACTION_VIEW, uri)
mapIntent.setPackage("com.google.android.apps.maps")
if (mapIntent.resolveActivity(packageManager) != null) {
    startActivity(mapIntent)
} else {
    startActivity(Intent(Intent.ACTION_VIEW, uri))
}
```

**Imagen sugerida**: diagrama de navegación indicando, para cada flecha, la llamada `startActivity(Intent(...))` asociada.

---

### 4.3. ListView vs RecyclerView

La aplicación SYSFORUM **no utiliza `ListView`**, sino `RecyclerView`, que es el patrón moderno recomendado para listas y feeds.

Para conectar con el tema de la semana se puede:

1. Explicar brevemente qué es `ListView` (lista de elementos simples, uso clásico con `ArrayAdapter`).
2. Mostrar cómo SYSFORUM implementa el concepto equivalente con `RecyclerView` y `QuestionAdapter` (esto se profundiza en la Semana 6).
3. Proponer un pequeño ejercicio teórico:
   - “Si en lugar de RecyclerView quisiéramos usar ListView para mostrar preguntas, podríamos crear un `ArrayAdapter<Question>` que infle un layout simple y lo asigne a un `ListView` en un layout alternativo.”

**Imagen sugerida**: comparativa visual (mockup) de una lista de preguntas mostrando la diferencia entre una lista simple tipo ListView y las tarjetas de RecyclerView.

---

## Semana 5: Fragment. Ciclo de Vida de un Fragment. Context Activity. Mensajes: Toast y SnackBar

### 5.1. Fragments – teoría aplicada al caso SYSFORUM

SYSFORUM, en su versión actual, **no utiliza `Fragment`** en el código de producción. Todas las pantallas se basan en Activities.

Sin embargo, sobre esta base se puede explicar:

- Qué es un Fragment y cómo se relaciona con una Activity.
- Cómo podríamos refactorizar la app para usar Fragments:
  - `MainActivity` podría contener un `HomeFragment` que muestre el feed de preguntas.
  - `QuestionDetailsActivity` podría convertirse en un `QuestionDetailsFragment` reutilizable en diferentes configuraciones (por ejemplo, modo tablet con dos paneles).

**Imagen sugerida**: diagrama que muestre una Activity conteniendo uno o varios Fragments, usando el contenido actual de `MainActivity` como ejemplo.

---

### 5.2. Context y uso de la Activity como contexto

El concepto de **Context** aparece constantemente en SYSFORUM:

- En Activities se usa `this` (que se refiere al Activity como contexto) para:
  - Crear `Toast`:

    ```kotlin
    Toast.makeText(this, "Mensaje", Toast.LENGTH_SHORT).show()
    ```

  - Crear diálogos:

    ```kotlin
    AlertDialog.Builder(this)
        .setTitle("Eliminar pregunta")
        ...
        .show()
    ```

  - Obtener servicios del sistema, como el teclado en `QuestionDetailsActivity`:

    ```kotlin
    val imm = getSystemService(Context.INPUT_METHOD_SERVICE) as InputMethodManager
    imm.showSoftInput(commentEditText, InputMethodManager.SHOW_IMPLICIT)
    ```

- En adaptadores (por ejemplo, `QuestionAdapter`) se usa el contexto de la vista:

  ```kotlin
  Glide.with(holder.itemView.context)
      .load(imageToLoad)
      .into(holder.questionImage)
  ```

  Aquí `holder.itemView.context` es el contexto asociado al View del RecyclerView.

**Imagen sugerida**: esquema con diferentes tipos de contexto (`Activity`, `Application`, `View.context`) y ejemplos concretos del código.

---

### 5.3. Mensajes al usuario: Toast en SYSFORUM

A lo largo de la app se usan `Toast` para informar al usuario de resultados o errores:

- En `LoginActivity`:
  - Al iniciar sesión correctamente: `Toast.makeText(this, getString(R.string.success_login), Toast.LENGTH_SHORT).show()`.
  - Al fallar el login: `Toast.makeText(this, getString(R.string.error_login_failed), Toast.LENGTH_SHORT).show()`.
- En `MainActivity`:
  - Al eliminar una pregunta: `Toast.makeText(this, "Pregunta eliminada", Toast.LENGTH_SHORT).show()`.
  - Al fallar la eliminación: muestra el mensaje de error.
- En `QuestionDetailsActivity`:
  - Al publicar un comentario.
  - Al fallar la operación de comentar.
- En `SettingsActivity`, `UserProfileActivity`, etc., para confirmar cambios o mostrar errores.

Ejemplo concreto al eliminar una pregunta en `MainActivity`:

```kotlin
firestore.collection("questions").document(question.id)
    .delete()
    .addOnSuccessListener {
        Toast.makeText(this, "Pregunta eliminada", Toast.LENGTH_SHORT).show()
        loadQuestions()
    }
    .addOnFailureListener { e ->
        Toast.makeText(this, "Error al eliminar: ${e.message}", Toast.LENGTH_SHORT).show()
    }
```

**Imagen sugerida**: mockup donde se ve un Toast en la parte inferior de la pantalla después de una acción.

---

### 5.4. SnackBar – propuesta de mejora

Aunque el proyecto no usa `SnackBar`, es un buen punto de mejora para seguir patrones de Material Design:

- **Caso de uso típico** en SYSFORUM:
  - Al eliminar una pregunta, se podría mostrar un `SnackBar` con la opción de “Deshacer”.
  - Al publicar una nueva pregunta, mostrar un `SnackBar` con la acción “Ver pregunta”.

Ejemplo de código propuesto (podría integrarse en `MainActivity`):

```kotlin
Snackbar.make(findViewById(R.id.rootLayout), "Pregunta eliminada", Snackbar.LENGTH_LONG)
    .setAction("Deshacer") {
        // Lógica para restaurar la pregunta eliminada
    }
    .show()
```

**Imagen sugerida**: mockup de la lista de preguntas con un SnackBar en la parte inferior ofreciendo “Deshacer”.

---

## Semana 6: RecyclerView. Material Design. CardView

### 6.1. RecyclerView para el feed de preguntas

El corazón de SYSFORUM es el feed de preguntas que se muestra en `MainActivity` usando un `RecyclerView`.

En `activity_main.xml`:

```xml
<androidx.swiperefreshlayout.widget.SwipeRefreshLayout
    android:id="@+id/swipeRefreshLayout"
    ...>

    <androidx.recyclerview.widget.RecyclerView
        android:id="@+id/questionsRecyclerView"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:padding="8dp"
        android:clipToPadding="false" />

</androidx.swiperefreshlayout.widget.SwipeRefreshLayout>
```

En `MainActivity.kt`, se configura el RecyclerView:

```kotlin
private fun setupRecyclerView() {
    val currentUserId = auth.currentUser?.uid ?: ""

    questionAdapter = QuestionAdapter(
        questions,
        currentUserId,
        onQuestionClick = { question -> openQuestionDetails(question) },
        onAuthorClick = { userId -> openUserProfile(userId) },
        onDeleteClick = { question -> deleteQuestion(question) }
    )

    questionsRecyclerView.apply {
        layoutManager = LinearLayoutManager(this@MainActivity)
        adapter = questionAdapter
    }
}
```

Los datos se cargan desde Firestore y se asignan a la lista:

```kotlin
firestore.collection("questions")
    .orderBy("createdAt", Query.Direction.DESCENDING)
    .addSnapshotListener { snapshot, exception ->
        if (exception != null) { /* manejar error */ return@addSnapshotListener }

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

**Imagen sugerida**: captura de la lista principal de preguntas con varias tarjetas visibles.

---

### 6.2. `QuestionAdapter` y el layout de ítem

El adaptador `QuestionAdapter` (`app/src/main/java/com/sysforum/app/adapters/QuestionAdapter.kt`) es responsable de unir los objetos `Question` con las vistas de cada item.

Estructura del ViewHolder:

```kotlin
class QuestionViewHolder(itemView: View) : RecyclerView.ViewHolder(itemView) {
    val titleText: TextView = itemView.findViewById(R.id.questionTitle)
    val descriptionText: TextView = itemView.findViewById(R.id.questionDescription)
    val authorText: TextView = itemView.findViewById(R.id.questionAuthor)
    val dateText: TextView = itemView.findViewById(R.id.questionDate)
    val likesText: TextView = itemView.findViewById(R.id.questionLikes)
    val dislikesText: TextView = itemView.findViewById(R.id.questionDislikes)
    val answersText: TextView = itemView.findViewById(R.id.questionAnswers)
    val authorProfileImage: ImageView = itemView.findViewById(R.id.authorProfileImage)
    val questionImage: ImageView = itemView.findViewById(R.id.questionImage)
    val moreOptionsButton: ImageView = itemView.findViewById(R.id.moreOptionsButton)
}
```

En `onBindViewHolder` se asignan los datos:

```kotlin
override fun onBindViewHolder(holder: QuestionViewHolder, position: Int) {
    val question = questions[position]

    holder.titleText.text = question.title
    holder.descriptionText.text = question.description
    holder.authorText.text = question.authorName

    val dateFormat = SimpleDateFormat("dd/MM/yyyy HH:mm", Locale.getDefault())
    holder.dateText.text = dateFormat.format(question.createdAt)

    holder.likesText.text = "${question.likes}"
    holder.dislikesText.text = "${question.dislikes}"
    holder.answersText.text = "${question.answers}"

    // Cargar imagen de la pregunta (URL o Base64)
    if (!question.imageUrl.isNullOrEmpty()) {
        holder.questionImage.visibility = View.VISIBLE
        val imageToLoad = question.imageUrl
        try {
            if (imageToLoad.startsWith("http")) {
                Glide.with(holder.itemView.context)
                    .load(imageToLoad)
                    .fitCenter()
                    .into(holder.questionImage)
            } else {
                val decodedString = Base64.decode(imageToLoad, Base64.NO_WRAP)
                val decodedByte = BitmapFactory.decodeByteArray(decodedString, 0, decodedString.size)
                Glide.with(holder.itemView.context)
                    .load(decodedByte)
                    .fitCenter()
                    .into(holder.questionImage)
            }
        } catch (e: Exception) {
            holder.questionImage.visibility = View.GONE
        }
    } else {
        holder.questionImage.visibility = View.GONE
    }

    // Cargar imagen de perfil del autor
    loadAuthorProfileImage(holder.authorProfileImage, question.authorId)

    holder.itemView.setOnClickListener { onQuestionClick(question) }
    holder.authorText.setOnClickListener { onAuthorClick?.invoke(question.authorId) }
    holder.authorProfileImage.setOnClickListener { onAuthorClick?.invoke(question.authorId) }

    if (question.authorId == currentUserId) {
        holder.moreOptionsButton.visibility = View.VISIBLE
        holder.moreOptionsButton.setOnClickListener { /* mostrar PopupMenu para eliminar */ }
    } else {
        holder.moreOptionsButton.visibility = View.GONE
    }
}
```

El layout `item_question.xml` define la estructura visual de cada tarjeta (título, descripción, avatar, contadores, imagen…).

**Imagen sugerida**: captura de una tarjeta individual de pregunta resaltando cada campo (título, autor, contador de respuestas, etc.).

---

### 6.3. Comentarios y RecyclerView anidado

En `QuestionDetailsActivity`, los comentarios también se muestran en un `RecyclerView` con su propio adaptador `CommentAdapter` y layout `item_comment.xml`.

Configuración básica:

```kotlin
private fun setupRecyclerView() {
    commentAdapter = CommentAdapter(
        comments,
        onCommentLike = { comment -> toggleCommentLike(comment) },
        onReplyClick = { comment ->
            commentEditText.setText("@${comment.authorName} ")
            commentEditText.setSelection(commentEditText.text?.length ?: 0)
            commentEditText.requestFocus()
            val imm = getSystemService(Context.INPUT_METHOD_SERVICE) as InputMethodManager
            imm.showSoftInput(commentEditText, InputMethodManager.SHOW_IMPLICIT)
        }
    )

    commentsRecyclerView.apply {
        layoutManager = LinearLayoutManager(this@QuestionDetailsActivity)
        adapter = commentAdapter
    }
}
```

**Imagen sugerida**: pantalla de detalle de pregunta con la tarjeta de la pregunta arriba y la lista de comentarios debajo.

---

### 6.4. Material Design: Toolbars, FAB, Chips y CardView

SYSFORUM utiliza varios componentes de **Material Design**:

- `MaterialToolbar` – barra superior en Activities como `MainActivity`, `QuestionDetailsActivity`, `UserProfileActivity`, `SettingsActivity`.
- `FloatingActionButton` – botón de acción principal para crear nueva pregunta.
- `MaterialButton` – botones de acción en formularios y pantallas de detalle.
- `TextInputLayout` / `TextInputEditText` – campos de formulario con estilos Material.
- `ChipGroup` y `Chip` – filtro de categorías en `MainActivity`.
- `MaterialCardView` – tarjetas que envuelven el contenido de una pregunta o un comentario.

En `QuestionDetailsActivity`, por ejemplo, hay un `MaterialCardView` (`questionCard`) que envuelve el contenido principal de la pregunta.

**Imagen sugerida**: “pantalla principal anotada” donde se señale:

1. `MaterialToolbar` con el título SYSFORUM.
2. `ChipGroup` con chips de categoría.
3. `RecyclerView` con tarjetas de preguntas.
4. `FloatingActionButton` para añadir una nueva pregunta.

---

### 6.5. CardView / MaterialCardView y experiencia de usuario

El uso de tarjetas (CardView / MaterialCardView) contribuye a una UI limpia y legible:

- Cada pregunta parece una tarjeta independiente con:
  - Sombra/elevación.
  - Esquinas redondeadas.
  - Imagen destacada y texto organizado.
- Los comentarios siguen una estructura similar pero más compacta.

Esto sigue las guías de Material Design para **tarjetas de contenido**, mejorando la experiencia del usuario al navegar por el foro.

**Imagen sugerida**: comparativa donde se vea cómo se vería la lista sin CardView (solo texto plano) frente a la versión con tarjetas Material.

---

## Resumen

A lo largo de estas seis semanas, SYSFORUM sirve como hilo conductor para:

- Entender la estructura de un proyecto Android real y el rol del `AndroidManifest`.
- Trabajar con Activities, su ciclo de vida y navegación mediante Intents.
- Diseñar pantallas con ConstraintLayout y componentes de entrada modernos (Material Design).
- Manejar eventos de clic y navegar entre pantallas.
- Discutir cómo se integrarían Fragments y SnackBars como mejoras de arquitectura y UX.
- Implementar listas potentes con RecyclerView, adaptadores personalizados y tarjetas Material.

Este documento puede complementarse con capturas de pantalla reales del emulador/dispositivo y diagramas que ilustren las relaciones entre pantallas, componentes y servicios backend (Firebase).
