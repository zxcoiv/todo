# Data e Horário nas Tarefas — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Permitir que uma `Tarefa` tenha um prazo (`data + horário`) opcional, mantendo tarefas avulsas (sem prazo) plenamente suportadas, com a lista priorizando quem tem prazo mais próximo.

**Architecture:** Campo nullable `dataHora: Long?` na entidade Room; ordenação priorizando prazo no `TarefaDao`; utilitários puros de conversão de data/hora (sem `java.time`, compatível com `minSdk=24`); `FormularioTarefaScreen` ganha `Switch` + `DatePickerDialog`/`TimePicker` do Material3; `ListaTarefasScreen` exibe o prazo e destaca atrasos.

**Tech Stack:** Kotlin, Jetpack Compose (Material3), Room 2.7.1, `java.util.Calendar`/`SimpleDateFormat` (sem desugaring).

## Global Constraints

- `minSdk = 24` — não usar `java.time` (exigiria desugaring); usar `java.util.Calendar`/`SimpleDateFormat`
- Sem frameworks de DI (Hilt/Koin) — segue o padrão manual já existente (`ViewModelProvider.Factory`)
- Interface em Português
- Toda tela/componente reutilizável precisa de função(ões) `@Preview` para cada estado visual novo
- Testes instrumentados (`androidTest`) exigem emulador/dispositivo conectado — já é o padrão deste projeto (`TarefaDaoTest` existente); rodam via `./gradlew connectedDebugAndroidTest`
- Composables Compose deste projeto não têm testes de UI automatizados — verificação visual é feita via `@Preview` no Android Studio (convenção já estabelecida no spec original); este plano segue o mesmo padrão para as tarefas 3 e 4
- `dataHora: Long? = null` na entidade `Tarefa` — `null` significa tarefa avulsa
- Spec de referência: `docs/superpowers/specs/2026-07-12-data-hora-tarefa-design.md`

---

### Task 1: Camada de dados — campo `dataHora`, ordenação e migração

**Files:**
- Modify: `app/src/main/java/carreiras/com/github/todolist/data/Tarefa.kt`
- Modify: `app/src/main/java/carreiras/com/github/todolist/data/TarefaDao.kt`
- Modify: `app/src/main/java/carreiras/com/github/todolist/data/TarefaDatabase.kt`
- Test: `app/src/androidTest/java/carreiras/com/github/todolist/data/TarefaDaoTest.kt`

**Interfaces:**
- Produces: `Tarefa.dataHora: Long?` (default `null`); `TarefaDao.listarTodas(): Flow<List<Tarefa>>` agora ordena tarefas com prazo primeiro (mais próximas), avulsas por último; `TarefaDatabase` em `version = 2` com `fallbackToDestructiveMigration()`

- [ ] **Step 1: Escrever o teste que falha (ordenação com prazo)**

Adicionar ao final da classe `TarefaDaoTest`, antes da última chave `}`:

```kotlin
    @Test
    fun tarefasComPrazoAparecemAntesDeAvulsasEOrdenadasPorProximidade() = runTest {
        val agora = System.currentTimeMillis()
        dao.inserir(Tarefa(titulo = "Avulsa", descricao = ""))
        dao.inserir(Tarefa(titulo = "Prazo distante", descricao = "", dataHora = agora + 100_000))
        dao.inserir(Tarefa(titulo = "Prazo proximo", descricao = "", dataHora = agora + 10_000))

        val tarefas = dao.listarTodas().first()

        assertEquals("Prazo proximo", tarefas[0].titulo)
        assertEquals("Prazo distante", tarefas[1].titulo)
        assertEquals("Avulsa", tarefas[2].titulo)
    }
```

- [ ] **Step 2: Rodar o teste e confirmar que falha**

Run: `./gradlew connectedDebugAndroidTest --tests carreiras.com.github.todolist.data.TarefaDaoTest` (requer emulador/dispositivo conectado)
Expected: FALHA de compilação — `Tarefa` ainda não tem o parâmetro `dataHora`

- [ ] **Step 3: Adicionar o campo `dataHora` à entidade**

Editar `Tarefa.kt` — arquivo completo:

