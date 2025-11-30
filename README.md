# Estructura y Análisis del Proyecto Pokémon Compose

Este documento describe la estructura de archivos y directorios del proyecto, explicando el propósito de cada componente principal con los códigos completos.

## 1. Esquema General de la Estructura

```
.
├── build.gradle.kts
├── settings.gradle.kts
├── gradle/libs.versions.toml
└── app/
    ├── build.gradle.kts
    └── src/
        ├── main/
        │   ├── AndroidManifest.xml
        │   ├── java/com/turingalan/pokemon/
        │   │   ├── MainActivity.kt
        │   │   ├── di/
        │   │   ├── data/
        │   │   └── ui/
        │   └── res/
        ├── test/
        └── androidTest/
```

---

## 2. Análisis Detallado del Código Fuente

A continuación se describe cada archivo de código Kotlin del directorio `app/src/main/java/com/turingalan/pokemon/`.

### 2.1. Raíz (`/pokemon`)

#### `MainActivity.kt`
Es el punto de entrada de la aplicación. Se encarga de configurar el tema de Compose y de inicializar el grafo de navegación (`NavGraph`). La anotación `@AndroidEntryPoint` la habilita para recibir dependencias con Hilt.

```kotlin
package com.turingalan.pokemon

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.activity.enableEdgeToEdge
import com.turingalan.pokemon.ui.navigation.NavGraph
import com.turingalan.pokemon.ui.theme.PokemonTheme
import dagger.hilt.android.AndroidEntryPoint

@AndroidEntryPoint
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()
        setContent {
            PokemonTheme {
                NavGraph()
            }
        }
    }
}
```

---

### 2.2. Capa de Datos (`/data`)

#### `PokemonDataSource.kt`
Es una interfaz que define un contrato común para las fuentes de datos. Tanto la fuente de datos local (`PokemonLocalDataSource`) como la remota (`PokemonRemoteDataSource`) la implementan, permitiendo que el repositorio las trate de forma intercambiable.

```kotlin
package com.turingalan.pokemon.data

import com.turingalan.pokemon.data.model.Pokemon
import kotlinx.coroutines.flow.Flow

interface PokemonDataSource {
    suspend fun addAll(pokemonList: List<Pokemon>)
    fun observe(): Flow<Result<List<Pokemon>>>
    suspend fun readAll(): Result<List<Pokemon>>
    suspend fun readOne(id: Long): Result<Pokemon>
    suspend fun isError()
}
```

#### `data/model/Pokemon.kt`
El modelo de dominio (o modelo "limpio"). Representa la estructura de datos fundamental de la aplicación. Esta es la clase que debe usarse en la capa de UI y de negocio. No contiene anotaciones de Room o Gson.

```kotlin
package com.turingalan.pokemon.data.model

data class Pokemon(
    val id:Long,
    val name:String,
    val sprite:String,
    val artwork:String,
)
```

#### `data/local/`
Contiene todo lo relacionado con la base de datos local (Room).

##### `PokemonDatabase.kt`
Define la base de datos Room, las entidades que contiene (`PokemonEntity`) y la versión. Provee acceso a los DAOs.

```kotlin
package com.turingalan.pokemon.data.local

import androidx.room.Database
import androidx.room.RoomDatabase

@Database(entities = [PokemonEntity::class], version = 1)
abstract class PokemonDatabase : RoomDatabase() {
    abstract fun getPokemonDao(): PokemonDao
}
```

##### `PokemonDao.kt`
(Data Access Object) Interfaz con las operaciones de la base de datos (Insertar, Consultar, Borrar). Room genera la implementación automáticamente. `observeAll()` devuelve un `Flow` para que la UI reaccione a los cambios en la base de datos.

```kotlin
package com.turingalan.pokemon.data.local

import androidx.room.Dao
import androidx.room.Delete
import androidx.room.Insert
import androidx.room.Query
import kotlinx.coroutines.flow.Flow

@Dao
interface PokemonDao {
    @Insert
    suspend fun insert(pokemon: PokemonEntity): Long

    @Delete
    suspend fun deleteOne(pokemon: PokemonEntity): Int

    @Query("SELECT * FROM pokemon")
    suspend fun getAll(): List<PokemonEntity>

    @Query("SELECT * FROM pokemon")
    fun observeAll(): Flow<List<PokemonEntity>>

    @Query("SELECT * FROM pokemon WHERE id = :id")
    suspend fun readPokemonById(id: Long): PokemonEntity?
}
```

