# SYSFORUM – Explicación sencilla de las semanas 1 a 6

En este documento se explican **solo** los contenidos relacionados con las semanas **1 a 6** del curso, usando el proyecto SYSFORUM como ejemplo.

El objetivo es que cualquiera pueda entender, sin ser experto en Android:

- Cómo está organizado el proyecto.
- Qué es una Activity y cómo se usa su ciclo de vida.
- Cómo se diseñan pantallas con layouts y controles.
- Cómo se manejan eventos de clic, navegación y listas.
- Qué papel juegan RecyclerView, Material Design y (a nivel conceptual) los Fragments.

A lo largo del documento se insiste en el **por qué** de cada decisión.

---

## Semana 1 – Estructura de un proyecto Android

La primera semana se centra en conocer el “esqueleto” de una app Android real.

En SYSFORUM, el módulo principal es `app/`. Dentro de ese módulo tenemos varias piezas importantes:

### 1.1. `AndroidManifest.xml`: el "DNI" de la app

Archivo: `app/src/main/AndroidManifest.xml`

Aquí se declaran:

- El nombre del paquete.
- Todas las Activities (pantallas) de la app.
- El punto de entrada (la Activity que se abre primero).
- Los permisos que requiere la app (Internet, cámara, ubicación, etc.).
- El tema visual por defecto (`Theme.SYSFORUM`).

**¿Por qué es tan importante el manifest?**

Porque Android lo lee **antes** de ejecutar nada. Si una Activity no está declarada aquí, no se puede abrir. Si falta un permiso, aunque el código esté correcto, Android bloqueará el acceso al recurso (cámara, ubicación…).

En SYSFORUM, por ejemplo, `SplashActivity` se marca como la Activity de arranque:

```xml
<activity
    android:name=".auth.SplashActivity"
    android:exported="true"
    android:theme="@style/Theme.SYSFORUM.Splash">
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
</activity>
```

Esto le dice al sistema: “cuando el usuario toque el icono de la app, abre `SplashActivity`”.

---

### 1.2. Paquetes de código: separar responsabilidades

En `app/src/main/java/com/sysforum/app/` el código Kotlin se organiza en paquetes:

- `com.sysforum.app` – Activities principales y servicio:
  - `MainActivity`, `CreateQuestionActivity`, `QuestionDetailsActivity`, `UserProfileActivity`, `ProfileSettingsActivity`, `SettingsActivity`, `FullScreenImageActivity`, `NotificationService`.
- `com.sysforum.app.auth` – autenticación:
  - `SplashActivity`, `LoginActivity`, `RegisterActivity`.
- `com.sysforum.app.adapters` – adaptadores de listas:
  - `QuestionAdapter`, `CommentAdapter`.
- `com.sysforum.app.models` – modelos de datos:
  - `Question`, `Comment`.
- `com.sysforum.app.utils` – utilidades:
  - `RankUtils` (cálculo de rango por puntos).

**¿Por qué separar así?**

- Para que sea más fácil encontrar el código relacionado con un tema concreto:
  - Si algo falla en login, voy directo a `auth/`.
  - Si hay un error en las listas, reviso `adapters/`.
- Para evitar “clases gigantes” donde todo está mezclado.
- Porque seguir una estructura lógica reduce mucho la dificultad de mantener el proyecto a largo plazo.

---

### 1.3. Recursos de interfaz: layouts y más

En `app/src/main/res/` se guardan todos los recursos visuales:

- `layout/` – define cómo se ve cada pantalla y cada ítem de lista:
  - `activity_main.xml`, `activity_login.xml`, `activity_register.xml`, `activity_create_question.xml`, `activity_question_details.xml`, etc.
  - Ítems como `item_question.xml` (cómo se ve una pregunta en el feed) y `item_comment.xml` (un comentario).
- `values/` – configura textos, colores, tamaños, estilos…
  - `strings.xml` – textos que se muestran en la UI (fácil de traducir o cambiar).
  - `colors.xml`, `themes.xml` – colores y temas Material.

**¿Por qué separar layout (XML) del código (Kotlin)?**

- Permite que diseño y lógica vayan “de la mano” pero no mezclados.
- Se puede cambiar el aspecto sin tocar la lógica (por ejemplo, mover botones, cambiar colores).
- Es el estándar de Android: facilita trabajar en equipo (un diseñador puede tocar XML, un programador toca Kotlin).

---

## Semana 2 – Activity y ciclo de vida

En esta semana la pregunta clave es: **¿qué es una Activity y cómo funciona su ciclo de vida?**

### 2.1. Qué es una Activity

Una **Activity** representa una pantalla con la que el usuario interactúa.

