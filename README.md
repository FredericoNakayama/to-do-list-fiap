# To-Do List — Android (FIAP)

Aplicativo Android de lista de tarefas, desenvolvido como atividade individual a partir do projeto base fornecido em aula. O app permite listar, cadastrar, editar, concluir e excluir tarefas, com navegação entre a tela de lista e o formulário, usando persistência local com Room.

## Tecnologias utilizadas

- Kotlin
- Jetpack Compose (Material 3)
- Room (persistência local em SQLite)
- Coroutines / Flow
- ViewModel (androidx.lifecycle)
- Navigation Compose

## Arquitetura

O projeto segue o padrão **MVVM com Repository**, em que cada camada só conhece a camada imediatamente abaixo dela:

```
Compose UI  →  ViewModel  →  Repository  →  DAO  →  Room Database
```

### TarefaRepository

Isola o resto do app do detalhe de que a fonte de dados é o Room. Recebe o `TarefaDao` no construtor e expõe:
- uma propriedade `tarefas: Flow<List<Tarefa>>`, que apenas repassa o `Flow` retornado pelo DAO;
- três funções `suspend` (`inserir`, `atualizar`, `deletar`) que delegam diretamente para o DAO.

É uma camada intencionalmente fina: se um dia a fonte de dados mudasse, só o Repository precisaria mudar.

### TarefaViewModel

É a camada que a UI enxerga — não conhece Room nem SQL, só o Repository. Duas decisões importantes:

- **StateFlow em vez de Flow**: a UI do Compose precisa de um valor sempre disponível para desenhar a tela, mesmo antes de qualquer emissão. Por isso o `Flow` do Repository é convertido com `.stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())`.
- **Factory manual**: como o projeto não usa Hilt/Koin, a própria `TarefaViewModel` expõe, num `companion object`, uma função `factory(context)` que monta manualmente a cadeia `TarefaDatabase → TarefaDao → TarefaRepository → TarefaViewModel`.

As operações de escrita (`inserir`, `atualizar`, `deletar`) rodam dentro de `viewModelScope.launch { ... }`, já que as funções do Repository são `suspend`.

### ListaTarefasScreen

Dividida em duas funções `@Composable`, seguindo a convenção do projeto (necessária porque o `@Preview` não consegue instanciar uma ViewModel real):

- `ListaTarefasScreen` (stateful) coleta `viewModel.tarefas` com `collectAsStateWithLifecycle()` e repassa a lista e os callbacks para a versão content. É a versão usada pela navegação.
- `ListaTarefasContent` (previewable) recebe só a lista de tarefas e lambdas simples, e desenha a `LazyColumn`. Cada item mostra título, descrição, um `Checkbox` para marcar como concluída (que dispara `onCheckedChange`) e um botão de excluir (`onDeletar`); tocar no item chama `onEditarTarefa(tarefa.id)`.

### FormularioTarefaScreen

Também dividida em stateful/content, e atende tanto cadastro quanto edição a partir de um único parâmetro: `tarefaId: Int`. Se `tarefaId == 0`, é uma tarefa nova; qualquer outro valor é o id de uma tarefa existente. A versão stateful procura essa tarefa no `StateFlow` coletado (`tarefas.find { it.id == tarefaId }`) para decidir os valores iniciais dos campos e o título da tela ("Nova Tarefa" ou "Editar Tarefa"). Ao salvar, ela decide entre `viewModel.inserir(...)` e `viewModel.atualizar(...)` usando essa mesma regra, e volta para a lista em seguida.

### AppNavigation

Define um `NavHost` com duas rotas:

- `"lista"` → `ListaTarefasScreen`;
- `"formulario/{tarefaId}"` → `FormularioTarefaScreen`, recebendo o id como argumento de rota (`0` para nova tarefa, ou o id de uma tarefa existente para edição).

O botão de nova tarefa navega para `"formulario/0"`; tocar numa tarefa existente navega para `"formulario/{id}"`; o botão voltar do formulário chama `popBackStack()`.

### MainActivity

Cria a `TarefaViewModel` com `viewModel(factory = TarefaViewModel.factory(applicationContext))` e, dentro do tema do app, chama `AppNavigation(viewModel = viewModel)` — substituindo a tela de exemplo ("Hello Android") gerada pelo template do Android Studio.

## Como executar

1. Abra o projeto no Android Studio.
2. Aguarde o Gradle sincronizar as dependências (Room, Navigation Compose, ViewModel Compose já estão declaradas no `libs.versions.toml`).
3. Rode em um emulador ou dispositivo físico (▶ **Run 'app'**).

## Evidências

Capturas de tela do funcionamento estão em [`docs/evidencias`](docs/evidencias):

- Tela inicial com a lista de tarefas em execução
- Cadastro de uma nova tarefa
- Tarefa cadastrada aparecendo na lista
- Edição de uma tarefa existente
- Tarefa marcada como concluída
- Exclusão de uma tarefa
- Navegação entre a lista e o formulário
- Build/execução do projeto sem erros