##### `PokemonEntity.kt`
Representa la tabla `pokemon` en la base de datos. Es una clase "sucia", específica de la capa de persistencia. Incluye funciones _mapper_ para convertir entre `PokemonEntity` y el modelo de dominio `Pokemon`.

```kotlin
package com.turingalan.pokemon.data.local

import androidx.room.Entity
import androidx.room.PrimaryKey
import com.turingalan.pokemon.data.model.Pokemon

@Entity(tableName = "pokemon")
data class PokemonEntity(
    @PrimaryKey
    val id: Long,
    val name: String,
    val sprite: String,
    val artwork: String
)

fun Pokemon.toEntity(): PokemonEntity {
    return PokemonEntity(
        id = this.id,
        name = this.name,
        sprite = this.sprite,
        artwork = this.artwork
    )
}

fun List<PokemonEntity>.toModel(): List<Pokemon> {
    return this.map {
        Pokemon(
            id = it.id,
            name = it.name,
            sprite = it.sprite,
            artwork = it.artwork
        )
    }
}

fun PokemonEntity.toModel(): Pokemon {
    return Pokemon(
        id = this.id,
        name = this.name,
        sprite = this.sprite,
        artwork = this.artwork
    )
}
```

##### `PokemonLocalDataSource.kt`
Implementación de `PokemonDataSource` que trabaja con Room. Se encarga de llamar al `PokemonDao` y de usar los _mappers_ para convertir los datos entre `Entity` y `Model`.

```kotlin
package com.turingalan.pokemon.data.local

import com.turingalan.pokemon.data.PokemonDataSource
import com.turingalan.pokemon.data.model.Pokemon
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.map
import kotlinx.coroutines.withContext
import javax.inject.Inject

class PokemonLocalDataSource @Inject constructor(
    private val scope: CoroutineScope, 
    private val pokemonDao: PokemonDao 
): PokemonDataSource {

    override suspend fun addAll(pokemonList: List<Pokemon>) {
        pokemonList.forEach { pokemon ->
            val entity = pokemon.toEntity()

            withContext(Dispatchers.IO) {
                pokemonDao.insert(entity)
            }
        }
    }

    override fun observe(): Flow<Result<List<Pokemon>>> {
        val databaseFlow = pokemonDao.observeAll()

        return databaseFlow.map { entities ->
            Result.success(entities.toModel())
        }
    }

    override suspend fun readAll(): Result<List<Pokemon>> {
        val result = Result.success(pokemonDao.getAll().toModel())
        return result
    }

    override suspend fun readOne(id: Long): Result<Pokemon> {
        val entity = pokemonDao.readPokemonById(id)

        return if(entity == null){
            Result.failure(PokemonNotFoundException())
        }
        else
            Result.success(entity.toModel())
    }

    override suspend fun isError() {
        TODO("Not yet implemented")
    }
}
```

##### `PokemonNotFoundException.kt`
Una excepción personalizada para un manejo de errores más claro y semántico cuando no se encuentra un Pokémon.

```kotlin
package com.turingalan.pokemon.data.local

class PokemonNotFoundException : RuntimeException() {
}
```

#### `data/remote/`
Contiene todo lo relacionado con la API remota (Retrofit y Gson).

##### `PokemonApi.kt`
Interfaz de Retrofit que define los _endpoints_ de la API. Las anotaciones `@GET`, `@Query` y `@Path` configuran las peticiones HTTP.

```kotlin
package com.turingalan.pokemon.data.remote

import androidx.compose.ui.geometry.Offset
import com.turingalan.pokemon.data.remote.model.PokemonListRemote
import com.turingalan.pokemon.data.remote.model.PokemonRemote
import retrofit2.Response
import retrofit2.http.GET
import retrofit2.http.Path
import retrofit2.http.Query

interface PokemonApi {
    @GET("/api/v2/pokemon/")
    suspend fun readAll(@Query("limit") limit:Int=60, @Query("offset") offset:Int=0): Response<PokemonListRemote>

    @GET("/api/v2/pokemon/{id}")
    suspend fun readOne(@Path("id") id: Long): Response<PokemonRemote>

    @GET("/api/v2/pokemon/{name}")
    suspend fun readOne(@Path("name") name: String): Response<PokemonRemote>
}
```