```kotlin
package carreiras.com.github.todolist.data

import androidx.room.Entity
import androidx.room.PrimaryKey

@Entity(tableName = "tarefas")
data class Tarefa(
    @PrimaryKey(autoGenerate = true) val id: Int = 0,
    val titulo: String,
    val descricao: String,
    val concluida: Boolean = false,
    val dataCriacao: Long = System.currentTimeMillis(),
    val dataHora: Long? = null
)
```

- [ ] **Step 4: Atualizar a query de listagem no DAO**

Editar `TarefaDao.kt` — arquivo completo:

```kotlin
package carreiras.com.github.todolist.data

import androidx.room.Dao
import androidx.room.Delete
import androidx.room.Insert
import androidx.room.Query
import androidx.room.Update
import kotlinx.coroutines.flow.Flow

@Dao
interface TarefaDao {

    @Query("""
        SELECT * FROM tarefas
        ORDER BY dataHora IS NULL, dataHora ASC, dataCriacao DESC
    """)
    fun listarTodas(): Flow<List<Tarefa>>

    @Insert
    suspend fun inserir(tarefa: Tarefa)

    @Update
    suspend fun atualizar(tarefa: Tarefa)

    @Delete
    suspend fun deletar(tarefa: Tarefa)
}
```

- [ ] **Step 5: Subir a versão do banco e adicionar `fallbackToDestructiveMigration`**

Editar `TarefaDatabase.kt` — arquivo completo:

```kotlin
package carreiras.com.github.todolist.data

import android.content.Context
import androidx.room.Database
import androidx.room.Room
import androidx.room.RoomDatabase

@Database(entities = [Tarefa::class], version = 2, exportSchema = false)
abstract class TarefaDatabase : RoomDatabase() {

    abstract fun tarefaDao(): TarefaDao

    companion object {
        @Volatile
        private var INSTANCE: TarefaDatabase? = null

        fun getDatabase(context: Context): TarefaDatabase {
            return INSTANCE ?: synchronized(this) {
                Room.databaseBuilder(
                    context.applicationContext,
                    TarefaDatabase::class.java,
                    "tarefas.db"
                ).fallbackToDestructiveMigration()
                    .build()
                    .also { INSTANCE = it }
            }
        }
    }
}
```

- [ ] **Step 6: Rodar o teste e confirmar que passa**

Run: `./gradlew connectedDebugAndroidTest --tests carreiras.com.github.todolist.data.TarefaDaoTest`
Expected: PASS — todos os testes da classe, incluindo o novo

- [ ] **Step 7: Commit**

```bash
git add app/src/main/java/carreiras/com/github/todolist/data/Tarefa.kt app/src/main/java/carreiras/com/github/todolist/data/TarefaDao.kt app/src/main/java/carreiras/com/github/todolist/data/TarefaDatabase.kt app/src/androidTest/java/carreiras/com/github/todolist/data/TarefaDaoTest.kt
git commit -m "feat: adicionar campo dataHora e ordenar tarefas por prazo"
```

---

### Task 2: Utilitário de conversão de data/hora

**Files:**
- Create: `app/src/main/java/carreiras/com/github/todolist/util/DataHoraUtil.kt`
- Test: `app/src/test/java/carreiras/com/github/todolist/util/DataHoraUtilTest.kt`

**Interfaces:**
- Consumes: nada (funções puras, `java.util.Calendar`/`SimpleDateFormat`)
- Produces:
  - `formatarDataHora(millis: Long): String` — formata timestamp local como `"dd/MM/yyyy 'às' HH:mm"`
  - `extrairDataDoDatePicker(millisUtc: Long): Triple<Int, Int, Int>` — `(ano, mes, dia)` a partir do valor UTC retornado pelo `DatePicker`
  - `paraMillisUtcDoDatePicker(ano: Int, mes: Int, dia: Int): Long` — inverso de `extrairDataDoDatePicker`, usado para pré-selecionar o `DatePicker` na edição
  - `combinarDataHora(ano: Int, mes: Int, dia: Int, hora: Int, minuto: Int): Long` — timestamp local final, salvo em `Tarefa.dataHora`

- [ ] **Step 1: Escrever os testes que falham**

