# SimpsonsApp - Revision del 2do Parcial

## Objetivo
Documentar de forma tecnica los 10 errores aprobados del proyecto `SimpsonsApp`, priorizando defectos reales de compilacion, ejecucion, arquitectura, lifecycle y UI.

## Metodologia de Analisis
Se realizo una revision estatica del codigo fuente, contrastando contratos entre capas, configuracion de dependencias, flujo de Paging, manejo de estado en Compose, navegacion y consistencia de tematizacion. El analisis se limito a los 10 hallazgos aprobados.

## Resumen Ejecutivo

| # | Archivo | Categoria | Severidad |
| - | ------- | --------- | --------- |
| 1 | `domain/model/Episode.kt` | Error de compilacion | Critical |
| 2 | `di/DataModule.kt` | Error de ejecucion | Critical |
| 3 | `data/repository/EpisodeRepositoryImpl.kt` | Error de compilacion | Critical |
| 4 | `domain/repository/EpisodeRepository.kt` / `data/repository/EpisodeRepositoryImpl.kt` | Error de compilacion | Critical |
| 5 | `androidTest/ui/main/MainScreenTest.kt` | Error de compilacion | High |
| 6 | `data/remote/EpisodeRemoteMediator.kt` | Error logico | Critical |
| 7 | `data/repository/EpisodeRepositoryImpl.kt` | Error de arquitectura | High |
| 8 | `main/MainScreen.kt` | Error de Android Lifecycle | Critical |
| 9 | `detail/DetailViewModel.kt` / `detail/DetailScreen.kt` | Error de ejecucion | High |
| 10 | `MainActivity.kt` | Error de UI | High |

### Error 1 - Bloque `init` invalido fuera de la data class

**Archivo**  
`app/src/main/java/com/example/simpsonsapp/domain/model/Episode.kt`

**Linea**  
13-14

**Severidad**  
Critical

**Categoria**  
Error de compilacion

**Codigo Problematico**

```kotlin
init {
    return Episode; //NO BORRAR
}
```

**Problema**  
Existe un bloque `init` fuera de la `data class Episode`.

**Por que es un error**  
Kotlin solo permite bloques `init` dentro de una clase. En este estado, el archivo no compila y el modelo de dominio queda inutilizable.

**Como deberia solucionarse**  
Eliminar ese bloque residual o mover cualquier inicializacion valida al interior de una clase.

### Error 2 - Retrofit se crea sin `baseUrl`

**Archivo**  
`app/src/main/java/com/example/simpsonsapp/di/DataModule.kt`

**Linea**  
34-38

**Severidad**  
Critical

**Categoria**  
Error de ejecucion

**Codigo Problematico**

```kotlin
return Retrofit.Builder()
    .client(client)
    .addConverterFactory(GsonConverterFactory.create())
    .build()
    .create(SimpsonsApi::class.java)
```

**Problema**  
La instancia de Retrofit se construye sin definir `baseUrl(...)`.

**Por que es un error**  
Retrofit requiere una URL base obligatoria. Sin esa configuracion, la creacion del cliente falla en runtime.

**Como deberia solucionarse**  
Definir una `baseUrl` valida y dejar en la interfaz solo endpoints relativos.

### Error 3 - Falta importar `SimpsonsApi` en el repositorio

**Archivo**  
`app/src/main/java/com/example/simpsonsapp/data/repository/EpisodeRepositoryImpl.kt`

**Linea**  
18

**Severidad**  
Critical

**Categoria**  
Error de compilacion

**Codigo Problematico**

```kotlin
class EpisodeRepositoryImpl @Inject constructor(
    private val simpsonsApi: SimpsonsApi,
    private val appDatabase: AppDatabase
) : EpisodeRepository {
```

**Problema**  
El archivo usa `SimpsonsApi` sin importar el tipo.

**Por que es un error**  
El compilador no puede resolver referencias a tipos externos si no estan visibles en el archivo o en el mismo paquete.