##### `PokemonRemoteDataSource.kt`
Implementación de `PokemonDataSource` que usa Retrofit (`PokemonApi`) para obtener datos de internet. Maneja las respuestas HTTP, los errores de red y convierte los modelos `Remote` (DTOs) al modelo de dominio `Pokemon`.

```kotlin
package com.turingalan.pokemon.data.remote

import com.turingalan.pokemon.data.PokemonDataSource
import com.turingalan.pokemon.data.model.Pokemon
import com.turingalan.pokemon.data.remote.model.PokemonRemote
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.SharingStarted
import kotlinx.coroutines.flow.flow
import kotlinx.coroutines.flow.shareIn
import javax.inject.Inject

class PokemonRemoteDataSource @Inject constructor(
    private val api: PokemonApi,
    private val scope: CoroutineScope
): PokemonDataSource {

    override suspend fun addAll(pokemonList: List<Pokemon>) {
        TODO("Not yet implemented")
    }

    override fun observe(): Flow<Result<List<Pokemon>>> {
        return flow {
            emit(Result.success(listOf<Pokemon>()))
            val result = readAll()
            emit(result)
        }.shareIn(
            scope = scope,
            started = SharingStarted.WhileSubscribed(5_000L), 
            replay = 1 
        )
    }

    override suspend fun readAll(): Result<List<Pokemon>> {
        try {
            val response = api.readAll(limit = 20, offset = 0)
            val finalList = mutableListOf<Pokemon>()

            return if (response.isSuccessful) {
                val body = response.body()!! 
                for (result in body.results) {
                    val remotePokemon = readOne(name = result.name)
                    remotePokemon?.let {
                        finalList.add(it)
                    }
                }
                Result.success(finalList)
            } else {
                Result.failure(RuntimeException("Error code: ${response.code()}"))
            }
        } catch (e: Exception) {
            return Result.failure(e)
        }
    }

    private suspend fun readOne(name: String): Pokemon? {
        val response = api.readOne(name)
        return if (response.isSuccessful) {
            response.body()!!.toExternal() 
        } else {
            null
        }
    }

    override suspend fun readOne(id: Long): Result<Pokemon> {
        try {
            val response = api.readOne(id)
            return if (response.isSuccessful) {
                val pokemon = response.body()!!.toExternal()
                Result.success(pokemon)
            } else {
                Result.failure(RuntimeException("Error code: ${response.code()}"))
            }
        } catch (e: Exception) {
            return Result.failure(e)
        }
    }

    override suspend fun isError() {
        TODO("Not yet implemented")
    }
}

fun PokemonRemote.toExternal(): Pokemon {
    return Pokemon(
        id = this.id,
        name = this.name,
        sprite = this.sprites.front_default,
        artwork = this.sprites.other.officialArtwork.front_default
    )
}
```

##### `model/PokemonApiModel.kt`
(DTO - Data Transfer Objects) Clases que representan la estructura exacta del JSON que devuelve la API. Se usan solo para deserializar la respuesta con Gson y después se mapean al modelo de dominio.

```kotlin
package com.turingalan.pokemon.data.remote.model

import com.google.gson.annotations.SerializedName
import com.turingalan.pokemon.data.model.Pokemon

data class PokemonListRemote(
    val results: List<PokemonListItemRemote>
)

data class PokemonListItemRemote(
    val name: String,
    val url: String,
)

data class PokemonRemote(
    val id: Long,
    val name: String,
    val sprites: PokemonSprites, 
)

data class PokemonSprites(
    val front_default: String,
    var other: PokemonOtherSprites,
)

data class PokemonOtherSprites(
    @SerializedName("official-artwork")
    val officialArtwork: PokemonOfficialArtwork,
)

data class PokemonOfficialArtwork(
    val front_default: String
)
```

#### `data/repository/`
El repositorio es el orquestador que decide de dónde obtener los datos.

##### `PokemonRepository.kt`
La interfaz del repositorio. Define las operaciones de datos que la capa de UI necesita, abstrayendo la complejidad del origen de los datos.

```kotlin
package com.turingalan.pokemon.data.repository

import com.turingalan.pokemon.data.model.Pokemon
import kotlinx.coroutines.flow.Flow

interface PokemonRepository {

    suspend fun readOne(id:Long): Result<Pokemon>
    suspend fun readAll(): Result<List<Pokemon>>
    fun observe(): Flow<Result<List<Pokemon>>>
}
```

