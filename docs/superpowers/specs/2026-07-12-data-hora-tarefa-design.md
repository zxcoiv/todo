# Data e Horário nas Tarefas — Design Spec

**Data:** 2026-07-12
**Contexto:** Evolução do app de lista de tarefas (ver [`2026-07-10-todo-list-design.md`](2026-07-10-todo-list-design.md)) — projeto pedagógico para alunos de 3º ano de Sistemas de Informação (FIAP).
**Objetivo:** Permitir que uma tarefa tenha data e horário (prazo), mantendo a possibilidade de tarefas avulsas (sem data/hora), sem quebrar a simplicidade didática do projeto.

---

## Requisitos

- Uma tarefa pode ter um prazo (`data + horário`) ou ser avulsa (sem prazo)
- Usuário escolhe data/hora com os pickers nativos do Material3 (`DatePicker`/`TimePicker`)
- Lista de tarefas mostra o prazo quando existir, e destaca visualmente tarefas atrasadas
- Tarefas com prazo aparecem primeiro na lista (mais próximas no topo); avulsas ficam depois
- Continua sem frameworks de DI, com `minSdk = 24` (sem depender de desugaring de `java.time`)
- Toda tela/componente reutilizável mantém `@Preview` para os novos estados visuais

---

## Camada de Dados

### Tarefa.kt
Novo campo:
```kotlin
val dataHora: Long? = null   // timestamp epoch millis; null = tarefa avulsa
```

### TarefaDao.kt
A query de listagem passa a ordenar por prazo, com avulsas por último:
```kotlin
@Query("""
    SELECT * FROM tarefas
    ORDER BY dataHora IS NULL, dataHora ASC, dataCriacao DESC
""")
fun listarTodas(): Flow<List<Tarefa>>
```
`dataHora IS NULL` avalia para `0` (tem data) ou `1` (avulsa) no SQLite — por isso tarefas com prazo vêm primeiro. Entre elas, ordena por proximidade (`dataHora ASC`). Entre as avulsas, mantém o critério atual (`dataCriacao DESC`).

`inserir`, `atualizar`, `deletar` não mudam de assinatura.

### TarefaDatabase.kt
- `@Database(entities = [Tarefa::class], version = 2, ...)`
- Builder ganha `.fallbackToDestructiveMigration()`
- Justificativa: banco local de estudo, sem dados de produção a preservar — evita introduzir o conceito de `Migration` explícita neste momento do curso

### Repository e ViewModel
Sem mudanças estruturais. `TarefaRepository` e `TarefaViewModel` já recebem/retornam `Tarefa` completa; o novo campo passa a reboque em `inserir`/`atualizar`.

---

## Formulário (`FormularioTarefaScreen.kt`)

### FormularioTarefaContent
Novo estado local:
```kotlin
var temDataHora by remember(dataHoraInicial) { mutableStateOf(dataHoraInicial != null) }
var dataMillis by remember(dataHoraInicial) { mutableStateOf(dataHoraInicial) }  // meia-noite do dia escolhido
var hora by remember(dataHoraInicial) { mutableStateOf<Int?>(...) }
var minuto by remember(dataHoraInicial) { mutableStateOf<Int?>(...) }
```

UI:
- `Switch` com label "Definir data e horário" controla `temDataHora`
- Quando `temDataHora == true`, exibe dois campos clicáveis (ex: `OutlinedButton` ou `AssistChip`):
  - "Data: 15/07/2026" (ou "Selecionar data" se ainda não escolhida) → abre `DatePickerDialog` do Material3, grava `selectedDateMillis`
  - "Hora: 14:30" (ou "Selecionar hora") → abre um `Dialog` com `TimePicker` do Material3 (Material3 não tem `TimePickerDialog` pronto — é um `AlertDialog`/`Dialog` customizado envolvendo `TimePicker`), grava `hour`/`minute`
- Botão "Salvar":
  - Habilitado apenas se `titulo` não vazio **e** (`temDataHora == false` OU (`dataMillis != null` E `hora != null` E `minuto != null`))
  - Ao salvar, combina `dataMillis + hora + minuto` num único `Long` via `java.util.Calendar` (compatível com `minSdk = 24`); se `temDataHora == false`, passa `null`
- Assinatura de callback muda para: `onSalvar: (titulo: String, descricao: String, dataHora: Long?) -> Unit`

### FormularioTarefaScreen (stateful)
Passa `dataHoraInicial = tarefaExistente?.dataHora` para o content, e monta `Tarefa(..., dataHora = dataHora)` no `inserir`/`atualizar`.

### Previews
- "Nova tarefa" (sem data)
- "Editar tarefa avulsa" (switch desligado)
- "Editar tarefa com data/hora" (switch ligado, campos preenchidos)

---

## Lista de Tarefas (`ListaTarefasScreen.kt`)

### Helper de formatação
Novo arquivo ou função utilitária:
```kotlin
fun formatarDataHora(millis: Long): String =
    SimpleDateFormat("dd/MM/yyyy 'às' HH:mm", Locale("pt", "BR")).format(Date(millis))
```

### TarefaItem
Quando `tarefa.dataHora != null`, exibe uma linha extra abaixo da descrição com o texto formatado:
- Se `tarefa.dataHora < System.currentTimeMillis()` e `!tarefa.concluida` → texto em `MaterialTheme.colorScheme.error`, negrito (tarefa atrasada)
- Caso contrário → texto no estilo padrão (`bodySmall`, cor normal)
- Tarefas avulsas (`dataHora == null`) não exibem essa linha, mesmo comportamento atual de `descricao` em branco

### Previews de TarefaItem
- "com prazo futuro"
- "atrasada"
- "avulsa" (comportamento atual, sem mudança)

---

## Dependências

Nenhuma dependência nova — `DatePicker`/`TimePicker` já fazem parte de `androidx.compose.material3`, já incluído no projeto.

---

## Conceitos Ensinados (adicionais)

- Campos `nullable` em entidades Room (`Long?`) e o efeito em queries (`IS NULL`)
- Ordenação condicional em SQL (`ORDER BY coluna IS NULL, ...`)
- `fallbackToDestructiveMigration()` e por que ele existe
- `DatePicker`/`TimePicker` do Material3 dentro de `Dialog`
- Combinação de estado de UI (`Switch` + campos condicionais) refletindo em um valor nullable do domínio
- `Calendar` para combinar data + hora em um timestamp único, evitando desugaring do `java.time` em `minSdk` baixo