Criar `app/src/test/java/carreiras/com/github/todolist/util/DataHoraUtilTest.kt`:

```kotlin
package carreiras.com.github.todolist.util

import org.junit.Assert.assertEquals
import org.junit.Test
import java.util.Calendar
import java.util.TimeZone

class DataHoraUtilTest {

    @Test
    fun formatarDataHoraRetornaTextoNoFormatoBrasileiro() {
        val millis = Calendar.getInstance().apply {
            clear()
            set(2026, Calendar.JULY, 15, 14, 30)
        }.timeInMillis

        assertEquals("15/07/2026 às 14:30", formatarDataHora(millis))
    }

    @Test
    fun extrairDataDoDatePickerLeAnoMesDiaEmUtc() {
        val millisUtc = Calendar.getInstance(TimeZone.getTimeZone("UTC")).apply {
            clear()
            set(2026, Calendar.JULY, 15)
        }.timeInMillis

        val (ano, mes, dia) = extrairDataDoDatePicker(millisUtc)

        assertEquals(2026, ano)
        assertEquals(Calendar.JULY, mes)
        assertEquals(15, dia)
    }

    @Test
    fun paraMillisUtcDoDatePickerEExtrairDataDoDatePickerSaoInversas() {
        val millisUtc = paraMillisUtcDoDatePicker(2026, Calendar.JULY, 15)

        val (ano, mes, dia) = extrairDataDoDatePicker(millisUtc)

        assertEquals(2026, ano)
        assertEquals(Calendar.JULY, mes)
        assertEquals(15, dia)
    }

    @Test
    fun combinarDataHoraGeraTimestampComComponentesCorretos() {
        val resultado = combinarDataHora(ano = 2026, mes = Calendar.JULY, dia = 15, hora = 14, minuto = 30)

        val calendario = Calendar.getInstance().apply { timeInMillis = resultado }
        assertEquals(2026, calendario.get(Calendar.YEAR))
        assertEquals(Calendar.JULY, calendario.get(Calendar.MONTH))
        assertEquals(15, calendario.get(Calendar.DAY_OF_MONTH))
        assertEquals(14, calendario.get(Calendar.HOUR_OF_DAY))
        assertEquals(30, calendario.get(Calendar.MINUTE))
    }
}
```

- [ ] **Step 2: Rodar os testes e confirmar que falham**

Run: `./gradlew testDebugUnitTest --tests carreiras.com.github.todolist.util.DataHoraUtilTest`
Expected: FALHA de compilação — `DataHoraUtil.kt` ainda não existe

- [ ] **Step 3: Implementar o utilitário**

Criar `app/src/main/java/carreiras/com/github/todolist/util/DataHoraUtil.kt`:

```kotlin
package carreiras.com.github.todolist.util

import java.text.SimpleDateFormat
import java.util.Calendar
import java.util.Date
import java.util.Locale
import java.util.TimeZone

fun formatarDataHora(millis: Long): String {
    val formato = SimpleDateFormat("dd/MM/yyyy 'às' HH:mm", Locale("pt", "BR"))
    return formato.format(Date(millis))
}

// O DatePicker do Material3 representa a data escolhida em milissegundos UTC
// (meia-noite do dia, fuso UTC), independente do fuso do dispositivo.
fun extrairDataDoDatePicker(millisUtc: Long): Triple<Int, Int, Int> {
    val calendario = Calendar.getInstance(TimeZone.getTimeZone("UTC")).apply { timeInMillis = millisUtc }
    return Triple(
        calendario.get(Calendar.YEAR),
        calendario.get(Calendar.MONTH),
        calendario.get(Calendar.DAY_OF_MONTH)
    )
}

fun paraMillisUtcDoDatePicker(ano: Int, mes: Int, dia: Int): Long {
    val calendario = Calendar.getInstance(TimeZone.getTimeZone("UTC"))
    calendario.clear()
    calendario.set(ano, mes, dia)
    return calendario.timeInMillis
}

fun combinarDataHora(ano: Int, mes: Int, dia: Int, hora: Int, minuto: Int): Long {
    val calendario = Calendar.getInstance()
    calendario.clear()
    calendario.set(ano, mes, dia, hora, minuto)
    return calendario.timeInMillis
}
```