##### `PokemonRepositoryImpl.kt`
Implementación del repositorio. Inyecta ambas fuentes de datos (`local` y `remote`) y coordina la estrategia "Single Source of Truth": devuelve datos de la base de datos local inmediatamente y lanza una actualización desde la red en segundo plano.

```kotlin
package com.turingalan.pokemon.data.repository

import com.turingalan.pokemon.data.model.Pokemon
import com.turingalan.pokemon.data.PokemonDataSource
import com.turingalan.pokemon.di.LocalDataSource
import com.turingalan.pokemon.di.RemoteDataSource
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.launch
import javax.inject.Inject

class PokemonRepositoryImpl @Inject constructor(
    @RemoteDataSource private val remoteDataSource: PokemonDataSource,
    @LocalDataSource private val localDataSource: PokemonDataSource,
    private val scope: CoroutineScope
): PokemonRepository {

    override suspend fun readOne(id: Long): Result<Pokemon> {
        return remoteDataSource.readOne(id)
    }

    override suspend fun readAll(): Result<List<Pokemon>> {
        return remoteDataSource.readAll()
    }

    override fun observe(): Flow<Result<List<Pokemon>>> {
        scope.launch {
            refresh() 
        }
        return localDataSource.observe()
    }

    private suspend fun refresh() {
        val resultRemotePokemon = remoteDataSource.readAll()
        if (resultRemotePokemon.isSuccess) {
            localDataSource.addAll(resultRemotePokemon.getOrNull()!!)
        }
    }
}
```

---

### 2.3. Inyección de Dependencias (`/di`)

#### `PokemonApplication.kt`
Clase `Application` personalizada y anotada con `@HiltAndroidApp` para inicializar el contenedor de dependencias de Hilt en toda la aplicación.

```kotlin
package com.turingalan.pokemon.di

import android.app.Application
import dagger.hilt.android.HiltAndroidApp

@HiltAndroidApp
class PokemonApplication(): Application()
```

#### `AppModule.kt`
Módulo de Hilt que usa `@Binds` para enlazar implementaciones a sus interfaces (`PokemonRepositoryImpl` a `PokemonRepository`). Los calificadores (`@LocalDataSource`, `@RemoteDataSource`) se usan para poder inyectar dos implementaciones distintas de la misma interfaz (`PokemonDataSource`).

```kotlin
package com.turingalan.pokemon.di

import com.turingalan.pokemon.data.PokemonDataSource
import com.turingalan.pokemon.data.local.PokemonLocalDataSource
import com.turingalan.pokemon.data.remote.PokemonRemoteDataSource
import com.turingalan.pokemon.data.repository.PokemonRepository
import com.turingalan.pokemon.data.repository.PokemonRepositoryImpl
import dagger.Binds
import dagger.Module
import dagger.hilt.InstallIn
import dagger.hilt.components.SingletonComponent
import javax.inject.Qualifier
import javax.inject.Singleton

@Module
@InstallIn(SingletonComponent::class)
abstract class AppModule {

    @Binds
    @Singleton
    @RemoteDataSource
    abstract fun bindsRemotePokemonDataSource(ds: PokemonRemoteDataSource): PokemonDataSource

    @Binds
    @Singleton
    @LocalDataSource
    abstract fun bindsLocalPokemonDataSource(ds: PokemonLocalDataSource): PokemonDataSource

    @Binds
    @Singleton
    abstract  fun bindPokemonRepository(repository: PokemonRepositoryImpl): PokemonRepository
}

@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class LocalDataSource

@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class RemoteDataSource
```

#### `DatabaseModule.kt`
Módulo de Hilt que usa `@Provides` para construir y proveer instancias que Hilt no puede crear por sí solo, como la base de datos Room (`PokemonDatabase`) y el `PokemonDao`.

```kotlin
package com.turingalan.pokemon.di

import android.content.Context
import androidx.room.Room
import com.turingalan.pokemon.data.local.PokemonDao
import com.turingalan.pokemon.data.local.PokemonDatabase
import dagger.Module
import dagger.Provides
import dagger.hilt.InstallIn
import dagger.hilt.android.qualifiers.ApplicationContext
import dagger.hilt.components.SingletonComponent
import javax.inject.Singleton

@Module
@InstallIn(SingletonComponent::class)
class DatabaseModule {

    @Provides
    @Singleton
    fun provideDatabase(
        @ApplicationContext applicationContext: Context
    ): PokemonDatabase {

        val database = Room.databaseBuilder(context = applicationContext,
            PokemonDatabase::class.java,
            name = "pokemon-db").build()
        return database
    }

    @Provides
    fun providePokemonDao(
        database: PokemonDatabase
    ): PokemonDao {
        return database.getPokemonDao()
    }
}
```

