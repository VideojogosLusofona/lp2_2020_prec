<!--
Projeto de Recurso de Linguagens de Programação II 2020/2021 (c) by Nuno Fachada

Projeto de Recurso de Linguagens de Programação II 2020/2021 is licensed under a
Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International License.

You should have received a copy of the license along with this
work. If not, see <http://creativecommons.org/licenses/by-nc-sa/4.0/>.
-->

# Projeto de Recurso de Linguagens de Programação II 2020/2021

## Introdução

Os grupos devem implementar o modelo de simulação [Pedra, Papel e Tesoura]
na forma de duas aplicações C#, uma em consola, outra em Unity.

Este modelo explora um ecossistema de três espécies, pedras (cor azul), papéis
(cor verde) e tesouras (cor vermelha), que competem por espaço no mundo de
simulação. As interações entre as espécies são baseadas no jogo
[Pedra, Papel e Tesoura], ou seja, tesoura vence papel, papel vence pedra, e
pedra vence tesoura. Os organismos competem com os seus vizinhos, movem-se no
mundo e reproduzem-se. Estas interações resultam em padrões de espiral cujo
tamanho e estabilidade dependem dos parâmetros da simulação.

O modelo está implementado como [exemplo][RPSNetLogo] no software [NetLogo]
(aberto e gratuito, também corre diretamente no _browser_), implementação esta
que pode e deve ser usada como termo de comparação ao projeto desenvolvido.
A Secção [Descrição](#descrição) apresenta o modelo em detalhe.

O jogo deve ser implementado em consola (.NET Core 3.1) e em Unity. A lógica,
regras e dados da simulação (o chamado **modelo**), devem ser completamente
independentes tanto da interface de consola (`WriteLines`, `ReadLines`), como
do Unity (ver Secção [MVC pattern](#mvc-pattern)). Este modelo deve ser
obrigatoriamente o mesmo em ambas as implementações, devendo ser partilhado de
uma forma que não implique copiar os ficheiros de um lado para o outro. Existe
forma de fazer isto ao nível do Git usando [sub-módulos][submodules] (ver
Secção [Sugestões para o uso de sub-módulos em
Git](#sugestões-para-o-uso-de-sub-módulos-em-git)).

Ambas as versões devem ser multi-plataforma, ou seja, devem funcionar em Linux
e macOS, pelo que devem ser evitados métodos e classes que apenas funcionam em
Windows.

Os grupos podem ter entre 1 a 3 elementos.

## Descrição

### O modelo de simulação

A simulação corre numa grelha com dimensões (`x`, `y`) toroidal com vizinhança
de [Von Neumann] de raio 1. Toroidal significa que a grelha "dá a volta" na
vertical e na horizontal, ou seja, na prática não tem paredes.

Cada célula da grelha pode estar ocupada por uma das três espécies ou estar
vazia (neste caso assume a cor de fundo do terminal). A simulação funciona por
turnos e em cada turno podem acontecer os seguintes eventos:

1. Evento de **Troca/Movimento**:
   * Duas células vizinhas são aleatoriamente escolhidas.
   * O estado de cada uma das células passa a ser o estado da célula vizinha
     (ou seja, as células trocam de estado).
2. Evento de **Reprodução**:
   * Duas células vizinhas são aleatoriamente escolhidas.
   * Se uma das células for vazia passa a ser ocupada por um elemento da
     espécie na célula vizinha.
   * Nada acontece se nenhuma das células ou ambas as células forem vazias.
3. Evento de **Seleção**:
   * Duas células vizinhas são aleatoriamente escolhidas.
   * Os seus ocupantes competem um com o outro segundo as regras do
     [Pedra, Papel e Tesoura].
   * A célula do vizinho perdedor torna-se vazia.
   * Caso alguma das células escolhidas seja vazia, nada acontece.

A simulação tem cinco parâmetros:

* `xdim` - Número inteiro que representa a dimensão horizontal da grelha
  de simulação.
* `ydim` - Número inteiro que representa a dimensão vertical da grelha
  de simulação.
* `swap_rate_exp`, número real entre -1.0 e 1.0, que representa a taxa dos
  eventos de troca/movimento.
* `repr_rate_exp`, número real entre -1.0 e 1.0, que representa a taxa dos
  eventos de reprodução.
* `selc_rate_exp`, número real entre -1.0 e 1.0, que representa a taxa dos
  eventos de seleção.

O número exato de cada tipo de evento em cada turno é determinado
aleatoriamente através de uma [distribuição de Poisson], que expressa a
probabilidade de uma série de eventos ocorrer num certo período de tempo. Uma
vez que este é um projeto de programação, não é exigido aos alunos que
compreendam a fundo esta distribuição, apenas que consigam implementar um
método chamado `Poisson()` que aceite um valor real `λ` (média da
distribuição) e devolva um número inteiro aleatório obtido através desta
distribuição. A secção
[Gerar inteiros aleatórios a partir da distribuição de Poisson](#gerar-inteiros-aleatórios-a-partir-da-distribuição-de-poisson) discute
possíveis abordagens. Uma possível declaração deste método é a seguinte:

```cs
private int Poisson(double lambda);
```

O número exato de eventos em cada turno é então obtido através do método
`Poisson()`, em que `λ` é dado por:

```cs
double lambda = (xdim * ydim / 3.0) * Math.Pow(10, rate_exp);
```

A variável `rate_exp` pode ser `swap_rate_exp`, `repr_rate_exp` ou
`selc_rate_exp`, dependendo do tipo de evento em questão. Por exemplo, o
número de trocas/movimentos num dado turno pode ser dado por:

```cs
// Obter o lambda λ, valor médio das trocas/movimentos
// Notar que este valor é constante ao longo da simulação
double lambdaSwap = (xdim * ydim / 3.0) * Math.Pow(10, swap_rate_exp);

// Obter o número de trocas/movimentos a efetuar no turno atual
int numSwaps = Poisson(lambdaSwap);
```

Uma vez obtido o número de cada tipo de eventos para o turno atual, os mesmos
devem ser individualmente colocados numa lista. Essa lista deve ser então
embaralhada (ver secção
[Embaralhar uma lista ou _array_](#embaralhar-uma-lista-ou-array)), e
finalmente percorrida, de modo a que cada evento seja executado. A ideia
é que os eventos sejam executados numa ordem aleatória. Por exemplo, se
num dado turno serão executadas 3 trocas, 4 reproduções e 2 seleções
(valores obtidos aleatoriamente a partir da distribuição de Poisson), a
lista com estes eventos deverá ter inicialmente o seguinte conteúdo:

```
Swap
Swap
Swap
Reproduction
Reproduction
Reproduction
Reproduction
Selection
Selection
```

Após o embaralhamento a ordem dos conteúdos é randomizada:

```
Reproduction
Swap
Reproduction
Selection
Reproduction
Selection
Swap
Swap
Reproduction
```

É nesta fase, após o embaralhamento, que a lista deve ser percorrida, e cada
um dos eventos executado para o turno atual.

A visualização deve ser atualizada após todos os eventos de dado turno terem
sido executados. A imagem/vídeo em baixo mostra uma implementação de consola em
C# com dimensões 300 x 70 e todos os `rates_exp` colocados a zero.

[![Rock Paper Scissors](https://img.youtube.com/vi/uYiQg-rkG6A/hqdefault.jpg)](https://youtu.be/uYiQg-rkG6A)

Resumindo, a simulação é executada de acordo com os seguintes passos:

1. Criar o mundo de simulação, cada célula inicializada aleatoriamente (pedra,
   papel, tesoura, vazia).
2. Determinar número de eventos de troca/movimento, reprodução e seleção,
   a partir da distribuição de Poisson tal como explicado em cima.
3. Colocar esses eventos numa lista, um a um.
4. Embaralhar a lista.
5. Percorrer a lista e executar esses eventos, um a um.
6. Limpar a lista.
7. Atualizar visualização.
8. Se utilizador tiver entretanto dado indicação para terminar a simulação,
   parar a execução da mesma. Caso contrário voltar para o ponto 2.

### Requisitos da versão em consola

O programa de consola deve aceitar os parâmetros da simulação como argumentos da
linhas de comandos, como indicado no seguinte exemplo (que assume que o projeto
se chama `ConsoleApp`):

```
dotnet run -p ConsoleApp -- 100 40 -0.02 0.75 0.00
```

A opção `--` serve para separar entre as opções do comando `dotnet` e as opções
do programa a ser executado, neste caso a nossa simulação. As opções seguintes
devem ser dadas por ordem e têm o seguinte significado:

* 1ª opção: `xdim`, número inteiro maior ou igual que 2.
* 2ª opção, `ydim`, número inteiro maior ou igual que 2.
* 3ª opção, `swap-rate-exp`, número real entre -1.0 e 1.0.
* 4ª opção, `repr-rate-exp`, número real entre -1.0 e 1.0.
* 5ª opção, `selc-rate-exp`, número real entre -1.0 e 1.0.

Caso não seja dado o número correto de opções, ou caso alguma das opções não
seja válida, o programa deve terminar com uma mensagem de erro apropriada.
Podem existir problemas no _parsing_ dos valores reais devido ao separador
decimal ser uma vírgula e não um ponto na língua portuguesa (ver secção
[_Parsing_ de números reais](#parsing-de-números-reais)). Se as opções forem
corretas, a simulação começa imediatamente.
<!-- , não existindo quaisquer paragens ou
demoras entre turnos. A simulação deve correr o mais rapidamente possível. -->

A simulação entra em pausa se o utilizador pressionar a barra de espaços, e
continua a sua execução se o utilizador carregar na barra de espaços novamente.
A simulação termina quando o utilizador pressionar a tecla `Escape`. É possível
verificar se alguma tecla for pressionada através da propriedade
[`Console.KeyAvailable`](https://docs.microsoft.com/dotnet/api/system.console.keyavailable),
evitando deste modo que o programa fique preso à espera de uma tecla. Em
alternativa, podem implementar um _game loop_ dinâmico tal como dado nas aulas,
existindo uma _thread_ separada a capturar as teclas pressionadas (esta solução
será valorizada, pois está mais em linha com a matéria de LP2).

Soluções mais eficientes (que executem a simulação mais rapidamente) serão
bonificadas na nota. Todas as otimizações implementadas devem ser mencionadas
no relatório. Duas sugestões mutuamente exclusivas (i.e., não podem ser
usadas em conjunto):

* Na visualização atualizar apenas as células que foram modificadas num turno e
  não todas.
* Atualizar o mundo de uma só vez (com um único `Console.Write()`), pré-criando
  uma _string_ com todos os seus conteúdos (isto requer o uso de [sequências de
  _escape_ ANSI](https://stackoverflow.com/questions/4842424/list-of-ansi-color-escape-sequences),
  indo além da formatação de cores disponível na classe `Console`). A forma mais
  eficiente de "ir construíndo" uma _string_ é através de um
  [`StringBuilder`](https://docs.microsoft.com/dotnet/api/system.text.stringbuilder),
  e não com concatenação.

Estas otimizações no projeto de consola só devem ser efetuadas após a simulação
estar completamente funcional, e é perfeitamente possível ter nota máxima, ou
perto disso, sem a implementação das mesmas.

A aplicação de consola deve funcionar em Windows, macOS e Linux. A melhor
estratégia para garantir que assim seja é testar o jogo em Linux (e.g., numa
máquina virtual). Algumas instruções incompatíveis com macOS e Linux são, por
exemplo:

* [Console.Beep()](https://docs.microsoft.com/dotnet/api/system.console.beep)
* [Console.SetBufferSize()](https://docs.microsoft.com/dotnet/api/system.console.setbuffersize)
* [Console.SetWindowPosition()](https://docs.microsoft.com/dotnet/api/system.console.setwindowposition)
* [Console.SetWindowSize()](https://docs.microsoft.com/dotnet/api/system.console.setwindowsize)
* Entre outras.

As instruções que só funcionam em Windows têm a seguinte indicação na sua
documentação:

![The current operating system is not Windows.](img/notsupported.png "The current operating system is not Windows.")

### Requisitos da versão Unity

A versão Unity deve ter um UI que permita definir os 5 parâmetros da simulação,
bem como botões para _Play_, _Pause_, _Stop_ e _Quit_. Os 5 parâmetros só podem
ser alterados quando a simulação estiver no estado _Stop_, e devem estar
limitados ao intervalo {2, 3, ..., 500} para os parâmetros `xdim` e  `ydim`, e
ao intervalo \[-1.0, 1.0] para os restantes parâmetros. A aplicação começa no
estado _Stop_, sendo necessário o utilizador clicar no _Play_ para dar início à
simulação. Podem adicionar outros elementos ao UI, caso considerem-nos úteis,
como por exemplo um _slider_ para controlar a velocidade da simulação. Este UI
deve pertencer à própria aplicação, e não ao editor do Unity.

Existem várias formas de criar a animação da simulação, das quais podemos
destacar três:

* Usar uma
  [`RawImage`](https://docs.unity3d.com/2018.3/Documentation/ScriptReference/UI.RawImage.html)
  e atualizar os píxeis da respetiva textura. É provavelmente a opção mais
  simples.
* Usar um [`TileMap`](https://docs.unity3d.com/Manual/class-Tilemap.html).
* Usar [_shaders_](https://docs.unity3d.com/Manual/Shaders.html). De longe a
  opção mais eficiente, mas provavelmente a mais complexa.

O uso de _shaders_ será bonificado, mas apenas se tiverem sido bem
implementados. É perfeitamente possível ter nota máxima usando apenas uma
`RawImage`. Notem que a primeira otimização indicada para o projeto de consola
também poderá ser válida para o projeto Unity.

## Considerações e sugestões técnicas

### MVC pattern

O [Model-View-Controller (MVC)][MVC] *design pattern* é o ponto de partida para
a realização deste projeto. Alguns recursos que podem utilizar para compreender
como funciona este _pattern_:

* [Video tutorial sobre como criar projetos MVC em consola e
  Unity][tutorial-video] (essencial).
* [MVCExample: código desse mesmo tutorial][tutorial-code].
* [ColorShapeLinks competition][ia-simplexity]: neste projeto o modelo é
  completamente independente do Unity ou da consola, existindo versões concretas
  em ambos os sistemas.
* [Jogo do Galo em consola implementado com MVC][ttt-mvc] (apenas com matéria
  de LP1, há coisas que podem ser melhoradas com matéria de LP2).
* [Solução do exercício PlayerManager4 de LP1 em MVC][pm4] (apenas consola e
  apenas usando matéria de LP1; neste caso o modelo é apenas uma lista de
  jogadores).

### Sugestões para o uso de sub-módulos em Git

Para fazerem _clone_ de um projeto com sub-módulos devem usar o seguinte
comando, exemplificado para o projeto exemplo [MVCExample][tutorial-code]:

```
git clone --recurse-submodules https://github.com/VideojogosLusofona/MVCExample.git
```

Caso se tenham esquecido de usar a opção `--recurse-submodules`, podem executar
o seguinte comando na raiz do projeto que obtém os conteúdos dos sub-módulos:

```
git submodule update --init --recursive
```

Os sub-módulos estão inicialmente no estado `HEAD detached`, isto é, não estão
em nenhum ramo. Para os sub-módulos ficarem no ramo pretendido, por exemplo o
ramo `common`, basta fazer `cd` até à pasta de cada sub-módulo e fazer
`git checkout common` (e depois `git pull` para obter as últimas alterações
ou `git add/commit/push` para criarem _commits_ específicos ao sub-módulo).

<!-- Em
alternativa podem obter as últimas alterações no ramo `common`

git submodule foreach --recursive git checkout common-->

O projeto [ColorShapeLinks][ia-simplexity] usa esta estratégia.

Como alternativa a um ramo separado, podem usar para o vosso projeto um segundo
repositório para conter o código comum.

### Gerar inteiros aleatórios a partir da distribuição de Poisson

Um gerador de números aleatórios obtidos a partir da distribuição de Poisson
recebe um valor real `λ` (média dos números aleatórios a devolver) e devolve
um número inteiro que corresponde ao número (aleatório) de eventos.

A linguagem C# apenas oferece a classe [Random], que produz números aleatórios
a partir da [distribuição uniforme]. O Wikipédia sugere [alguns
algoritmos](genPoisson) para obter valores a partir da distribuição de
Poisson tendo como base a distribuição uniforme. No entanto, apenas o segundo,
"**algorithm** _poisson random number (Junhao, based on Knuth)_",
funciona bem com valores elevados de `λ`, necessários para este projeto.
Na parte do algoritmo que diz "_Generate uniform random number u  in (0,1)_",
podem usar o método [`NextDouble()`] da classe [Random] para obter valores
aleatórios uniformes entre 0 e 1. Todas as variáveis internas deste algoritmo
devem ser `double`, exceto a variável `k`, que deve ser um `int`. O parâmetro
`STEP` deve ser uma constante com o valor 500.

Embora seja preferível os alunos implementarem esta função a partir
do pseudo-código disponível no Wikipédia, também se aceita o uso de código
encontrado na Internet com esta funcionalidade. Nesse caso, deve ser feita
referência à fonte.

### Embaralhar uma lista ou _array_

O algoritmo
[Fisher–Yates](https://en.wikipedia.org/wiki/Fisher%E2%80%93Yates_shuffle)
é um método de embaralhamento (_shuffling_) tipicamente utilizado para
embaralhar listas ou _arrays_.

Tal como no caso do gerador de números aleatórios de Poisson, é preferível
serem os alunos a implementar este algoritmo a partir do pseudo-código
disponível no Wikipédia. No entanto, também se aceita o uso código
encontrado na Internet, com a devida referência à fonte.

### _Parsing_ de números reais

De modo a converter uma _string_ num número real (neste caso, um `double`),
usa-se tipicamente uma das seguintes abordagens:

```cs
// s é uma string, x é um double
x = Convert.ToDouble(s);   // Abordagem 1
x = double.Parse(s);       // Abordagem 2
double.TryParse(s, out x); // Abordagem 3 (preferida)
```

A última forma é a preferida, pois permite-nos verificar se a conversão foi
inválida. No entanto pode ocorrer um problema caso o PC esteja configurado com
a língua portuguesa, na qual o separador decimal é uma vírgula e não um ponto.
Para evitar este problema, podemos indicar ao C# que pretendemos uma conversão
independente da língua configurada no computador, assumindo o ponto como
separador decimal:

```cs
// Requer using extra no início da classe
using System.Globalization;
//...
// s é uma string, x é um double
x = Convert.ToDouble(s, CultureInfo.InvariantCulture);               // Abordagem 1
x = double.Parse(s, NumberStyles.Any, CultureInfo.InvariantCulture); // Abordagem 2
double.TryParse(s, NumberStyles.Any, CultureInfo.InvariantCulture, out x); // Abordagem 3 (preferida)
```

Notar que a abordagem com `TryParse()` é normalmente usada com um `if`:

```cs
if (double.TryParse(...))
{
    // Conversão feita com sucesso
}
else
{
    // Conversão falhou
}
```

### Sugestão sobre como começar a abordar este projeto

Antes de começarem a preocupar-se com a arquitetura do projeto, MVC e submódulos
em Git, sugiro que implementem a simulação em consola e/ou Unity tendo como
única preocupação ver a simulação a funcionar (não se esqueçam de usar Git logo
de início, pois todos os _commits_ contam). Sugiro também que comparem a vossa
simulação com o _output_ produzido pelo [modelo em NetLogo][RPSNetLogo], de modo
a confirmarem que está tudo alinhado e que os resultados sao parecidos.

Quando tiverem confiança que a simulação está bem implementada, tentem separar
todo o código específico da simulação (que não contenha código de consola, e.g.
`Console.QualquerCoisa()`, nem código de Unity, e.g. `using UnityEngine`). Esse
código é o **modelo** (o **M** em MVC), e deve ser colocado num
projeto/pasta/_namespace_ próprio, e posteriormente num ramo (ou repositório)
próprio, tal como apresentado no [tutorial][tutorial-video]. Nesta fase podem
implementar a outra versão que ainda não implementaram, usando esse código
comum do modelo, tal como descrito no tutorial. Pode ser necessário
eventualmente fazer algum ajuste ao código comum. Devem garantir que tanto a
versão de consola como a versão Unity têm sempre o código comum igual e
atualizado.

## Objetivos e critério de avaliação

O projeto tem um peso de 10 valores na componente prática de LP2. A nota final
do projeto será atribuída segundo os seguintes objetivos:

* **O1** - Programa deve funcionar como especificado. Atenção aos detalhes,
  pois é fácil desviarem-se das especificações caso não leiam o enunciado com
  atenção.
* **O2** - Projeto e código bem organizados, nomeadamente:
  * O projeto deve estar devidamente organizado, fazendo uso de classes,
    `struct`s e/ou enumerações, consoante seja mais apropriado. Cada tipo
    (i.e., classe, `struct` ou enumeração) deve ser colocado num ficheiro com o
    mesmo nome. Por exemplo, uma classe chamada `Simulation` deve ser colocada
    no ficheiro `Simulation.cs`.
  * A escolha da coleções e *design patterns* deve ser adequada a cada
    situação. Serão privilegiadas soluções que tenham em consideração bons
    princípios de design de classes, como é o caso dos princípios [SOLID].
    Estes *patterns* e princípios devem ser balanceados com o princípio
    [KISS], crucial no desenvolvimento de qualquer aplicação.
  * O código deve ser o mais eficiente possível sem comprometer a arquitetura.
  * O código deve estar devidamente comentado e indentado.
  * Não deve existir código "morto", que não faz nada, como por exemplo
    variáveis, propriedades ou métodos nunca usados.
  * Projeto compila e executa sem erros e/ou *warnings*.
* **O3** - Projeto adequadamente documentado com [comentários de documentação
  XML][XML]. A documentação gerada em formato HTML em [Doxygen] ou [DocFX],
  deve estar incluída no `zip` do projeto, mas **não** integrada no repositório
  Git.
* **O4** - Repositório Git deve refletir boa utilização do mesmo, nomeadamente:
  * Devem existir *commits* de todos os elementos do grupo, _commits_ esses
    com mensagens que sigam as melhores práticas para o efeito (como indicado
    [aqui](https://chris.beams.io/posts/git-commit/),
    [aqui](https://gist.github.com/robertpainsi/b632364184e70900af4ab688decf6f53),
    [aqui](https://github.com/erlang/otp/wiki/writing-good-commit-messages) e
    [aqui](https://stackoverflow.com/questions/2290016/git-commit-messages-50-72-formatting)).
  * Ficheiros binários não necessários, como por exemplo todos os que são
    criados nas pastas `bin` e `obj`, bem como os ficheiros de configuração
    do Visual Studio (na pasta `.vs` ou `.vscode`), não devem estar no
    repositório. Ou seja, devem ser ignorados ao nível do ficheiro
    `.gitignore`.
  * *Assets* binários necessários, como é o caso da imagem do diagrama UML,
    devem ser integrados no repositório em modo Git LFS.
* **O5** - Relatório em formato [Markdown] (ficheiro `README.md`),
  organizado da seguinte forma:
  * Título do projeto.
  * Autoria:
    * Nome dos autores (primeiro e último) e respetivos números de aluno.
    * Informação de quem fez o quê no projeto. Esta informação é
      **obrigatória** e deve refletir os *commits* feitos no Git.
    * Indicação do repositório Git utilizado. Esta indicação é
      opcional, pois podem preferir manter o repositório privado após a
      entrega.
  * Arquitetura da solução:
    * Descrição da solução, com breve explicação de como o código foi
      organizado, bem como dos algoritmos não triviais que tenham sido
      implementados.
    * Um diagrama UML de classes simples (i.e., sem indicação dos
      membros da classe) descrevendo a estrutura de classes.
  * Observações e resultados:
    * Indicar o que acontece ao colocar `swap-rate-exp` a 1.0 deixando as
      restantes `rate-exp` a zero.
    * Indicar o que acontece ao colocar `swap-rate-exp` a -1.0 deixando as
      restantes `rate-exp` a zero.
    * É possível encontrar algum conjunto de parâmetros que resulte na extinção
      de uma das espécies? Quando uma espécie se extingue, o que acontece às
      outras duas?
  * Referências, incluindo trocas de ideias com colegas, código aberto
    reutilizado (e.g., do StackOverflow) e bibliotecas de terceiros
    utilizadas. Devem ser o mais detalhados possível.
  * **Nota:** o relatório deve ser simples e breve, com informação mínima e
    suficiente para que seja possível ter uma boa ideia do que foi feito.
    Atenção aos erros ortográficos e à correta formatação [Markdown], pois
    ambos serão tidos em conta na nota final.

O projeto tem um peso de 10 valores na nota final da disciplina e será avaliado
de forma qualitativa. Isto significa que todos os objetivos têm de ser
parcialmente ou totalmente cumpridos. A cada objetivo, O1 a O5, será atribuída
uma nota entre 0 e 1. A nota do projeto será dada pela seguinte fórmula:

*N = 10 x O1 x O2 x O3 x O4 x O5 x D x A*

Em que *D* corresponde à nota da discussão e percentagem equitativa de
realização do projeto, também entre 0 e 1. Isto significa que se os alunos
ignorarem completamente um dos objetivos, não tenham feito nada no projeto ou
não comparecerem na discussão, a nota final será zero.

O termo *A* representa uma penalização por atrasos na entrega.

## Entrega

O projeto deve ser submetido no Moodle até às **23h00 de 14 de julho de 2021**.
O projeto entregue deve ter os seguintes conteúdos:

* Todos os ficheiros do projeto incluídos no repositório git.
* Pasta escondida `.git` com o repositório Git local do projeto.
* Documentação HTML ou CHM gerada com [Doxygen], [DocFX] ou
  ferramenta similar.
* Ficheiro `README.md` contendo o relatório do projeto em formato [Markdown].
* Ficheiros de imagens, contendo o diagrama UML de classes e outras figuras
  que considerem úteis. Estes ficheiros, bem como ficheiros binários da versão
  Unity, devem ser incluídos no repositório em modo Git LFS.

**Atenção:** A submissão tem de ter menos de 20 Megabytes e ser efetuada mesmo
através do Moodle. Não são aceites _links_ para Google Drive, etc. Desta forma
tenham muito cuidado com o `.gitignore` e o `.gitattributes` no início do
projeto, e tenham em atenção que a versão Unity e de consola podem precisar de
ficheiros de configuração Git diferentes (ver [código do
tutorial][tutorial-code]).

## Honestidade académica

Nesta disciplina, espera-se que cada aluno siga os mais altos padrões de
honestidade académica. Isto significa que cada ideia que não seja do
aluno deve ser claramente indicada, com devida referência ao respectivo
autor. O não cumprimento desta regra constitui plágio.

O plágio inclui a utilização de ideias, código ou conjuntos de soluções
de outros alunos ou indivíduos, ou quaisquer outras fontes para além
dos textos de apoio à disciplina, sem dar o respectivo crédito a essas
fontes. Os alunos são encorajados a discutir os problemas com outros
alunos e devem mencionar essa discussão quando submetem os projetos.
Essa menção **não** influenciará a nota. Os alunos não deverão, no
entanto, copiar códigos, documentação e relatórios de outros alunos, ou dar os
seus próprios códigos, documentação e relatórios a outros em qualquer
circunstância. De facto, não devem sequer deixar códigos, documentação e
relatórios em computadores de uso partilhado.

Nesta disciplina, a desonestidade académica é considerada fraude, com
todas as consequências legais que daí advêm. Qualquer fraude terá como
consequência imediata a anulação dos projetos de todos os alunos envolvidos
(incluindo os que possibilitaram a ocorrência). Qualquer suspeita de
desonestidade académica será relatada aos órgãos superiores da escola
para possível instauração de um processo disciplinar. Este poderá
resultar em reprovação à disciplina, reprovação de ano ou mesmo suspensão
temporária ou definitiva da ULHT.

*Texto adaptado da disciplina de [Algoritmos e
Estruturas de Dados][aed] do [Instituto Superior Técnico][ist]*

## Referências

* **Pedra, Papel e Tesoura**. Retrieved from <https://pt.wikipedia.org/wiki/Pedra,_papel_e_tesoura>.
* Head, B., Grider, R. and Wilensky, U. (2017). **NetLogo Rock Paper Scissors
  model**. http://ccl.northwestern.edu/netlogo/models/RockPaperScissors. Center
  for Connected Learning and Computer-Based Modeling, Northwestern University,
  Evanston, IL.
* Whitaker, R. B. (2016). **The C# Player's Guide** (3rd Edition).
  Starbound Software.
* Albahari, J. (2017). **C# 7.0 in a Nutshell**. O’Reilly Media.
* Dorsey, T. (2017). **Doing Visual Studio and .NET Code Documentation
  Right**. Visual Studio Magazine. Retrieved from
  <https://visualstudiomagazine.com/articles/2017/02/21/vs-dotnet-code-documentation-tools-roundup.aspx>.

## Licenças

Este enunciado é disponibilizado através da licença [CC BY-NC-SA 4.0].

## Metadados

* Autor: [Nuno Fachada]
* Curso:  [Licenciatura em Videojogos][lamv]
* Instituição: [Universidade Lusófona de Humanidades e Tecnologias][ULHT]

[Pedra, Papel e Tesoura]:https://pt.wikipedia.org/wiki/Pedra,_papel_e_tesoura
[MVC]:https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93controller
[Von Neumann]:https://en.wikipedia.org/wiki/Von_Neumann_neighborhood
[RPSNetLogo]:http://ccl.northwestern.edu/netlogo/models/RockPaperScissors
[NetLogo]:http://ccl.northwestern.edu/netlogo
[distribuição de Poisson]:https://en.wikipedia.org/wiki/Poisson_distribution
[distribuição uniforme]:https://en.wikipedia.org/wiki/Uniform_distribution_(continuous)
[Random]:https://docs.microsoft.com/dotnet/api/system.random
[`NextDouble()`]:https://docs.microsoft.com/dotnet/api/system.random.nextdouble
[genPoisson]:https://en.wikipedia.org/wiki/Poisson_distribution#Generating_Poisson-distributed_random_variables
[submodules]:https://git-scm.com/book/en/v2/Git-Tools-Submodules
[ia-simplexity]:https://github.com/VideojogosLusofona/color-shape-links-ai-competition
[tutorial-video]:https://www.youtube.com/watch?v=_z_iRUjmvzE
[tutorial-code]:https://github.com/VideojogosLusofona/MVCExample.git
[ttt-mvc]:https://github.com/VideojogosLusofona/lp1_2020_galo_solucao
[pm4]:https://github.com/VideojogosLusofona/lp1_2020_aulas/tree/main/Aula12/PlayerManager4
[rps-nl]:http://ccl.northwestern.edu/netlogo/models/RockPaperScissors
[Markdown]:https://guides.github.com/features/mastering-markdown/
[Doxygen]:https://www.stack.nl/~dimitri/doxygen/
[DocFX]:https://dotnet.github.io/docfx/
[KISS]:https://en.wikipedia.org/wiki/KISS_principle
[XML]:https://docs.microsoft.com/dotnet/csharp/codedoc
[SOLID]:https://en.wikipedia.org/wiki/SOLID
[CC BY-NC-SA 4.0]:https://creativecommons.org/licenses/by-nc-sa/4.0/
[lamv]:https://www.ulusofona.pt/licenciatura/videojogos
[Nuno Fachada]:https://github.com/fakenmc
[ULHT]:https://www.ulusofona.pt/
[aed]:https://fenix.tecnico.ulisboa.pt/disciplinas/AED-2/2009-2010/2-semestre/honestidade-academica
[ist]:https://tecnico.ulisboa.pt/pt/