En SYSFORUM, por ejemplo:

- `LoginActivity` → pantalla de inicio de sesión.
- `RegisterActivity` → registro de usuario.
- `MainActivity` → pantalla principal con el listado de preguntas.
- `QuestionDetailsActivity` → detalle de una pregunta concreta.

Cada Activity:

- Tiene un método `onCreate` donde se inicializa todo.
- Normalmente se asocia a un layout con `setContentView(R.layout.mi_layout)`.

Ejemplo en `LoginActivity`:

```kotlin
class LoginActivity : AppCompatActivity() {

    private lateinit var auth: FirebaseAuth

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_login)

        auth = FirebaseAuth.getInstance()
        initViews()
        setupListeners()
    }
}
```

**¿Por qué todo empieza en `onCreate`?**

- `onCreate` se llama cuando la pantalla “nace”.
- Es el lugar adecuado para:
  - Inflar el layout.
  - Buscar las vistas (`findViewById`).
  - Inicializar servicios (Firebase, adaptadores, etc.).
  - Configurar listeners.

Más adelante se pueden usar otros métodos (`onStart`, `onResume`, etc.), pero para aprender es mejor centralizar las cosas en `onCreate`.

---

### 2.2. Ciclo de vida (resumen práctico)

El ciclo de vida completo de una Activity incluye métodos como:

- `onCreate` – se crea la Activity.
- `onStart` – está a punto de hacerse visible.
- `onResume` – el usuario ya la ve y puede interactuar.
- `onPause` – la Activity pierde foco (se abre otra encima, aparece un diálogo…).
- `onStop` – ya no se ve.
- `onDestroy` – se elimina.

En SYSFORUM, la mayoría del trabajo se hace en `onCreate`, y el resto del ciclo se maneja implícitamente (sin sobrescribir todos los métodos) para simplificar.

**¿Por qué esto es aceptable en este proyecto?**

- No se hace un uso intensivo de recursos que deban liberarse manualmente (por ejemplo, sensores siempre activos).
- Firebase maneja internamente muchas optimizaciones.
- Para una app de foro, el ciclo mínimo (centrado en `onCreate`) es suficiente para lograr una buena experiencia.

Más adelante, conocer este ciclo sirve para optimizar, pero al principio se busca **no complicar el código**.

---

### 2.3. Navegación entre Activities con Intents

SYSFORUM navega entre pantallas con **Intents explícitos**. Ejemplos:

- Desde `MainActivity` al formulario de nueva pregunta:

  ```kotlin
  addQuestionFab.setOnClickListener {
      startActivity(Intent(this, CreateQuestionActivity::class.java))
  }
  ```

- Desde `SplashActivity` a `LoginActivity` o `MainActivity` según el estado de sesión.
- Desde el menú de `MainActivity` a `SettingsActivity` o `ProfileSettingsActivity`.

**¿Por qué usar Intents explícitos (en vez de, por ejemplo, Fragments)?**

- Son fáciles de entender: “voy de una pantalla a otra”.
- Encajan muy bien con el nivel del curso en semanas iniciales.
- Permiten explicar paso a paso la navegación antes de introducir conceptos más avanzados como Fragments.

---

## Semana 3 – Layouts, ConstraintLayout y controles de entrada

Aquí se profundiza en **cómo se construyen las pantallas**.

### 3.1. ConstraintLayout en `activity_main.xml`

`activity_main.xml` es un buen ejemplo de cómo organizar una pantalla de forma flexible:

- `ConstraintLayout` como raíz.
- Una `AppBarLayout` con `MaterialToolbar` anclada arriba.
- Un `LinearLayout` que ocupa el resto del espacio y dentro contiene:
  - Un `ChipGroup` para filtrar categorías.
  - Un `SwipeRefreshLayout` con un `RecyclerView` dentro (lista de preguntas).
- Un `FloatingActionButton` anclado en la esquina inferior derecha.

**¿Por qué usar ConstraintLayout en lugar de muchos LinearLayouts anidados?**

- Menos profundidad de vistas = mejor rendimiento.
- Más control sobre posiciones relativas (arriba, abajo, centrado, etc.).
- Es el layout recomendado por Google para la mayoría de pantallas complejas.

---

### 3.2. Formularios con TextInputLayout y TextInputEditText

En `LoginActivity` y `RegisterActivity` se usan componentes de **Material Design** para los formularios:

- `TextInputLayout` envuelve cada `TextInputEditText` (email, contraseña, etc.).

Ejemplo simplificado (XML):