#### `RemoteModule.kt`
Módulo de Hilt que provee las dependencias de la capa remota, como el cliente de Retrofit (`PokemonApi`) y un `CoroutineScope` para el repositorio.

```kotlin
package com.turingalan.pokemon.di

import com.turingalan.pokemon.data.remote.PokemonApi
import com.turingalan.pokemon.data.PokemonDataSource
import com.turingalan.pokemon.data.remote.PokemonRemoteDataSource
import dagger.Module
import dagger.Provides
import dagger.hilt.InstallIn
import dagger.hilt.components.SingletonComponent
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.SupervisorJob
import retrofit2.Retrofit
import retrofit2.converter.gson.GsonConverterFactory
import javax.inject.Singleton


@Module
@InstallIn(SingletonComponent::class)
class RemoteModule {

    @Provides
    @Singleton
    fun providePokemonApi(): PokemonApi{
        val retrofit = Retrofit.Builder()
            .baseUrl("https://pokeapi.co")
            .addConverterFactory(GsonConverterFactory.create())
            .build()

        return retrofit.create(PokemonApi::class.java)
    }

    @Provides

    fun provideCoroutineScope(): CoroutineScope {
        return CoroutineScope(SupervisorJob() + Dispatchers.Default)
    }

}
```

---

### 2.4. Capa de UI (`/ui`)

#### `ui/common/`

##### `AppTopBar.kt`
Define la barra superior de la aplicación.

```kotlin
package com.turingalan.pokemon.ui.common

import androidx.compose.material3.ExperimentalMaterial3Api
import androidx.compose.material3.Text
import androidx.compose.material3.TopAppBar
import androidx.compose.runtime.Composable
import androidx.compose.ui.res.stringResource
import androidx.hilt.lifecycle.viewmodel.compose.hiltViewModel
import androidx.navigation.NavBackStackEntry
import com.turingalan.pokemon.R
import com.turingalan.pokemon.ui.detail.PokemonDetailViewModel

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun AppTopBar(
    backStackEntry: NavBackStackEntry? = null,
) {
    TopAppBar(
        title = {
            Text(text=stringResource(R.string.app_name))
        }
    )

}
```

#### `ui/detail/`
Componentes de la pantalla de detalle de un Pokémon.

##### `PokemonDetailScreen.kt`
La `Composable` para la pantalla de detalle. Muestra la imagen (artwork) y el nombre del Pokémon que recibe del `DetailUiState`.

```kotlin
package com.turingalan.pokemon.ui.detail

import androidx.compose.foundation.Image
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.material3.Surface
import androidx.compose.runtime.Composable
import androidx.compose.runtime.collectAsState
import androidx.compose.runtime.getValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.layout.ContentScale
import androidx.compose.ui.res.painterResource
import androidx.compose.ui.tooling.preview.Preview
import androidx.hilt.lifecycle.viewmodel.compose.hiltViewModel
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import coil3.compose.AsyncImage
import com.turingalan.pokemon.R


@Composable
fun PokemonDetailScreen(
    modifier: Modifier = Modifier,
    viewModel: PokemonDetailViewModel = hiltViewModel()
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    PokemonDetailScreen(
        modifier = modifier,
        name = uiState.name,
        artwork =  uiState.artwork,
    )
}

@Composable
fun PokemonDetailScreen(
    modifier: Modifier = Modifier,
    name: String,
    artwork: String?,
    )
{

    Column(modifier = modifier.fillMaxSize(),

        horizontalAlignment = Alignment.CenterHorizontally) {
        if (artwork != null)  {
            AsyncImage(contentDescription = name,
                model = artwork)
        }
    }

}
```

##### `PokemonDetailViewModel.kt`
Similar al de la lista, pero para la pantalla de detalle. Usa `SavedStateHandle` para obtener el `id` del Pokémon de los argumentos de navegación y pide al repositorio los datos de ese Pokémon en concreto.