- [ ] **Step 4: Rodar os testes e confirmar que passam**

Run: `./gradlew testDebugUnitTest --tests carreiras.com.github.todolist.util.DataHoraUtilTest`
Expected: PASS — 4 testes passando

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/carreiras/com/github/todolist/util/DataHoraUtil.kt app/src/test/java/carreiras/com/github/todolist/util/DataHoraUtilTest.kt
git commit -m "feat: adicionar utilitario de conversao de data/hora"
```

---

### Task 3: Formulário — Switch, DatePicker, TimePicker e validação

**Files:**
- Modify: `app/src/main/java/carreiras/com/github/todolist/ui/FormularioTarefaScreen.kt`

**Interfaces:**
- Consumes: `combinarDataHora`, `extrairDataDoDatePicker`, `paraMillisUtcDoDatePicker` de `carreiras.com.github.todolist.util` (Task 2); `Tarefa.dataHora: Long?` (Task 1)
- Produces: `FormularioTarefaContent` com novo parâmetro `dataHoraInicial: Long?` e callback `onSalvar: (titulo: String, descricao: String, dataHora: Long?) -> Unit`

- [ ] **Step 1: Substituir o conteúdo do arquivo**

Editar `FormularioTarefaScreen.kt` — arquivo completo:

```kotlin
package carreiras.com.github.todolist.ui

import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.padding
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.automirrored.filled.ArrowBack
import androidx.compose.material3.Button
import androidx.compose.material3.DatePicker
import androidx.compose.material3.DatePickerDialog
import androidx.compose.material3.ExperimentalMaterial3Api
import androidx.compose.material3.Icon
import androidx.compose.material3.IconButton
import androidx.compose.material3.OutlinedButton
import androidx.compose.material3.OutlinedTextField
import androidx.compose.material3.Scaffold
import androidx.compose.material3.Switch
import androidx.compose.material3.Text
import androidx.compose.material3.TextButton
import androidx.compose.material3.TimePicker
import androidx.compose.material3.TopAppBar
import androidx.compose.material3.rememberDatePickerState
import androidx.compose.material3.rememberTimePickerState
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.tooling.preview.Preview
import androidx.compose.ui.unit.dp
import androidx.compose.ui.window.Dialog
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import carreiras.com.github.todolist.data.Tarefa
import carreiras.com.github.todolist.util.combinarDataHora
import carreiras.com.github.todolist.util.extrairDataDoDatePicker
import carreiras.com.github.todolist.util.paraMillisUtcDoDatePicker
import carreiras.com.github.todolist.viewmodel.TarefaViewModel
import java.util.Calendar

