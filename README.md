# To-Do List

![Kotlin](https://img.shields.io/badge/Kotlin-2.2.10-7F52FF?logo=kotlin&logoColor=white)
![Jetpack Compose](https://img.shields.io/badge/Jetpack%20Compose-BOM%202026.02.01-4285F4?logo=jetpackcompose&logoColor=white)
![Room](https://img.shields.io/badge/Room-2.7.1-3DDC84?logo=android&logoColor=white)
![minSdk](https://img.shields.io/badge/minSdk-24-blue)
![License](https://img.shields.io/badge/license-MIT-lightgrey)

Aplicativo Android de lista de tarefas (to-do list) desenvolvido como projeto didático para a disciplina de Sistemas de Informação da FIAP. Construído com **Kotlin**, **Jetpack Compose** e **Room**, seguindo o padrão arquitetural **MVVM**.

## Funcionalidades

- Criar, editar e excluir tarefas
- Marcar tarefas como concluídas
- Definir data e horário de prazo para uma tarefa
- Lista ordenada por prazo, com destaque visual para tarefas atrasadas
- Persistência local dos dados (SQLite via Room) — as tarefas continuam disponíveis após fechar o app

## Tecnologias

| Camada | Tecnologia |
|---|---|
| Linguagem | Kotlin |
| UI | Jetpack Compose + Material 3 |
| Navegação | Navigation Compose |
| Persistência | Room (SQLite) |
| Assincronismo | Kotlin Coroutines + Flow |
| Testes | JUnit, Espresso, Compose UI Test |

## Arquitetura

O projeto segue o padrão **MVVM (Model-View-ViewModel)**, sem uso de frameworks de injeção de dependência (Hilt/Koin) — a criação de objetos é feita por uma fábrica manual, propositalmente, para fins didáticos.

```
app/src/main/java/carreiras/com/github/todolist/
├── data/          # Model — Entity, DAO e configuração do banco (Room)
├── repository/     # Model — abstrai o acesso a dados para a ViewModel
├── viewmodel/       # ViewModel — estado observável (StateFlow) e regras de apresentação
├── ui/             # View — telas em Jetpack Compose
├── navigation/       # View — rotas e navegação entre telas
└── util/           # Funções puras auxiliares (formatação/conversão de data e hora)
```

Uma explicação detalhada e comparada da arquitetura (MVC, MVP, MVI e MVVM), com trechos de código reais do projeto e diagramas, está em **[ARQUITETURA-MVVM.md](ARQUITETURA-MVVM.md)**.

## Pré-requisitos

- [Android Studio](https://developer.android.com/studio) (Ladybug ou mais recente)
- JDK 11+
- Um emulador Android (API 24+) ou dispositivo físico

## Como executar

1. Clone o repositório:
   ```bash
   git clone https://github.com/carreiras/fiap-to-do-list.git
   ```
2. Abra a pasta do projeto no Android Studio e aguarde a sincronização do Gradle.
3. Selecione um emulador ou conecte um dispositivo físico.
4. Clique em **Run** ▶ ou execute pelo terminal:
   ```bash
   ./gradlew installDebug
   ```

## Testes

O projeto tem testes unitários (JVM) e instrumentados (Android):

```bash
# Testes unitários (ex: DataHoraUtilTest)
./gradlew test

# Testes instrumentados (ex: TarefaDaoTest), requer emulador/dispositivo conectado
./gradlew connectedAndroidTest
```

## Estrutura de navegação

O app tem duas telas, conectadas via Navigation Compose e compartilhando a mesma instância de `TarefaViewModel`:

- **Lista de tarefas** — tela inicial, exibe todas as tarefas ordenadas por prazo.
- **Formulário** — criação/edição de uma tarefa, incluindo seleção opcional de data e horário.

## Autor

Ewerton Carreira

## Licença

Este projeto é distribuído sob a licença MIT.