```kotlin
package com.turingalan.pokemon.ui.detail

import androidx.lifecycle.SavedStateHandle
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import androidx.navigation.toRoute
import com.turingalan.pokemon.data.model.Pokemon
import com.turingalan.pokemon.data.repository.PokemonRepository
import com.turingalan.pokemon.ui.navigation.Route
import dagger.hilt.android.lifecycle.HiltViewModel
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.launch
import javax.inject.Inject

data class DetailUiState(
    val name:String = "",
    val artwork:String? = ""
)

@HiltViewModel
class PokemonDetailViewModel @Inject constructor(
    savedStateHandle: SavedStateHandle, 
    private val pokemonRepository: PokemonRepository

): ViewModel() {
    private val _uiState: MutableStateFlow<DetailUiState> =
        MutableStateFlow(DetailUiState())
    val uiState: StateFlow<DetailUiState>
        get() = _uiState.asStateFlow()

    init {
        viewModelScope.launch {
            val route = savedStateHandle.toRoute<Route.Detail>()
            val pokemonId = route.id
            val pokemonResult = pokemonRepository.readOne(pokemonId)

            val pokemon = pokemonResult.getOrNull()
            pokemon?.let {
                _uiState.value = it.toDetailUiState()
            }
        }
    }
}

fun Pokemon.toDetailUiState(): DetailUiState = DetailUiState(
    name = this.name,
    artwork = this.artwork,
)
```

#### `ui/list/`
Componentes de la pantalla que muestra la lista de Pokémon.

##### `PokemonListViewModel.kt`
Gestiona el estado y la lógica de la pantalla de lista. Obtiene los datos del repositorio, los convierte a un `UiState` y los expone a la `Composable` a través de un `StateFlow`.

```kotlin
package com.turingalan.pokemon.ui.list

import androidx.lifecycle.SavedStateHandle
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.turingalan.pokemon.data.model.Pokemon
import com.turingalan.pokemon.data.repository.PokemonRepository
import dagger.hilt.android.lifecycle.HiltViewModel
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.launch
import javax.inject.Inject

@HiltViewModel
class PokemonListViewModel @Inject constructor(
    private val savedStateHandle: SavedStateHandle, 
    private val repository: PokemonRepository 
): ViewModel() {

    private val _uiState: MutableStateFlow<ListUiState> =
        MutableStateFlow(value = ListUiState.Initial)
    val uiState: StateFlow<ListUiState>
        get() = _uiState.asStateFlow()

    init {
        viewModelScope.launch {
            _uiState.value = ListUiState.Loading 

            repository.observe().collect { result ->
                if (result.isSuccess) {
                    val pokemons = result.getOrNull()!!
                    val uiPokemons = pokemons.asListUiState()
                    _uiState.value = ListUiState.Success(uiPokemons)
                } else {
                    _uiState.value = ListUiState.Error
                }
            }
        }
    }
}

sealed class ListUiState {
    object Initial: ListUiState()
    object Loading: ListUiState()
    object Error: ListUiState()
    data class Success(
        val pokemons: List<ListItemUiState>
    ): ListUiState()
}

data class ListItemUiState(
    val id: Long,
    val name: String,
    val sprite: String,
)

fun Pokemon.asListItemUiState(): ListItemUiState {
    return ListItemUiState(
        id = this.id,
        name = this.name.replaceFirstChar { it.uppercase() }, 
        sprite = this.sprite
    )
}

fun List<Pokemon>.asListUiState(): List<ListItemUiState> = this.map(Pokemon::asListItemUiState)
```

##### `PokemonListScreen.kt`
La función `Composable` que renderiza la pantalla de lista. Es "tonta": solo observa el `uiState` del ViewModel y pinta lo que este le dice (una pantalla de carga, un error o la lista).