@Composable
fun FormularioTarefaScreen(
    viewModel: TarefaViewModel,
    tarefaId: Int,
    onVoltar: () -> Unit
) {
    val tarefas by viewModel.tarefas.collectAsStateWithLifecycle()
    val tarefaExistente = remember(tarefas, tarefaId) {
        tarefas.find { it.id == tarefaId }
    }

    FormularioTarefaContent(
        isEdicao = tarefaId != 0,
        tituloInicial = tarefaExistente?.titulo ?: "",
        descricaoInicial = tarefaExistente?.descricao ?: "",
        dataHoraInicial = tarefaExistente?.dataHora,
        onSalvar = { titulo, descricao, dataHora ->
            if (tarefaId == 0) {
                viewModel.inserir(Tarefa(titulo = titulo, descricao = descricao, dataHora = dataHora))
            } else {
                tarefaExistente?.let {
                    viewModel.atualizar(it.copy(titulo = titulo, descricao = descricao, dataHora = dataHora))
                }
            }
            onVoltar()
        },
        onVoltar = onVoltar
    )
}

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun FormularioTarefaContent(
    isEdicao: Boolean,
    tituloInicial: String,
    descricaoInicial: String,
    dataHoraInicial: Long?,
    onSalvar: (titulo: String, descricao: String, dataHora: Long?) -> Unit,
    onVoltar: () -> Unit
) {
    var titulo by remember(tituloInicial) { mutableStateOf(tituloInicial) }
    var descricao by remember(descricaoInicial) { mutableStateOf(descricaoInicial) }
    var temDataHora by remember(dataHoraInicial) { mutableStateOf(dataHoraInicial != null) }

    val calendarioInicial = remember(dataHoraInicial) {
        dataHoraInicial?.let { Calendar.getInstance().apply { timeInMillis = it } }
    }
    var ano by remember(dataHoraInicial) { mutableStateOf(calendarioInicial?.get(Calendar.YEAR)) }
    var mes by remember(dataHoraInicial) { mutableStateOf(calendarioInicial?.get(Calendar.MONTH)) }
    var dia by remember(dataHoraInicial) { mutableStateOf(calendarioInicial?.get(Calendar.DAY_OF_MONTH)) }
    var hora by remember(dataHoraInicial) { mutableStateOf(calendarioInicial?.get(Calendar.HOUR_OF_DAY)) }
    var minuto by remember(dataHoraInicial) { mutableStateOf(calendarioInicial?.get(Calendar.MINUTE)) }

    var mostrarSeletorData by remember { mutableStateOf(false) }
    var mostrarSeletorHora by remember { mutableStateOf(false) }

    if (mostrarSeletorData) {
        val estadoDatePicker = rememberDatePickerState(
            initialSelectedDateMillis = if (ano != null) paraMillisUtcDoDatePicker(ano!!, mes!!, dia!!) else null
        )
        DatePickerDialog(
            onDismissRequest = { mostrarSeletorData = false },
            confirmButton = {
                TextButton(onClick = {
                    estadoDatePicker.selectedDateMillis?.let { millisUtc ->
                        val (a, m, d) = extrairDataDoDatePicker(millisUtc)
                        ano = a
                        mes = m
                        dia = d
                    }
                    mostrarSeletorData = false
                }) { Text("OK") }
            },
            dismissButton = {
                TextButton(onClick = { mostrarSeletorData = false }) { Text("Cancelar") }
            }
        ) {
            DatePicker(state = estadoDatePicker)
        }
    }

    if (mostrarSeletorHora) {
        val estadoTimePicker = rememberTimePickerState(
            initialHour = hora ?: 12,
            initialMinute = minuto ?: 0,
            is24Hour = true
        )
        Dialog(onDismissRequest = { mostrarSeletorHora = false }) {
            Column(
                modifier = Modifier.padding(24.dp),
                verticalArrangement = Arrangement.spacedBy(16.dp)
            ) {
                TimePicker(state = estadoTimePicker)
                Row(
                    modifier = Modifier.fillMaxWidth(),
                    horizontalArrangement = Arrangement.End
                ) {
                    TextButton(onClick = { mostrarSeletorHora = false }) { Text("Cancelar") }
                    TextButton(onClick = {
                        hora = estadoTimePicker.hour
                        minuto = estadoTimePicker.minute
                        mostrarSeletorHora = false
                    }) { Text("OK") }
                }
            }
        }
    }

    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text(if (isEdicao) "Editar Tarefa" else "Nova Tarefa") },
                navigationIcon = {
                    IconButton(onClick = onVoltar) {
                        Icon(
                            imageVector = Icons.AutoMirrored.Filled.ArrowBack,
                            contentDescription = "Voltar"
                        )
                    }
                }
            )
        }
    ) { padding ->
        Column(
            modifier = Modifier
                .fillMaxSize()
                .padding(padding)
                .padding(16.dp),
            verticalArrangement = Arrangement.spacedBy(16.dp)
        ) {
            OutlinedTextField(
                value = titulo,
                onValueChange = { titulo = it },
                label = { Text("Título") },
                modifier = Modifier.fillMaxWidth(),
                singleLine = true
            )
            OutlinedTextField(
                value = descricao,
                onValueChange = { descricao = it },
                label = { Text("Descrição") },
                modifier = Modifier.fillMaxWidth(),
                minLines = 3
            )
            Row(
                modifier = Modifier.fillMaxWidth(),
                verticalAlignment = Alignment.CenterVertically
            ) {
                Text("Definir data e horário", modifier = Modifier.weight(1f))
                Switch(checked = temDataHora, onCheckedChange = { temDataHora = it })
            }
            if (temDataHora) {
                Row(
                    modifier = Modifier.fillMaxWidth(),
                    horizontalArrangement = Arrangement.spacedBy(8.dp)
                ) {
                    OutlinedButton(
                        onClick = { mostrarSeletorData = true },
                        modifier = Modifier.weight(1f)
                    ) {
                        Text(
                            if (ano != null) String.format("%02d/%02d/%04d", dia, mes!! + 1, ano)
                            else "Selecionar data"
                        )
                    }
                    OutlinedButton(
                        onClick = { mostrarSeletorHora = true },
                        modifier = Modifier.weight(1f)
                    ) {
                        Text(
                            if (hora != null) String.format("%02d:%02d", hora, minuto)
                            else "Selecionar hora"
                        )
                    }
                }
            }
            Button(
                onClick = {
                    val dataHora = if (temDataHora) {
                        combinarDataHora(ano!!, mes!!, dia!!, hora!!, minuto!!)
                    } else {
                        null
                    }
                    onSalvar(titulo.trim(), descricao.trim(), dataHora)
                },
                modifier = Modifier.fillMaxWidth(),
                enabled = titulo.isNotBlank() && (!temDataHora || (ano != null && hora != null))
            ) {
                Text("Salvar")
            }
        }
    }
}