```xml
<com.google.android.material.textfield.TextInputLayout
    android:id="@+id/emailInputLayout"
    ...>

    <com.google.android.material.textfield.TextInputEditText
        android:id="@+id/emailEditText"
        android:inputType="textEmailAddress" />

</com.google.android.material.textfield.TextInputLayout>
```

En Kotlin se validan los campos y se ponen errores en el `TextInputLayout`:

```kotlin
if (TextUtils.isEmpty(email)) {
    emailInputLayout.error = getString(R.string.error_email_required)
} else if (!Patterns.EMAIL_ADDRESS.matcher(email).matches()) {
    emailInputLayout.error = getString(R.string.error_invalid_email)
} else {
    emailInputLayout.error = null
}
```

**¿Por qué usar estos componentes y no simples `EditText`?**

- Muestran errores de forma clara y bonita.
- Siguen las guías de Material Design.
- Mejoran la experiencia de usuario sin añadir mucha complejidad al código.

---

## Semana 4 – Evento Clic, Intents explícitos y listas

En esta semana se refuerza cómo la app responde a la interacción del usuario.

### 4.1. Eventos de clic (`setOnClickListener`)

Muchas acciones clave se activan por clic:

- Botón de login → llama a `loginUser()`.
- Botón de registro → `registerUser()`.
- FAB → abre `CreateQuestionActivity`.
- Elemento de la lista de preguntas → abre `QuestionDetailsActivity`.
- Icono de menú en una tarjeta de pregunta → mostrar menú de opciones (eliminar).

Ejemplo en `QuestionAdapter`:

```kotlin
holder.itemView.setOnClickListener {
    onQuestionClick(question)
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

**¿Por qué usar lambdas (`onQuestionClick`, `onDeleteClick`) en lugar de manejar todo dentro del adaptador?**

- Separa la lógica de UI (cómo se ve una pregunta) de la lógica de negocio (qué hacer al pulsar).
- Permite reutilizar el mismo adaptador en diferentes pantallas cambiando solo las acciones.

---

### 4.2. Intents explícitos (repaso práctico)

Navegar entre pantallas sigue el mismo patrón:

```kotlin
startActivity(Intent(this, OtraActivity::class.java))
```

**Por qué esto es importante en esta semana:**

- Es la base de la navegación en Android.
- Permite introducir el paso de datos entre Activities (por ejemplo, pasar un objeto `Question` a `QuestionDetailsActivity`).
- Sirve como base para luego explicar Intents **implícitos** (como los que abren Google Maps en Semana 13).

### 4.3. ListView vs RecyclerView

SYSFORUM ya usa `RecyclerView` (la opción moderna), pero el tema ListView se ve de forma conceptual:

- `ListView` fue la solución clásica para listas simples.
- `RecyclerView` es más flexible, extensible y eficiente.

En lugar de enseñar ListView “antiguo” en el código, se muestra cómo hoy en día lo normal es:

- `RecyclerView` + adaptador personalizado (como `QuestionAdapter`).
- Layout de ítem definido en XML (`item_question.xml`).

Esto conecta los conceptos de la semana con una implementación actualizada.

---

## Semana 5 – Fragment (concepto), Context y mensajes: Toast y SnackBar

Aunque SYSFORUM **no usa Fragments en producción**, el proyecto sirve para explicar varios conceptos.

### 5.1. Fragments (a nivel idea)

Un **Fragment** es como una “mini-pantalla” reutilizable dentro de una Activity.

En SYSFORUM, podríamos imaginar:

- Un `HomeFragment` que muestra la lista de preguntas dentro de `MainActivity`.
- Un `QuestionDetailsFragment` para el detalle, reutilizable en distintas configuraciones.

Sin embargo, el proyecto decidió usar Activities para simplificar:

- Menos abstracciones al inicio.
- Más fácil entender el flujo Activity → Activity.

**¿Por qué es útil igual hablar de Fragments?**

- Porque muchas apps grandes los usan (navegación con un solo Activity y muchos Fragments).
- Conocer Fragments ayuda a entender mejor las diferencias cuando se vea un proyecto más avanzado.

---

### 5.2. Context: `this`, `applicationContext`, `view.context`

En el código aparecen muchas veces variantes de “contexto”:

- `this` dentro de una Activity.
- `applicationContext` (contexto global de la app).
- `itemView.context` dentro de un ViewHolder de RecyclerView.

Ejemplos:

- Mostrar un Toast:

  ```kotlin
  Toast.makeText(this, "Mensaje", Toast.LENGTH_SHORT).show()
  ```

- Crear un diálogo:

  ```kotlin
  AlertDialog.Builder(this)
      .setTitle("Eliminar pregunta")
      // ...
      .show()
  ```

- Cargar una imagen con Glide desde un adaptador:

  ```kotlin
  Glide.with(holder.itemView.context)
      .load(imageToLoad)
      .into(holder.questionImage)
  ```

**¿Por qué es importante entender esto?**

- Porque muchas APIs de Android necesitan saber “desde qué contexto” se ejecutan.
- Usar el contexto correcto evita errores (por ejemplo, no se puede mostrar un diálogo con el contexto de aplicación, necesita un Activity).

---

### 5.3. Toast y (futuro) SnackBar

SYSFORUM usa muchos **Toast** para avisar al usuario:

- Login correcto o incorrecto.
- Pregunta eliminada.
- Comentario publicado.

Ejemplo:

```kotlin
Toast.makeText(this, "Pregunta eliminada", Toast.LENGTH_SHORT).show()
```

**¿Por qué Toast y no siempre SnackBar?**

- El Toast es muy fácil de usar y suficiente para mensajes rápidos.
- No depende del layout (aparece “por encima” de todo).
- Pedagógicamente, es un primer paso para hablar de feedback al usuario.

Luego se puede introducir **SnackBar** como mejora:

- Aparece dentro de la vista.
- Puede tener un botón “Deshacer”.
- Recomendado por Material Design para muchas interacciones.

SYSFORUM es un buen punto de partida para, por ejemplo, cambiar el Toast al eliminar pregunta por un SnackBar con opción de deshacer.

---

## Semana 6 – RecyclerView, Material Design y CardView

Esta semana se profundiza en listas modernas y en el estilo visual.

### 6.1. RecyclerView: la columna vertebral del feed

En `activity_main.xml` hay un `RecyclerView` dentro de un `SwipeRefreshLayout`:

- `SwipeRefreshLayout` permite el gesto de “tirar para refrescar”.
- `RecyclerView` muestra la lista de preguntas.

En `MainActivity` se configura así:

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

**¿Por qué usar un adaptador personalizado (`QuestionAdapter`)?**

- Porque cada ítem de la lista es complejo (avatar, título, descripción, contadores, imagen, menú de opciones).
- Un adaptador personalizado permite controlar exactamente cómo se muestra cada pregunta y cómo responde a los clics.

---

### 6.2. CardView / MaterialCardView: diseño de tarjetas

Cada pregunta se representa visualmente como una **tarjeta** (CardView o MaterialCardView) en `item_question.xml`.

- La tarjeta incluye:
  - Foto de perfil del autor.
  - Nombre del autor.
  - Título y descripción.
  - Imagen opcional de la pregunta.
  - Contadores de likes, dislikes, respuestas.
  - Botón de menú para opciones (eliminar, si es el autor).

**¿Por qué tarjetas?**

- Mejoran la legibilidad: cada pregunta es un bloque visual separado.
- Siguen las guías de Material Design (contenido como “cards”).
- Hacen que la app se vea moderna y profesional.

---

### 6.3. Material Design en toda la app

SYSFORUM adopta muchos componentes Material:

- `MaterialToolbar` – para la barra superior con título y menú.
- `FloatingActionButton` – botón flotante para acción principal (crear pregunta).
- `MaterialButton` – botones en formularios y pantallas de detalle.
- `TextInputLayout` y `TextInputEditText` – entradas de texto con errores y hints.
- `ChipGroup` y `Chip` – filtros de categorías.
- `MaterialCardView` – tarjetas de preguntas y comentarios.

**¿Por qué es importante esto en el curso?**

- Enseña a usar componentes modernos desde el principio.
- Hace que los estudiantes vean una app “como las de verdad”, no solo cajas grises.
- Facilita luego aprender otros componentes Material (SnackBar, BottomNavigation, etc.).

---

## Resumen final (Semanas 1–6)

Con las semanas 1–6 se construye toda la **base** de SYSFORUM:

- Se entiende la estructura de un proyecto Android (manifest, paquetes, recursos).
- Se dominan las Activities como pantallas principales y su ciclo de vida básico.
- Se aprende a diseñar pantallas con ConstraintLayout y controles de entrada amigables.
- Se manejan eventos de clic y navegación con Intents.
- Se usan listas modernas con RecyclerView, adaptadores y tarjetas Material.
- Se introduce el concepto de contexto, mensajes al usuario (Toast) y se prepara el terreno para mejoras (Fragments, SnackBar, etc.).

Todo esto permite que, cuando se añaden los temas más avanzados (semanas 9–14: hardware, Firebase, Maps, tiempo real), el proyecto siga siendo entendible: la base ya está bien construida y cada nuevo tema encaja en un lugar claro dentro de la app.