```kotlin
package com.turingalan.pokemon.ui.list

import androidx.compose.foundation.clickable
import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.PaddingValues
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.size
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.material3.Card
import androidx.compose.material3.ExperimentalMaterial3ExpressiveApi
import androidx.compose.material3.LoadingIndicator
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.runtime.collectAsState
import androidx.compose.runtime.getValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.layout.ContentScale
import androidx.compose.ui.unit.dp
import androidx.hilt.lifecycle.viewmodel.compose.hiltViewModel
import coil3.compose.AsyncImage

@OptIn(ExperimentalMaterial3ExpressiveApi::class)
@Composable
fun PokemonListScreen(
    modifier: Modifier = Modifier,
    viewModel: PokemonListViewModel = hiltViewModel(),
    onShowDetail:(Long)->Unit,
) {
    val uiState by viewModel.uiState.collectAsState()
    when(uiState) {
        is ListUiState.Initial -> {

        }
        is ListUiState.Loading -> {
            PokemonLoadingScreen(modifier)
        }
        is ListUiState.Success -> {
            PokemonList(modifier, uiState, onShowDetail)
        }
        is ListUiState.Error -> {
            PokemonError()
        }
    }
}

@Composable
private fun PokemonError(
    modifier: Modifier = Modifier
) {
    Column(
        modifier = modifier.fillMaxSize(),
        verticalArrangement = Arrangement.Center,
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Text("Se ha producido un error", style = MaterialTheme.typography.titleLarge)
    }
}

@Composable
@OptIn(ExperimentalMaterial3ExpressiveApi::class)
private fun PokemonLoadingScreen(modifier: Modifier) {
    Column(
        modifier = modifier.fillMaxSize(),
        verticalArrangement = Arrangement.Center,
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        LoadingIndicator(
            modifier = Modifier.size(128.dp)
        )
    }
}

@Composable
private fun PokemonList(
    modifier: Modifier,
    uiState: ListUiState,
    onShowDetail: (Long) -> Unit
) {
    LazyColumn(
        modifier = modifier,
        contentPadding = PaddingValues(horizontal = 8.dp),
        verticalArrangement = Arrangement.spacedBy(4.dp)

    ) {
        items(
            items = (uiState as ListUiState.Success).pokemons,
            key = { item ->
                item.id
            }
        )
        {
            PokemonListItemCard(
                pokemonId = it.id,
                name = it.name,
                sprite = it.sprite,
                onShowDetail = onShowDetail
            )
        }
    }
}

@Composable
fun PokemonListItemCard(
    modifier: Modifier = Modifier,
    pokemonId: Long,
    name: String,
    sprite: String,
    onShowDetail: (Long) -> Unit,
)
{
    Card(
        modifier = Modifier
            .fillMaxWidth()
            .height(128.dp)
            .clickable(
                enabled = true,
                onClick = {
                    onShowDetail(pokemonId)
                })
    ) {
        Row(
            verticalAlignment = Alignment.CenterVertically,
            horizontalArrangement = Arrangement.SpaceAround
        ) {
            AsyncImage(
                modifier = Modifier.size(64.dp),
                model = sprite,
                contentDescription = name,
                contentScale = ContentScale.Fit
            )
            Text(text= name,
                style = MaterialTheme.typography.headlineSmall)
        }

    }
}
```

#### `ui/navigation/`
Clases y funciones para gestionar la navegación con Jetpack Compose Navigation.

##### `Route.kt`
Define todas las rutas de la aplicación como una `sealed class` serializable. Esto permite una navegación fuertemente tipada y más segura. También incluye funciones de extensión para simplificar las llamadas de navegación.

```kotlin
package com.turingalan.pokemon.ui.navigation

import androidx.compose.ui.Modifier
import androidx.navigation.NavController
import androidx.navigation.NavGraphBuilder
import androidx.navigation.compose.composable
import com.turingalan.pokemon.ui.detail.PokemonDetailScreen
import com.turingalan.pokemon.ui.list.PokemonListScreen
import kotlinx.serialization.Serializable

@Serializable
sealed class Route(val route:String) {
    @Serializable
    data object List:Route("pokemon_list")
    @Serializable
    data class Detail(val id:Long):Route(route = "pokemon_detail[$id]")
}

fun NavController.navigateToPokemonDetails(id:Long) {
    this.navigate(Route.Detail(id))
}

fun NavGraphBuilder.pokemonDetailDestination(
    modifier:Modifier = Modifier,
) {
    composable<Route.Detail> {


            backStackEntry ->
        PokemonDetailScreen(
            modifier = modifier,
        )


    }
}

fun NavGraphBuilder.pokemonListDestination(
    modifier:Modifier = Modifier,
    onNavigateToDetails:(Long)->Unit
) {
    composable<Route.List> {

        PokemonListScreen(modifier = modifier,
            onShowDetail = {
                    id ->
                onNavigateToDetails(id)
            })


    }
}
```