@Preview(showBackground = true, name = "Nova tarefa")
@Composable
private fun FormularioTarefaContentNovaPreview() {
    FormularioTarefaContent(
        isEdicao = false,
        tituloInicial = "",
        descricaoInicial = "",
        dataHoraInicial = null,
        onSalvar = { _, _, _ -> },
        onVoltar = {}
    )
}

@Preview(showBackground = true, name = "Editar tarefa avulsa")
@Composable
private fun FormularioTarefaContentEditarAvulsaPreview() {
    FormularioTarefaContent(
        isEdicao = true,
        tituloInicial = "Estudar Room",
        descricaoInicial = "Revisar anotações e DAO",
        dataHoraInicial = null,
        onSalvar = { _, _, _ -> },
        onVoltar = {}
    )
}

@Preview(showBackground = true, name = "Editar tarefa com data/hora")
@Composable
private fun FormularioTarefaContentEditarComPrazoPreview() {
    val calendario = Calendar.getInstance().apply { set(2026, Calendar.JULY, 15, 14, 30) }
    FormularioTarefaContent(
        isEdicao = true,
        tituloInicial = "Entregar atividade",
        descricaoInicial = "Upload no portal da FIAP",
        dataHoraInicial = calendario.timeInMillis,
        onSalvar = { _, _, _ -> },
        onVoltar = {}
    )
}
```

- [ ] **Step 2: Compilar e confirmar que não há erros**

Run: `./gradlew compileDebugKotlin`
Expected: BUILD SUCCESSFUL

- [ ] **Step 3: Verificar visualmente no Android Studio**

Abrir `FormularioTarefaScreen.kt` no Android Studio e checar as 3 `@Preview`: "Nova tarefa" (switch desligado), "Editar tarefa avulsa" (switch desligado, campos preenchidos), "Editar tarefa com data/hora" (switch ligado, botões mostrando "15/07/2026" e "14:30")

- [ ] **Step 4: Commit**

```bash
git add app/src/main/java/carreiras/com/github/todolist/ui/FormularioTarefaScreen.kt
git commit -m "feat: adicionar selecao de data e hora no formulario de tarefa"
```

---

### Task 4: Lista — exibição do prazo e destaque de atraso

**Files:**
- Modify: `app/src/main/java/carreiras/com/github/todolist/ui/ListaTarefasScreen.kt`

**Interfaces:**
- Consumes: `formatarDataHora(millis: Long): String` de `carreiras.com.github.todolist.util` (Task 2); `Tarefa.dataHora: Long?` (Task 1)
- Produces: nada consumido por outros arquivos (mudança visual isolada em `TarefaItem`)

- [ ] **Step 1: Adicionar imports necessários**

Em `ListaTarefasScreen.kt`, adicionar às importações existentes (mantendo ordem alfabética):

```kotlin
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.text.font.FontWeight
import carreiras.com.github.todolist.util.formatarDataHora
```

- [ ] **Step 2: Exibir o prazo em `TarefaItem`**

Substituir o bloco `Column(modifier = Modifier.weight(1f)) { ... }` dentro de `TarefaItem` por:

```kotlin
            Column(modifier = Modifier.weight(1f)) {
                Text(
                    text = tarefa.titulo,
                    style = MaterialTheme.typography.titleMedium,
                    textDecoration = if (tarefa.concluida) TextDecoration.LineThrough else TextDecoration.None
                )
                if (tarefa.descricao.isNotBlank()) {
                    Text(
                        text = tarefa.descricao,
                        style = MaterialTheme.typography.bodySmall,
                        maxLines = 1,
                        overflow = TextOverflow.Ellipsis
                    )
                }
                if (tarefa.dataHora != null) {
                    val atrasada = tarefa.dataHora < System.currentTimeMillis() && !tarefa.concluida
                    Text(
                        text = formatarDataHora(tarefa.dataHora),
                        style = MaterialTheme.typography.bodySmall,
                        color = if (atrasada) MaterialTheme.colorScheme.error else Color.Unspecified,
                        fontWeight = if (atrasada) FontWeight.Bold else FontWeight.Normal
                    )
                }
            }
