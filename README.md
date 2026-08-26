# To-Do List

Aplicativo Android de lista de tarefas desenvolvido em Kotlin para a atividade da FIAP. O objetivo da aplicação é permitir que o usuário **cadastre, edite, marque como concluída e exclua tarefas**, com os dados persistidos localmente no dispositivo, de forma que a lista permaneça disponível entre execuções do app.

## Tecnologias utilizadas

- **Kotlin** — linguagem principal do projeto.
- **Jetpack Compose** — construção de toda a interface de forma declarativa (telas, componentes e temas).
- **Room** — persistência local das tarefas em um banco de dados SQLite, abstraído por meio de entidade (`Tarefa`) e DAO (`TarefaDao`).
- **Coroutines/Flow** — operações assíncronas de acesso ao banco e observação reativa da lista de tarefas (`Flow`/`StateFlow`).
- **ViewModel** — retenção de estado da UI e orquestração das chamadas ao repositório, sobrevivendo a mudanças de configuração (ex.: rotação de tela).
- **Navigation Compose** — navegação entre as telas do app (lista e formulário) usando rotas com `NavHost`.

## Arquitetura

O projeto segue uma estrutura simples em camadas, inspirada em MVVM:

```
data/          -> Tarefa (entidade), TarefaDao, TarefaDatabase
repository/    -> TarefaRepository
viewmodel/     -> TarefaViewModel
ui/            -> ListaTarefasScreen, FormularioTarefaScreen
navigation/    -> AppNavigation
MainActivity.kt
```

### `TarefaRepository`

Localizado em [`TarefaRepository.kt`](app/src/main/java/souzaraphael/com/github/todolist/repository/TarefaRepository.kt), é a camada responsável por **abstrair a origem dos dados** para o restante do app. Ele recebe um `TarefaDao` no construtor e expõe:

- `tarefas: Flow<List<Tarefa>>` — repassa diretamente o `Flow` retornado por `dao.listarTodas()`, permitindo que quem observar receba automaticamente a lista atualizada sempre que o banco mudar.
- `inserir(tarefa)`, `atualizar(tarefa)` e `deletar(tarefa)` — funções `suspend` que apenas delegam a operação correspondente ao DAO.

Ou seja, o repositório não contém regra de negócio própria: sua responsabilidade é isolar o `ViewModel` dos detalhes do Room, servindo como ponto único de acesso aos dados (o que facilita, por exemplo, trocar a fonte de dados no futuro sem alterar a camada de UI/ViewModel).

### `TarefaViewModel`

Localizado em [`TarefaViewModel.kt`](app/src/main/java/souzaraphael/com/github/todolist/viewmodel/TarefaViewModel.kt), é responsável por **preparar e expor o estado da tela** e por **executar as ações do usuário** de forma segura em relação ao ciclo de vida:

- Converte o `Flow` do repositório em um `StateFlow<List<Tarefa>>` (`tarefas`) usando `stateIn`, com `SharingStarted.WhileSubscribed(5_000)` — a coleta do banco só fica ativa enquanto há um observador (ou até 5s após o último desaparecer) e sempre parte de `initialValue = emptyList()`.
- Expõe `inserir`, `atualizar` e `deletar`, cada uma disparando um `viewModelScope.launch { ... }` que chama o método correspondente do `TarefaRepository`, garantindo que as operações de banco (suspend) rodem em uma coroutine atrelada ao ciclo de vida do ViewModel.
- Fornece uma `factory` (companion object) que cria a instância do `TarefaViewModel` já injetando o `TarefaRepository`, construído a partir do `TarefaDao` obtido em `TarefaDatabase.getDatabase(context)`. Essa factory é o que permite criar o ViewModel manualmente (sem um framework de DI) a partir do `Context` da aplicação.

### `ListaTarefasScreen`

Localizado em [`ListaTarefasScreen.kt`](app/src/main/java/souzaraphael/com/github/todolist/ui/ListaTarefasScreen.kt), é a tela inicial que exibe a lista de tarefas:

- Observa o estado com `viewModel.tarefas.collectAsStateWithLifecycle()`, que coleta o `StateFlow` do ViewModel de forma consciente do ciclo de vida (pausa a coleta quando a tela não está visível) e converte o valor em um `State` do Compose (`by tarefas`), fazendo a UI recompor automaticamente sempre que a lista mudar.
- Repassa esse estado para `ListaTarefasContent` (função stateless, mais fácil de testar/pré-visualizar), junto com callbacks que traduzem interações da UI em ações do ViewModel:
  - Ao marcar/desmarcar o `Checkbox` de uma tarefa, chama `viewModel.atualizar(tarefa.copy(concluida = concluida))`.
  - Ao clicar no ícone de lixeira, chama `viewModel.deletar(tarefa)`.
  - Ao clicar no `FloatingActionButton` (`onNovaTarefa`) ou em um item da lista (`onEditarTarefa`), aciona callbacks de navegação recebidos como parâmetro (não chama o ViewModel diretamente), que são conectados à navegação real em `AppNavigation`.

### `FormularioTarefaScreen`

Localizado em [`FormularioTarefaScreen.kt`](app/src/main/java/souzaraphael/com/github/todolist/ui/FormularioTarefaScreen.kt), é usado tanto para **cadastrar** quanto para **editar** uma tarefa, reaproveitando o mesmo layout:

- Recebe um `tarefaId: Int`. O valor `0` indica **cadastro de uma nova tarefa**; qualquer outro valor é tratado como o **id de uma tarefa existente**, usado para edição.
- Observa `viewModel.tarefas` e usa `remember(tarefas, tarefaId)` para localizar a tarefa correspondente (`tarefas.find { it.id == tarefaId }`), preenchendo os campos de título e descrição com os valores atuais quando existir (`tarefaExistente`).
- Calcula `isEdicao = tarefaId != 0`, usado para adaptar a UI (título da `TopAppBar`: "Nova Tarefa" ou "Editar Tarefa").
- Ao salvar (`onSalvar`), decide a ação com base no `tarefaId`:
  - Se `tarefaId == 0`, chama `viewModel.inserir(...)` criando uma nova `Tarefa`.
  - Caso contrário, chama `viewModel.atualizar(...)` a partir de `tarefaExistente?.copy(...)`, preservando o `id` e demais campos (como `concluida`) e alterando apenas título/descrição.
- Após salvar, chama `onVoltar()` para retornar à tela anterior.

### Rotas em `AppNavigation`

Localizado em [`AppNavigation.kt`](app/src/main/java/souzaraphael/com/github/todolist/navigation/AppNavigation.kt), configura o grafo de navegação com `NavHost` e duas rotas:

- **`"lista"`** (rota inicial) — exibe `ListaTarefasScreen`, passando o `TarefaViewModel` (compartilhado entre as telas) e dois callbacks de navegação:
  - `onNovaTarefa` navega para `"formulario/0"`, ou seja, abre o formulário em modo de cadastro (id `0`).
  - `onEditarTarefa(id)` navega para `"formulario/$id"`, abrindo o formulário já com o id da tarefa selecionada.
- **`"formulario/{tarefaId}"`** — recebe o argumento `tarefaId` via `backStackEntry.arguments`, convertendo a `String` da rota para `Int` (`?.toInt() ?: 0`), e passa esse valor para `FormularioTarefaScreen`. É exatamente esse id que a tela de formulário usa para diferenciar cadastro (`0`) de edição (id existente). O callback `onVoltar` chama `navController.popBackStack()` para retornar à lista.

Dessa forma, o **id da tarefa é propagado pela própria rota de navegação** (como parte da URL de navegação), e não por um estado compartilhado à parte — o formulário volta a buscar os dados da tarefa a partir do `TarefaViewModel` usando esse id.

### `MainActivity`

Localizado em [`MainActivity.kt`](app/src/main/java/souzaraphael/com/github/todolist/MainActivity.kt), é o ponto de entrada do app:

- Em `onCreate`, chama `enableEdgeToEdge()` e define o conteúdo da Activity com `setContent`.
- Dentro do tema `FiaptodolistTheme`, cria o `TarefaViewModel` usando a função `viewModel(factory = TarefaViewModel.factory(applicationContext))`, do Compose (`androidx.lifecycle.viewmodel.compose.viewModel`) — isso garante que o ViewModel seja criado (usando a factory que injeta o repositório/banco) e retido corretamente pelo Android durante o ciclo de vida da Activity.
- Por fim, chama `AppNavigation(viewModel = viewModel)`, iniciando a navegação do app e injetando essa mesma instância do ViewModel, que passa a ser compartilhada entre as telas de lista e formulário.

## Como executar o projeto

**Pré-requisitos:**
- Android Studio (versão recente, compatível com AGP 9.1.0 / Kotlin 2.2.10).
- JDK 21.
- Um emulador Android configurado ou um dispositivo físico com **Android 7.0 (API 24)** ou superior.

**Passos:**

1. Clone ou abra a pasta do projeto no Android Studio.
2. Aguarde a sincronização do Gradle (o Android Studio baixa as dependências automaticamente).
3. Selecione um emulador ou conecte um dispositivo físico.
4. Clique em **Run ▶** (ou use o atalho `Shift + F10`) para compilar e instalar o app.

Alternativamente, pela linha de comando, na raiz do projeto:

```bash
./gradlew installDebug
```

(no Windows, use `gradlew.bat installDebug`).

Ao abrir o app, a tela inicial exibe a lista de tarefas (vazia na primeira execução). Use o botão flutuante (**+**) para cadastrar uma nova tarefa, toque em uma tarefa existente para editá-la, marque o checkbox para concluí-la e use o ícone de lixeira para excluí-la.


# Evidencias

#### Tela inicial

![Tela inicial](docs/evidencias/Tela%20inicial.png)

#### Cadastro de tarefa

![Cadastro de tarefa](docs/evidencias/Cadastro%20de%20tarefa.png)

#### Tarefa na tela inicial

![Tarefa na tela inicial](docs/evidencias/Tarefa%20na%20tela%20inicial.png)

#### Edicao de tarefa

![Edicao de tarefa](docs/evidencias/Edicao%20de%20tarefa.png)

#### Tarefa concluida

![Tarefa concluida](docs/evidencias/Tarefa%20concluida.png)

#### Tarefa excluida

![Tarefa excluida](docs/evidencias/Tarefa%20excluida.png)

#### Projeto sendo executado

![Projeto sendo executado](docs/evidencias/Projeto%20sendo%20executado.png)