##### `NavGraph.kt`
El `Composable` principal que configura el `NavHost`. Define qué pantalla (`Composable`) se muestra para cada ruta y gestiona la lógica de navegación entre ellas, como pasar el `id` a la pantalla de detalle.

```kotlin
package com.turingalan.pokemon.ui.navigation

import androidx.compose.foundation.layout.consumeWindowInsets
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.padding
import androidx.compose.material3.ExperimentalMaterial3Api
import androidx.compose.material3.Scaffold
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.ui.Modifier
import androidx.navigation.NavController
import androidx.navigation.NavGraphBuilder
import androidx.navigation.compose.NavHost
import androidx.navigation.compose.composable
import androidx.navigation.compose.currentBackStackEntryAsState
import androidx.navigation.compose.rememberNavController
import com.turingalan.pokemon.ui.navigation.Route
import com.turingalan.pokemon.ui.common.AppTopBar
import com.turingalan.pokemon.ui.detail.PokemonDetailScreen
import com.turingalan.pokemon.ui.list.PokemonListScreen


@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun NavGraph() {
    val navController = rememberNavController()
    val startDestination = Route.List
    val backStackEntry by navController.currentBackStackEntryAsState()

    Scaffold(
        modifier = Modifier.fillMaxSize(),
        topBar = {
            AppTopBar(backStackEntry)
        }
    )
    {
        innerPadding ->

            val contentModifier = Modifier.consumeWindowInsets(innerPadding).padding(innerPadding)
            NavHost(
                navController = navController,
                startDestination = startDestination
            )
            {

                pokemonListDestination(contentModifier,
                    onNavigateToDetails = {
                        navController.navigateToPokemonDetails(it)
                        }
                    )
                pokemonDetailDestination(contentModifier)
            }
    }
}
```

#### `ui/theme/`
Archivos estándar para definir el tema de la aplicación (colores, tipografía).

##### `Color.kt`
Define las paletas de colores para los modos claro y oscuro.

```kotlin
package com.turingalan.pokemon.ui.theme

import androidx.compose.ui.graphics.Color

val Purple80 = Color(0xFFD0BCFF)
val PurpleGrey80 = Color(0xFFCCC2DC)
val Pink80 = Color(0xFFEFB8C8)

val Purple40 = Color(0xFF6650a4)
val PurpleGrey40 = Color(0xFF625b71)
val Pink40 = Color(0xFF7D5260)
```

##### `Type.kt`
Define los estilos de tipografía (`Typography`) usados en la aplicación.

```kotlin
package com.turingalan.pokemon.ui.theme

import androidx.compose.material3.Typography
import androidx.compose.ui.text.TextStyle
import androidx.compose.ui.text.font.FontFamily
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.sp

val Typography = Typography(
    bodyLarge = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.Normal,
        fontSize = 16.sp,
        lineHeight = 24.sp,
        letterSpacing = 0.5.sp
    )
)
```

##### `Theme.kt`
El `Composable` `PokemonTheme` que aplica todo el estilo (colores y tipografía) a la aplicación, manejando los modos claro/oscuro y el color dinámico de Android 12+.

```kotlin
package com.turingalan.pokemon.ui.theme

import android.app.Activity
import android.os.Build
import androidx.compose.foundation.isSystemInDarkTheme
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.darkColorScheme
import androidx.compose.material3.dynamicDarkColorScheme
import androidx.compose.material3.dynamicLightColorScheme
import androidx.compose.material3.lightColorScheme
import androidx.compose.runtime.Composable
import androidx.compose.ui.platform.LocalContext

private val DarkColorScheme = darkColorScheme(
    primary = Purple80,
    secondary = PurpleGrey80,
    tertiary = Pink80
)

private val LightColorScheme = lightColorScheme(
    primary = Purple40,
    secondary = PurpleGrey40,
    tertiary = Pink40
)

@Composable
fun PokemonTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    dynamicColor: Boolean = true,
    content: @Composable () -> Unit
) {
    val colorScheme = when {
        dynamicColor && Build.VERSION.SDK_INT >= Build.VERSION_CODES.S -> {
            val context = LocalContext.current
            if (darkTheme) dynamicDarkColorScheme(context) else dynamicLightColorScheme(context)
        }

        darkTheme -> DarkColorScheme
        else -> LightColorScheme
    }

    MaterialTheme(
        colorScheme = colorScheme,
        typography = Typography,
        content = content
    )
}
```