**Como deberia solucionarse**  
Agregar el import correcto de `SimpsonsApi`.

### Error 4 - El contrato del repositorio no coincide con la implementacion

**Archivo**  
`app/src/main/java/com/example/simpsonsapp/domain/repository/EpisodeRepository.kt`  
`app/src/main/java/com/example/simpsonsapp/data/repository/EpisodeRepositoryImpl.kt`

**Linea**  
`EpisodeRepository.kt`: 8  
`EpisodeRepositoryImpl.kt`: 23

**Severidad**  
Critical

**Categoria**  
Error de compilacion

**Codigo Problematico**

```kotlin
interface EpisodeRepository {
    fun get_episodes(): Flow<PagingData<Episode>>
}
```

```kotlin
override fun getEpisodes(): Flow<PagingData<Episode>> {
```

**Problema**  
La interfaz y la implementacion exponen nombres distintos para el mismo metodo.

**Por que es un error**  
El `override` no coincide con el contrato y la capa de datos deja de compilar correctamente.

**Como deberia solucionarse**  
Unificar la firma del metodo en interfaz, implementacion y casos de uso.

### Error 5 - La prueba instrumentada llama a `MainScreen` con una firma invalida

**Archivo**  
`app/src/androidTest/java/com/example/simpsonsapp/ui/main/MainScreenTest.kt`

**Linea**  
18-23

**Severidad**  
High

**Categoria**  
Error de compilacion

**Codigo Problematico**

```kotlin
composeTestRule.setContent { MainScreen(FAKE_DATA) }
FAKE_DATA.forEach { composeTestRule.onNodeWithText("Hello $it!").assertExists() }
```

**Problema**  
La prueba usa una firma antigua de `MainScreen` y valida contenido que ya no pertenece a la UI actual.

**Por que es un error**  
La pantalla ahora requiere `onNavigateToDetail` y resuelve su `ViewModel` desde Hilt. El test quedo desacoplado de la implementacion real.

**Como deberia solucionarse**  
Actualizar la prueba para usar la API actual de la pantalla y asserts alineados con la UI vigente.

### Error 6 - El calculo de pagina en `REFRESH` puede recargar una pagina incorrecta

**Archivo**  
`app/src/main/java/com/example/simpsonsapp/data/remote/EpisodeRemoteMediator.kt`

**Linea**  
29-31

**Severidad**  
Critical

**Categoria**  
Error logico

**Codigo Problematico**

```kotlin
LoadType.REFRESH -> {
    val remoteKeys = getRemoteKeyClosestToCurrentPosition(state)
    remoteKeys?.nextKey?.minus(1) ?: 1
}
```

**Problema**  
El refresh puede reiniciarse desde una pagina intermedia antes de limpiar la cache.

**Por que es un error**  
Si luego se borran episodios y claves remotas, un refresh parcial puede dejar inconsistente el dataset reconstruido.

**Como deberia solucionarse**  
Usar una estrategia de `REFRESH` coherente con la invalidacion de cache, normalmente reiniciando desde la primera pagina.

### Error 7 - El filtrado por temporada no usa `RemoteMediator`

**Archivo**  
`app/src/main/java/com/example/simpsonsapp/data/repository/EpisodeRepositoryImpl.kt`

**Linea**  
34-39

**Severidad**  
High

**Categoria**  
Error de arquitectura

**Codigo Problematico**

```kotlin
override fun getEpisodesBySeason(season: Int): Flow<PagingData<Episode>> {
    val pagingSourceFactory = { appDatabase.episodeDao().pagingSourceBySeason(season) }
    return Pager(
        config = PagingConfig(pageSize = 20, enablePlaceholders = false),
        pagingSourceFactory = pagingSourceFactory
    ).flow.map { pagingData ->
        pagingData.map { it.toDomain() }
    }
}
```

**Problema**  
El filtro por temporada depende solo de la base local.