```

- [ ] **Step 3: Adicionar previews de `TarefaItem` com prazo**

Adicionar ao final do arquivo, após `TarefaItemConcluidaPreview`:

```kotlin
@Preview(showBackground = true, name = "Item com prazo futuro")
@Composable
private fun TarefaItemComPrazoPreview() {
    val prazo = System.currentTimeMillis() + 86_400_000L
    TarefaItem(
        tarefa = Tarefa(id = 3, titulo = "Entregar atividade", descricao = "Upload no portal da FIAP", concluida = false, dataHora = prazo),
        onCheckedChange = {},
        onEditar = {},
        onDeletar = {}
    )
}

@Preview(showBackground = true, name = "Item atrasado")
@Composable
private fun TarefaItemAtrasadaPreview() {
    val prazo = System.currentTimeMillis() - 86_400_000L
    TarefaItem(
        tarefa = Tarefa(id = 4, titulo = "Entregar atividade", descricao = "Upload no portal da FIAP", concluida = false, dataHora = prazo),
        onCheckedChange = {},
        onEditar = {},
        onDeletar = {}
    )
}
```

- [ ] **Step 4: Compilar e confirmar que não há erros**

Run: `./gradlew compileDebugKotlin`
Expected: BUILD SUCCESSFUL

- [ ] **Step 5: Verificar visualmente no Android Studio**

Checar as novas previews "Item com prazo futuro" (texto normal, data de amanhã) e "Item atrasado" (texto vermelho e negrito, data de ontem)

- [ ] **Step 6: Commit**

```bash
git add app/src/main/java/carreiras/com/github/todolist/ui/ListaTarefasScreen.kt
git commit -m "feat: exibir prazo da tarefa na lista com destaque de atraso"
```

---

### Task 5: Verificação final

**Files:** nenhum (task de verificação, sem alterações de código)

**Interfaces:**
- Consumes: todo o trabalho das Tasks 1-4
- Produces: confirmação de que o app compila, os testes passam e o comportamento combinado (ordenação + formulário + lista) funciona de ponta a ponta

- [ ] **Step 1: Rodar todos os testes unitários**

Run: `./gradlew testDebugUnitTest`
Expected: BUILD SUCCESSFUL — inclui `DataHoraUtilTest` (Task 2)

- [ ] **Step 2: Rodar os testes instrumentados (se houver emulador/dispositivo conectado)**

Run: `./gradlew connectedDebugAndroidTest`
Expected: BUILD SUCCESSFUL — inclui `TarefaDaoTest` (Task 1), incluindo o novo teste de ordenação por prazo

- [ ] **Step 3: Build completo de debug**

Run: `./gradlew assembleDebug`
Expected: BUILD SUCCESSFUL

- [ ] **Step 4: Fluxo manual (opcional, se houver emulador disponível)**

Instalar o APK de debug, criar uma tarefa avulsa, uma com prazo futuro e uma com prazo já vencido; confirmar que a lista ordena com prazo futuro no topo, avulsa por último, e que a tarefa vencida aparece destacada em vermelho