**Por que es un error**  
Si la temporada no fue descargada por completo, la UI puede mostrar resultados parciales o vacios aunque la API tenga datos disponibles.

**Como deberia solucionarse**  
Incorporar sincronizacion remota al flujo filtrado o rediseñar la estrategia de carga por temporada.

### Error 8 - Se ejecuta un side effect durante la composicion

**Archivo**  
`app/src/main/java/com/example/simpsonsapp/main/MainScreen.kt`

**Linea**  
50-53

**Severidad**  
Critical

**Categoria**  
Error de Android Lifecycle

**Codigo Problematico**

```kotlin
if (episodes.loadState.refresh is LoadState.NotLoading && seasons.isEmpty()) {
    viewModel.refreshSeasons()
}
```

**Problema**  
La pantalla dispara trabajo del `ViewModel` directamente desde el cuerpo del composable.

**Por que es un error**  
Las recomposiciones pueden repetir esa llamada multiples veces, generando consultas duplicadas y efectos no deterministas.

**Como deberia solucionarse**  
Mover ese comportamiento a `LaunchedEffect` u otro mecanismo controlado por estado y lifecycle.

### Error 9 - La pantalla de detalle puede quedar cargando indefinidamente

**Archivo**  
`app/src/main/java/com/example/simpsonsapp/detail/DetailViewModel.kt`  
`app/src/main/java/com/example/simpsonsapp/detail/DetailScreen.kt`

**Linea**  
`DetailViewModel.kt`: 24-33  
`DetailScreen.kt`: 64-65

**Severidad**  
High

**Categoria**  
Error de ejecucion

**Codigo Problematico**

```kotlin
val episodeId = savedStateHandle.get<Int>("episodeId")
episodeId?.let { id ->
    loadEpisodeDetail(id)
}
```

```kotlin
} else {
    CircularProgressIndicator(modifier = Modifier.align(Alignment.Center))
}
```

**Problema**  
Si el episodio no existe o falta el argumento, la pantalla queda en loading permanente.

**Por que es un error**  
No hay estados explicitos de error o vacio, por lo que el usuario nunca recibe feedback ni salida del spinner.

**Como deberia solucionarse**  
Modelar estados de carga, error y vacio, y validar el argumento antes de renderizar la UI.

### Error 10 - `MainActivity` usa `MaterialTheme` en lugar de `SimpsonsAppTheme`

**Archivo**  
`app/src/main/java/com/example/simpsonsapp/MainActivity.kt`

**Linea**  
17-20

**Severidad**  
High

**Categoria**  
Error de UI

**Codigo Problematico**

```kotlin
setContent {
    MaterialTheme {
        Surface(
            modifier = Modifier.fillMaxSize(),
            color = MaterialTheme.colorScheme.background
        ) {
            AppNavigation()
        }
    }
}
```

**Problema**  
La actividad principal evita el tema propio definido por el proyecto.

**Por que es un error**  
Se pierden la paleta, tipografia y configuraciones centralizadas de `SimpsonsAppTheme`, afectando la consistencia visual de toda la app.

**Como deberia solucionarse**  
Envolver el contenido con `SimpsonsAppTheme` en lugar de `MaterialTheme` directo.

## Conclusiones
El proyecto concentra fallas criticas en compilacion, configuracion de red, contrato entre capas y manejo de efectos en Compose. Tambien presenta problemas importantes de arquitectura de Paging, estados de detalle y consistencia visual.

## Prioridad de Correccion

1. Corregir primero los errores que impiden compilar: `Episode.kt`, `EpisodeRepository.kt` y `EpisodeRepositoryImpl.kt`.
2. Resolver luego la configuracion de Retrofit y la logica de `REFRESH` en `EpisodeRemoteMediator`.
3. Ajustar el side effect de `MainScreen`, el flujo filtrado por temporada y el estado de detalle.
4. Actualizar la prueba instrumentada y aplicar `SimpsonsAppTheme` en `MainActivity`.

## Evaluacion Final
Needs fixes before submission
