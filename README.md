# ReportsGenerator

Projeto Elixir que extrai dados de um arquivo `.csv` que corresponde a um relatório de usuários de um aplicativo de delivery, contendo os dados da quantidade e de qual alimento cada usuário consumiu.

Os dados do arquivo será extraído, processado e avaliado conforme seu desempenho durante a execução que retornará:

- qual o usuário que mais consumiu, e
- qual o alimento mais consumido.

Teremos duas formas de execução:

- um arquivo contendo 300 mil linhas de dados
- 3 arquivos contendo 100 mil linhas de dados cada

## Inicialização

Iniciando o projeto

```bash
mix new reports_generator
```

## Configuração de Credo

Credo é um analisador sintático do código, que nos ajuda a manter a boas de escrita do nosso código. Para instalar, seguir as instruções desse [repo](https://github.com/rrrene/credo).

Adicionar a dependência no arquivo `mix.exs` na parte de `deps`:

```elixir
defp deps do
  [
    {:credo, "~> 1.5", only: [:dev, :test], runtime: false}
  ]
end
```

Observe que o primeiro argumento da tupla é o nome da lib `:credo` e o segundo `"~> 1.5"` é a versão, e a terceira parte não costuma ter, mas no caso do Credo tem porque é usado somente para ambiente de desenvolvimento `:dev` e de teste `:test` para checarmos nosso código. Além disso, `runtime: false` não encapsula essa lib no binário que vai ser gerado para ser executado no nosso servidor.

No VSCode, as dependências já são baixadas automaticamente. Mas é possível baixá-las pelo terminal com o comando:

```bash
mix deps.get
```

Uma vez instalado o Credo, vamos dar o comando que vai gerar o arquivo `.credo.exs` para configurá-lo (o que vai checar ou não):

```bash
mix credo gen.config
```

Por exemplo, como não vamos fazer a documentação dos módulos, vamos alterar na parte de `Readability Checks` essa parte:

```elixir
{Credo.Check.Readability.ModuleDoc, false},
```

Para rodar o Credo:

```bash
mix credo
```

E para pegar até o que está com `[priority: :low]` como nesse caso:

```elixir
{Credo.Check.Readability.MaxLineLength, [priority: :low, max_length: 120]},
```

usamos o comando do Credo mais severo:

```bash
mix credo --strict
```

E caso não queira ficar rodando na linha de comando sempre, podemos baixar a extensão do VSCode [ElixirLinter](https://marketplace.visualstudio.com/items?itemName=iampeterbanjo.elixirlinter) e deixar a opção `Use strict mode with Credo` habilitada.

## Lendo arquivo

Vamos criar a função `build` que vai receber o nome do arquivo .csv como argumento dentro do nosso módulo `ReportsGenerator`

```elixir
defmodule ReportsGenerator do
  def build(filename) do
    file = File.read("reports/#{filename}")
  end
end
```

E rodar no `iex`

```bash
iex -S mix
```

Ao ler um arquivo existente:

```elixir
ReportsGenerator.build("report_test.csv")
#..> {:ok, "1,pizza,48\r\n2,açaí,45\r\n3,hambúrguer,31\r\n4,esfirra,42\r\n5,hambúrguer,49\r\n6,esfirra,18\r\n7,pizza,27\r\n8,esfirra,25\r\n9,churrasco,24\r\n10,churrasco,36"}
```

E para um arquivo inexistente:

```elixir
ReportsGenerator.build("report_testq.csv")
#..> {:error, :enoent}
```

Para tratar ambas situações, podemos usar o [case](https://elixir-lang.org/getting-started/case-cond-and-if.html#case), no qual o `_ -> "caso qualquer"` seria o `default`, mas não será necessário para esse caso especificamente:

```elixir
defmodule ReportsGenerator do
  def build(filename) do
    case File.read("reports/#{filename}") do
      {:ok, result} -> result
      {:error, reason} -> reason
      _ -> "caso qualquer"
    end
  end
end
```

## Pipe operator

[Pipe operator](https://elixir-lang.org/getting-started/enumerables-and-streams.html#the-pipe-operator) é uma feature do Elixir que ilustra bem como funciona as operações como um chão de fábrica.

Por exemplo, ao tratar a string `" aAaaAaaa \n"`, podemos fazer em várias etapas no `iex`:

```elixir
string = " aAaaAaaa \n"
#..> " aAaaAaaa \n"
string = String.trim(string)
#..> "aAaaAaaa"
string = String.downcase(string)
#..> "aaaaaaaa"
```

Ou poderíamos fazer em uma linha só, porém não fica tão legível a concatenação das funções:

```elixir
string = String.trim(String.downcase(" aAaaAaaa \n"))
#..> "aaaaaaaa"
```

O pipe operator pega o resultado da operação anterior e passa como primeiro argumento da função seguinte. Então, fica mais elegante e legível a concatenação das funções:

```elixir
" aaaAaaaaAAAAaaaa \n" |> String.trim() |> String.downcase()
#..> "aaaaaaaaaaaaaaaa"
```

Refatorando o código:

```elixir
defmodule ReportsGenerator do
  def build(filename) do
    "reports/#{filename}"
    |> File.read()
    |> handle_file()
  end

  def handle_file({:ok, file_content}), do: file_content
  def handle_file({:error, _reason}), do: "Error while opening file"
end
```

O primeiro argumento fica implícito na chamada da função seguinte. Caso necessite passar outros argumentos para funções de maior aridade, então precisamos passá-los explicitamente:

```elixir
defmodule ReportsGenerator do
  def build(filename) do
    "reports/#{filename}"
    |> File.read()
    |> handle_file("lalala")
  end

  def handle_file({:ok, file_content}, msg), do: file_content <> msg
  def handle_file({:error, _reason}, msg), do: "Error while opening file" <> msg
end
```

Obs: a concatenação de string é por `<>`.

No `iex`:

```elixir
ReportsGenerator.build("report_test.csv")
#..> "1,pizza,48\r\n2,açaí,45\r\n3,hambúrguer,31\r\n4,esfirra,42\r\n5,hambúrguer,49\r\n6,esfirra,18\r\n7,pizza,27\r\n8,esfirra,25\r\n9,churrasco,24\r\n10,churrasco,36lalala"

ReportsGenerator.build("report_test.csvs")
#..> "Error while opening filelalala"
```

## Leitura em stream

Ao invés usar [File.read](https://hexdocs.pm/elixir/File.html#read/1), iremos usar o [File.stream!](https://hexdocs.pm/elixir/File.html#stream!/3).

```elixir
  def build(filename) do
    "reports/#{filename}"
    |> File.stream!()
    |> IO.inspect()
  end
```

Somente para vermos o que o `File.stream!` está retornando no `iex`:

```elixir
ReportsGenerator.build("report_test.csv")
#..> %File.Stream{
#..>   line_or_bytes: :line,
#..>   modes: [:raw, :read_ahead, :binary],
#..>   path: "reports/report_test.csv",
#..>   raw: true
#..> }
#..> %File.Stream{
#..>   line_or_bytes: :line,
#..>   modes: [:raw, :read_ahead, :binary],
#..>   path: "reports/report_test.csv",
#..>   raw: true
#..> }
```

Vemos que retorna uma [struct](https://elixir-lang.org/getting-started/structs.html#structs-are-bare-maps-underneath), que nada mais é que um `Map` com um nome.

O `File.Stream` não possui o conteúdo do arquivo, nele temos os metadados como:

- `line_or_bytes` que é como ele vai ler o arquivo
- `path` que é o caminho do arquivo

Ele não tem o conteúdo em si do arquivo, pois é uma função com característica `laziness`, que só traz o conteúdo quando quiser utilizar.

O `File.stream!` combinado com o `Enum.each` vai dar o efeito de: ler uma linha e executar uma operação. De forma que vai lendo o arquivo e executando linha a linha, sem a necessidade de carregar o conteúdo inteiro do arquivo na memória. E tem o `!` porque diferentemente do `File.read` que retorna tupla de `:ok` ou `:error` para ler o arquivo e retornar o conteúdo, no `File.stream!` só vai ler o arquivo no futuro.

```elixir
defmodule ReportsGenerator do
  def build(filename) do
    "reports/#{filename}"
    |> File.stream!()
    |> Enum.each(fn elem -> IO.inspect(elem) end)
  end
end
```

Retornando assim:

```elixir
ReportsGenerator.build("report_test.csv")
#..> "1,pizza,48\n"
#..> "2,açaí,45\n"
#..> "3,hambúrguer,31\n"
#..> "4,esfirra,42\n"
#..> "5,hambúrguer,49\n"
#..> "6,esfirra,18\n"
#..> "7,pizza,27\n"
#..> "8,esfirra,25\n"
#..> "9,churrasco,24\n"
#..> "10,churrasco,36"
#..> :ok
```

Ou seja, para cada linha do arquivo, foi executado um `IO.inspect()`. No final da operação, o `Enum.each()` retorna um `:ok` para dizer que a operação ocorreu com sucesso.

Para melhorar, vamos separar da linha `"1,pizza,48\n"` quem é `id`, `produto`, `valor` e remover `\n`.

```elixir
defmodule ReportsGenerator do
  def build(filename) do
    "reports/#{filename}"
    |> File.stream!()
    |> Enum.map(fn line -> parse_line(line) end)
  end

  defp parse_line(line) do
    line
    |> String.trim()
    |> String.split(",")
  end
end
```

Ao rodar no `iex`, temos:

```elixir
ReportsGenerator.build("report_test.csv")
#..> [
#..>   ["1", "pizza", "48"],
#..>   ["2", "açaí", "45"],
#..>   ["3", "hambúrguer", "31"],
#..>   ["4", "esfirra", "42"],
#..>   ["5", "hambúrguer", "49"],
#..>   ["6", "esfirra", "18"],
#..>   ["7", "pizza", "27"],
#..>   ["8", "esfirra", "25"],
#..>   ["9", "churrasco", "24"],
#..>   ["10", "churrasco", "36"]
#..> ]
```

Então vemos que o primeiro elemento da lista é o `id`, o segundo é o `produto` e o terceiro é o `preço`. Mas vamos alterar o preço de string para valor

```elixir
  defp parse_line(line) do
    line
    |> String.trim()
    |> String.split(",")
    |> List.update_at(2, &String.to_integer/1)
  end
```

Retornando o terceiro elemento como `integer`:

```elixir
ReportsGenerator.build("report_test.csv")
#..> [
#..>   ["1", "pizza", 48],
#..>   ["2", "açaí", 45],
#..>   ["3", "hambúrguer", 31],
#..>   ["4", "esfirra", 42],
#..>   ["5", "hambúrguer", 49],
#..>   ["6", "esfirra", 18],
#..>   ["7", "pizza", 27],
#..>   ["8", "esfirra", 25],
#..>   ["9", "churrasco", 24],
#..>   ["10", "churrasco", 36]
#..> ]
```

Ao criar uma função anônima, podemos usar a sintaxe:

```elixir
    |> Enum.map(fn line -> parse_line(line) end)
```

Ou de maneira implícita com o `&`:

```elixir
    |> Enum.map(&parse_line(&1))
```

No caso de uma função de módulo externo, a função anônima explícita seria:

```elixir
    |> List.update_at(2, fn elem -> String.to_integer(elem) end)
```

E a função implícita:

```elixir
    |> List.update_at(2, &String.to_integer/1)
```

## Parse do arquivo

Ao invés de retornar uma lista de listas, seria melhor retornar um Map `%{id: valor}`, onde o `id` seria o `id` do usuário e o `valor` seria a soma dos valores gastos pelo usuário.

[Enum.reduce](https://hexdocs.pm/elixir/Enum.html#reduce/3)

```elixir
Enum.reduce([1,2,3,4], 0, fn elem, acc -> acc + elem end)
#..> 10
```

E vamos observar o que acontece em cada parte:

```elixir
Enum.reduce([1,2,3,4], %{}, fn elem, acc -> IO.inspect(elem); IO.inspect(acc); Map.put(acc, elem, elem) end)
#..> 1
#..> %{}
#..> 2
#..> %{1 => 1}
#..> 3
#..> %{1 => 1, 2 => 2}
#..> 4
#..> %{1 => 1, 2 => 2, 3 => 3}
#..> %{1 => 1, 2 => 2, 3 => 3, 4 => 4}
```

Vamos refatorar o método `build` para retornar um Map contendo o id do usuário como chave e o price como valor.

```elixir
  def build(filename) do
    "reports/#{filename}"
    |> File.stream!()
    |> Enum.reduce(%{}, fn line, report ->
      [id, _food_name, price] = parse_line(line)
      Map.put(report, id, price)
    end)
  end
```

```elixir
ReportsGenerator.build("report_test.csv")
#..> %{
#..>   "1" => 48,
#..>   "10" => 36,
#..>   "2" => 45,
#..>   "3" => 31,
#..>   "4" => 42,
#..>   "5" => 49,
#..>   "6" => 18,
#..>   "7" => 27,
#..>   "8" => 25,
#..>   "9" => 24
#..> }
```

Mas nosso método ainda está retornando somente o valor da último registro do usuário no arquivo csv por causa do `Map.put(report, id, price)`.

Para acessar o valor atual, acessar por `report[id]` e somar com o `price`, alterando então para `Map.put(report, id, report[id] + price)`. Mas isso gera uma exceção na primeira vez que roda.

```elixir
ReportsGenerator.build("report_test.csv")
#..> ** (ArithmeticError) bad argument in arithmetic expression: nil + 48
#..>     :erlang.+(nil, 48)
#..>     (reports_generator 0.1.0) lib/reports_generator.ex:7: anonymous fn/2 in ReportsGenerator.build/1
#..>     (elixir 1.11.4) lib/enum.ex:3473: anonymous fn/3 in Enum.reduce/3
#..>     (elixir 1.11.4) lib/stream.ex:1449: Stream.do_element_resource/6
#..>     (elixir 1.11.4) lib/enum.ex:3473: Enum.reduce/3
```

Para contornar a soma de `nil` com um valor qualquer, podemos criar um Map acumulador inicial com essa estrutura:

```elixir
%{
  "1" => 0,
  "2" => 0,
  "3" => 0,
  "4" => 0,
  "5" => 0,
  "..." => 0,
  "30" => 0,
}
```

Pois temos 30 usuários nesses arquivos.

[Enum.into](https://hexdocs.pm/elixir/Enum.html#into/3) é uma função onde podemos pegar uma coleção e converter em outra.

Gerar um `range` pegando o valor inicial e até onde queremos ir

```elixir
Enum.map(1..30, fn elem -> elem end)
#..> [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30]
```

Usando syntax sugar:

```elixir
Enum.into(1..30, %{}, &{&1, &1})
#..> %{
#..>   1 => 1,
#..>   2 => 2,
#..>   3 => 3,
#..>   ...
#..>   29 => 29,
#..>   30 => 30
#..> }
```

Ou, de maneira mais explícita:

```elixir
Enum.into(1..30, %{}, fn x -> {x, x} end)

#..>   1 => 1,
#..>   2 => 2,
#..>   3 => 3,
#..>   ...
#..>   29 => 29,
#..>   30 => 30
#..> }
```

Mas queremos a chave como string e o valor zerado:

```elixir
Enum.into(1..30, %{}, fn x -> {Integer.to_string(x), 0} end)
#..> %{
#..>   "1" => 0,
#..>   "10" => 0,
#..>   "11" => 0,
#..>   "12" => 0,
#..>   ...
#..> }
```

Ou, como sugar syntax:

```elixir
Enum.into(1..30, %{}, &{Integer.to_string(&1), 0})
```

Vamos criar uma função `report_acc` que retorna esse `Enum.into` acima e refatorar o método `build` do nosso módulo:

```elixir
defmodule ReportsGenerator do
  def build(filename) do
    "reports/#{filename}"
    |> File.stream!()
    |> Enum.reduce(report_acc(), fn line, report ->
      [id, _food_name, price] = parse_line(line)
      Map.put(report, id, report[id] + price)
    end)
  end

  defp parse_line(line) do
    line
    |> String.trim()
    |> String.split(",")
    |> List.update_at(2, &String.to_integer/1)
  end

  defp report_acc, do: Enum.into(1..30, %{}, &{Integer.to_string(&1), 0})
end
```

E ao recompilar e executar:

```elixir
ReportsGenerator.build("report_complete.csv")
#..> %{
#..>   "1" => 278849,
#..>   "10" => 268317,
#..>   "11" => 268877,
#..>   "12" => 276306,
#..>   "13" => 282953,
#..>   ...
#..> }
```

## Extraindo o parser para um novo módulo

A responsabilidade de parse das linhas pode ficar em outro módulo. Criamos um novo arquivo dentro de `lib` chamado `parser.ex`. E para evitar de repetir o uso de `Enum` nos dois arquivos, vamos usar o módulo `Stream`. Assim como o `File.stream!()` é _lazy_ na execução (só executa quando precisamos), o módulo [Stream](https://hexdocs.pm/elixir/Stream.html) também é. A gente monta um conjunto de operações que vão ser executadas, mas elas serão de fato executadas quando a gente precisar dos valores.

```elixir
defmodule ReportsGenerator.Parser do
  def parse_file(filename) do
    "reports/#{filename}"
    |> File.stream!()
    |> Stream.map(fn line -> parse_line(line) end)
    |> Enum.each(fn elem -> IO.inspect(elem) end) # remover essa linha depois
  end

  defp parse_line(line) do
    line
    |> String.trim()
    |> String.split(",")
    |> List.update_at(2, &String.to_integer/1)
  end
end
```

Ao rodar no `iex`:

```elixir
ReportsGenerator.Parser.parse_file("report_test.csv")
#..> ["1", "pizza", 48]
#..> ["2", "açaí", 45]
#..> ["3", "hambúrguer", 31]
#..> ["4", "esfirra", 42]
#..> ["5", "hambúrguer", 49]
#..> ["6", "esfirra", 18]
#..> ["7", "pizza", 27]
#..> ["8", "esfirra", 25]
#..> ["9", "churrasco", 24]
#..> ["10", "churrasco", 36]
#..> :ok
```

Refatorando agora no módulo `ReportsGenerator`:

```elixir
defmodule ReportsGenerator do
  def build(filename) do
    filename
    |> ReportsGenerator.Parser.parse_file()
    |> Enum.reduce(report_acc(), fn [id, _food_name, price], report ->
      Map.put(report, id, report[id] + price)
    end)
  end

  defp report_acc, do: Enum.into(1..30, %{}, &{Integer.to_string(&1), 0})
end
```

A primeira melhoria que podemos fazer é quando usamos um módulo externo que tem um nome muito grande, podemos fazer um alias. E outra melhoria é isolar a soma em uma função.

```elixir
defmodule ReportsGenerator do
  alias ReportsGenerator.Parser

  def build(filename) do
    filename
    |> Parser.parse_file()
    |> Enum.reduce(report_acc(), fn line, report -> sum_values(line, report) end)
  end

  defp sum_values([id, _food_name, price], report), do: Map.put(report, id, report[id] + price)

  defp report_acc, do: Enum.into(1..30, %{}, &{Integer.to_string(&1), 0})
end
```

## Retornando o maior valor

Estamos retornando um relatório de quanto usuário consumiu. Agora faremos uma função que recebe esse `report` e retorne qual usuário que mais consumiu.

O módulo `Enum` tem a função [Enum.max_by()](https://hexdocs.pm/elixir/Enum.html#max_by/4) que vai retornar o ítem de maior valor de uma coleção dada.

Como o `report` está vindo como um `Map` de chave e valor, e quero que retorne pelo maior `value`.

```elixir
  def fetch_higher_cost(report), do: Enum.max_by(report, fn {_key, value} -> value end)
```

Pelo `iex`, podemos usa o `pipe operator`:

```elixir
"report_complete.csv" |> ReportsGenerator.build() |> ReportsGenerator.fetch_higher_cost()
#..> {"13", 282953}
```

O usuário número `13` foi o que mais consumiu.

## Calculando a frequência de cada tipo de pedido

Também vamos calcular qual comida foi mais frequente. Vamos alterar nosso `report_acc` para percorrer a lista e somar de acordo com a comida. Da mesma forma que criamos um Map zerado para todos os valores, faremos para as comidas.

Ao invés de fazer um `Enum` de `1..30`, vamos criar uma variável de module que é um valor constante que vai ser usado dentro desse module.

```elixir
  def report_acc do
    foods = Enum.into(@available_foods, %{}, &{&1, 0})
    users = Enum.into(1..30, %{}, &{Integer.to_string(&1), 0})

    %{"users" => users, "foods" => foods}
  end
```

Deixamos a função pública somente para ver no `iex` como ficaria:

```elixir
ReportsGenerator.report_acc()
#..> %{
#..>   "foods" => %{
#..>     "açaí" => 0,
#..>     "churrasco" => 0,
#..>     "esfirra" => 0,
#..>     "hambúrguer" => 0,
#..>     "pastel" => 0,
#..>     "pizza" => 0,
#..>     "prato_feito" => 0,
#..>     "sushi" => 0
#..>   },
#..>   "users" => %{
#..>     "1" => 0,
#..>     "10" => 0,
#..>     "11" => 0,
#..>     "12" => 0,
#..>     "13" => 0,
#..>     "14" => 0,
#..>     "15" => 0,
#..>     "16" => 0,
#..>     "17" => 0,
#..>     "18" => 0,
#..>     "19" => 0,
#..>     "2" => 0,
#..>     "20" => 0,
#..>     "21" => 0,
#..>     "22" => 0,
#..>     "23" => 0,
#..>     "24" => 0,
#..>     "25" => 0,
#..>     "26" => 0,
#..>     "27" => 0,
#..>     "28" => 0,
#..>     "29" => 0,
#..>     "3" => 0,
#..>     "30" => 0,
#..>     "4" => 0,
#..>     "5" => 0,
#..>     "6" => 0,
#..>     "7" => 0,
#..>     "8" => 0,
#..>     "9" => 0
#..>   }
#..> }
```

Agora com o acumulador contendo os dois valores, temos que alterar o `sum_values`. Vamos separar cada um dos Maps pelo pattern matching. Vamos usar cada um dos Maps isolados para atualizar o `report`. Como estamos numa linguagem imutável, precisamos reatribuir o valor em `users` e em `foods`. Por fim, devolvemos o Map com os dois valores.

```elixir
  defp sum_values([id, food_name, price], %{"foods" => foods, "users" => users} = report) do
    users = Map.put(users, id, users[id] + price)
    foods = Map.put(foods, food_name, foods[food_name] + 1)

    report
    |> Map.put("users", users)
    |> Map.put("foods", foods)
  end
```

No `iex`

```elixir
"report_complete.csv" |> ReportsGenerator.build()
#..> %{
#..>   "foods" => %{
#..>     "açaí" => 37742,
#..>     "churrasco" => 37650,
#..>     "esfirra" => 37462,
#..>     "hambúrguer" => 37577,
#..>     "pastel" => 37392,
#..>     "pizza" => 37365,
#..>     "prato_feito" => 37519,
#..>     "sushi" => 37293
#..>   },
#..>   "users" => %{
#..>     "1" => 278849,
#..>     "10" => 268317,
#..>     "11" => 268877,
#..>     "12" => 276306,
#..>     "13" => 282953,
#..>     "14" => 277084,
#..>     "15" => 280105,
#..>     "16" => 271831,
#..>     "17" => 272883,
#..>     "18" => 271421,
#..>     "19" => 277720,
#..>     "2" => 271031,
#..>     "20" => 273446,
#..>     "21" => 275026,
#..>     "22" => 278025,
#..>     "23" => 276523,
#..>     "24" => 274481,
#..>     "25" => 274512,
#..>     "26" => 274199,
#..>     "27" => 278001,
#..>     "28" => 274256,
#..>     "29" => 273030,
#..>     "3" => 272250,
#..>     "30" => 275978,
#..>     "4" => 277054,
#..>     "5" => 270926,
#..>     "6" => 272053,
#..>     "7" => 273112,
#..>     "8" => 275161,
#..>     "9" => 274003
#..>   }
#..> }
```

Outra forma de atualizar o `report` com o pipe operator

```elixir
    %{report | "users" => users, "foods" => foods}
```

## Refatorando o retorno do maior preço

Nossa função `fetch_higher_cost` não funciona mais, pois está só comparando entre `users` e `food` qual é o maior. No caso, o retorno é de `users` e isso não faz sentido.

```elixir
  def fetch_higher_cost(report, option), do: Enum.max_by(report[option], fn {_key, value} -> value end)
```

Verificando no `iex`

```elixir
"report_complete.csv" |> ReportsGenerator.build() |> ReportsGenerator.fetch_higher_cost("users")
#..> {"13", 282953}
"report_complete.csv" |> ReportsGenerator.build() |> ReportsGenerator.fetch_higher_cost("foods")
#..> {"açaí", 37742}
```

Vamos criar uma nova variável de módulo para evitar que o usuário passe uma opção inválida, pois nosso Map não tem uma chave chamada `banana`.

```elixir
"report_complete.csv" |> ReportsGenerator.build() |> ReportsGenerator.fetch_higher_cost("banana")
#..> ** (Protocol.UndefinedError) protocol Enumerable not implemented for nil of type Atom. This protocol is implemented for the following type(s): HashSet, Range, Map, Function, List, Stream, Date.Range, HashDict, GenEvent.Stream, MapSet, File.Stream, IO.Stream
#..>     (elixir 1.11.4) lib/enum.ex:1: Enumerable.impl_for!/1
#..>     (elixir 1.11.4) lib/enum.ex:141: Enumerable.reduce/3
#..>     (elixir 1.11.4) lib/enum.ex:3473: Enum.reduce/3
#..>     (elixir 1.11.4) lib/enum.ex:3310: Enum.aggregate_by/4
```

[Guards](https://hexdocs.pm/elixir/guards.html) são funcionalidades que podemos usar nas nossas definições para estender o poder do pattern matching.

```elixir
  @options ["foods", "users"]

  def fetch_higher_cost(report, option) when option in @options do
    Enum.max_by(report[option], fn {_key, value} -> value end)
  end
```

No `iex`

```elixir
"report_complete.csv" |> ReportsGenerator.build() |> ReportsGenerator.fetch_higher_cost("banana")
#..> ** (FunctionClauseError) no function clause matching in ReportsGenerator.fetch_higher_cost/2
#..>     ...
#..>     The following arguments were given to ReportsGenerator.fetch_higher_cost/2:
#..>     Attempted function clauses (showing 1 out of 1):
#..>         def fetch_higher_cost(report, option) when option === "foods" or option === "users"
#..>     (reports_generator 0.1.0) lib/reports_generator.ex:23: ReportsGenerator.fetch_higher_cost/2
```

Via pattern matching, criamos uma função para quando a cláusula de cima não for satisfeita e retornamos um erro

```elixir
  def fetch_higher_cost(report, option), do: {:error, "Invalid option!"}
```

No `iex`

```elixir
"report_complete.csv" |> ReportsGenerator.build() |> ReportsGenerator.fetch_higher_cost("banana")
#..> {:error, "Invalid option!"}
```

No caso do sucesso, vamos retornar como tupla também

```elixir
  def fetch_higher_cost(report, option) when option in @options do
    {:ok, Enum.max_by(report[option], fn {_key, value} -> value end)}
  end
```

No `iex`

```elixir
"report_complete.csv" |> ReportsGenerator.build() |> ReportsGenerator.fetch_higher_cost("foods")
#..> {:ok, {"açaí", 37742}}
```

## Testando nosso Parser

Criamos o arquivo `test/parser_test.exs`. Acrescentamos o `ExUnit.Case` e fazemos o `alias` do nosso módulo `ReportsGenerator.Parser`.

Em `describe` colocamos o nome da função e sua aridade.

O cenário que queremos testar é para fazer o `parse` do arquivo em `test`. Utilizar sempre o arquivo de teste `report_test.csv`. Primeiro faz o teste falhar, como `banana test driven development` 🤣

```elixir
defmodule ReportsGenerator.ParserTest do
  use ExUnit.Case

  alias ReportsGenerator.Parser

  describe "parse_file/1" do
    test "parses the file" do
      file_name = "report_test.csv"

      response = Parser.parse_file(file_name)

      expected_response = "banana"

      assert response == expected_response
    end
  end
end
```

Para rodar somente o arquivo desejado

```bash
mix test test/parser_test.exs
  # 1) test parse_file/1 parses the file (ReportsGenerator.ParserTest)
    #  test/parser_test.exs:7
    #  Assertion with == failed
    #  code:  assert response == expected_response
    #  left:  #Stream<[enum: %File.Stream{line_or_bytes: :line, modes: [:raw, :read_ahead, :binary], path: "reports/report_test.csv", raw: true}, funs: [#Function<47.104660160/1 in Stream.map/2>]]>
    #  right: "banana"
    #  stacktrace:
      #  test/parser_test.exs:29: (test)
# Finished in 0.03 seconds
# 1 test, 1 failure
```

No nosso `parse` devolvemos um `Stream.map`, ou seja, não tem conteúdo nenhum pois o `Stream` é `lazy`. Só vai executar quando precisarmos do conteúdo. Para não temos que fazer um `pattern matching` com a structure esquisita dentro do Stream, vamos começar com o `file_name`, passar para o `parse_file` e o resultado vamos passar para o `Enum.map` apenas imprimindo o valor dessa linha.

Ao invés de usar o style com pipe na horizontal, seguir o [style guide](https://github.com/christopheradams/elixir_style_guide) e deixar o style com pipe na vertical.

```elixir
response = file_name |> Parser.parse_file() |> Enum.map(& &1)
```

```elixir
defmodule ReportsGenerator.ParserTest do
  use ExUnit.Case

  alias ReportsGenerator.Parser

  describe "parse_file/1" do
    test "parses the file" do
      file_name = "report_test.csv"

      response =
        file_name
        |> Parser.parse_file()
        |> Enum.map(& &1)

      expected_response = "banana"

      assert response == expected_response
    end
  end
end
```

Agora que usamos `Enum.map`, teremos o resultado de cada linha. Para rodar somente um teste passando o número da linha de código que está.

```bash
mix test test/parser_test.exs:7
#   1) test parse_file/1 parses the file (ReportsGenerator.ParserTest)
#      test/parser_test.exs:7
#      Assertion with == failed
#      code:  assert response == expected_response
#      left:  [["1", "pizza", 48], ["2", "açaí", 45], ["3", "hambúrguer", 31], ["4", "esfirra", 42], ["5", "hambúrguer", 49], ["6", "esfirra", 18], ["7", "pizza", 27], ["8", "esfirra", 25], ["9", "churrasco", 24], ["10", "churrasco", 36]]
#      right: "banana"
#      stacktrace:
#        test/parser_test.exs:17: (test)
```

Arrumando `expected_response`:

```elixir
defmodule ReportsGenerator.ParserTest do
  use ExUnit.Case

  alias ReportsGenerator.Parser

  describe "parse_file/1" do
    test "parses the file" do
      file_name = "report_test.csv"

      response =
        file_name
        |> Parser.parse_file()
        |> Enum.map(& &1)

      expected_response = [
        ["1", "pizza", 48],
        ["2", "açaí", 45],
        ["3", "hambúrguer", 31],
        ["4", "esfirra", 42],
        ["5", "hambúrguer", 49],
        ["6", "esfirra", 18],
        ["7", "pizza", 27],
        ["8", "esfirra", 25],
        ["9", "churrasco", 24],
        ["10", "churrasco", 36]
      ]

      assert response == expected_response
    end
  end
end
```

```bash
mix test test/parser_test.exs
# .

# Finished in 0.05 seconds
# 1 test, 0 failures
```

## Testando o ReportsGenerator

Vamos limpar o arquivo `test/reports_generator_test.exs` criado por padrão. Temos duas funções públicas em `ReportsGenerator` que são `build` e `fetch_higher_cost`.

No `build` não tem cenários de borda para testar. Apenas o cenário de buildar:

```elixir
defmodule ReportsGeneratorTest do
  use ExUnit.Case

  describe "build/1" do
    test "builds the report" do
      file_name = "report_test.csv"

      response = ReportsGenerator.build(file_name)

      expected_response = "banana"

      assert response == expected_response
    end
  end
end

```

```bash
mix test test/reports_generator_test.exs
  # 1) test build/1 builds the report (ReportsGeneratorTest)
  #    test/reports_generator_test.exs:5
  #    Assertion with == failed
  #    code:  assert response == expected_response
  #    left:  %{"foods" => %{"açaí" => 1, "churrasco" => 2, "esfirra" => 3, "hambúrguer" => 2, "pastel" => 0, "pizza" => 2, "prato_feito" => 0, "sushi" => 0}, "users" => %{"1" => 48, "10" => 36, "11" => 0, "12" => 0, "13" => 0, "14" => 0, "15" => 0, "16" => 0, "17" => 0, "18" => 0, "19" => 0, "2" => 45, "20" => 0, "21" => 0, "22" => 0, "23" => 0, "24" => 0, "25" => 0, "26" => 0, "27" => 0, "28" => 0, "29" => 0, "3" => 31, "30" => 0, "4" => 42, "5" => 49, "6" => 18, "7" => 27, "8" => 25, "9" => 24}}
  #    right: "banana"
```

Não ver como negativo ver um `expected_response` bem extenso.

```elixir
defmodule ReportsGeneratorTest do
  use ExUnit.Case

  describe "build/1" do
    test "builds the report" do
      # SETUP
      file_name = "report_test.csv"

      # EXERCISE
      response = ReportsGenerator.build(file_name)

      expected_response = %{
        "foods" => %{
          "açaí" => 1,
          "churrasco" => 2,
          "esfirra" => 3,
          "hambúrguer" => 2,
          "pastel" => 0,
          "pizza" => 2,
          "prato_feito" => 0,
          "sushi" => 0
        },
        "users" => %{
          "1" => 48,
          "10" => 36,
          "11" => 0,
          "12" => 0,
          "13" => 0,
          "14" => 0,
          "15" => 0,
          "16" => 0,
          "17" => 0,
          "18" => 0,
          "19" => 0,
          "2" => 45,
          "20" => 0,
          "21" => 0,
          "22" => 0,
          "23" => 0,
          "24" => 0,
          "25" => 0,
          "26" => 0,
          "27" => 0,
          "28" => 0,
          "29" => 0,
          "3" => 31,
          "30" => 0,
          "4" => 42,
          "5" => 49,
          "6" => 18,
          "7" => 27,
          "8" => 25,
          "9" => 24
        }
      }

      # ASSERTION
      assert response == expected_response
    end
  end
end
```

No `fetch_higher_cost` temos mais cenários para testar

```elixir
  describe "fetch_higher_cost/2" do
    test "when the option is 'users', returns the user who spent the most" do
      file_name = "report_test.csv"

      response = file_name
      |> ReportsGenerator.build()
      |> ReportsGenerator.fetch_higher_cost("users")

      expected_response = "banana"

      assert response == expected_response
    end
  end
```

Executando

```bash
mix test test/reports_generator_test.exs:60
  # 1) test fetch_higher_cost/2 when the option is 'users', returns the user who spent the most (ReportsGeneratorTest)
  #    test/reports_generator_test.exs:60
  #    Assertion with == failed
  #    code:  assert response == expected_response
  #    left:  {:ok, {"5", 49}}
  #    right: "banana"
  #    stacktrace:
  #      test/reports_generator_test.exs:69: (test)
```

E arrumando o teste:

```elixir
  describe "fetch_higher_cost/2" do
    test "when the option is 'users', returns the user who spent the most" do
      file_name = "report_test.csv"

      response = file_name
      |> ReportsGenerator.build()
      |> ReportsGenerator.fetch_higher_cost("users")

      expected_response = {:ok, {"5", 49}}

      assert response == expected_response
    end
  end
```

Vamos fazer para `foods` e para uma opção inválida

```elixir
defmodule ReportsGeneratorTest do
  use ExUnit.Case

  describe "build/1" do
    test "builds the report" do
      file_name = "report_test.csv"

      response = ReportsGenerator.build(file_name)

      expected_response = %{
        "foods" => %{
          "açaí" => 1,
          "churrasco" => 2,
          "esfirra" => 3,
          "hambúrguer" => 2,
          "pastel" => 0,
          "pizza" => 2,
          "prato_feito" => 0,
          "sushi" => 0
        },
        "users" => %{
          "1" => 48,
          "10" => 36,
          "11" => 0,
          "12" => 0,
          "13" => 0,
          "14" => 0,
          "15" => 0,
          "16" => 0,
          "17" => 0,
          "18" => 0,
          "19" => 0,
          "2" => 45,
          "20" => 0,
          "21" => 0,
          "22" => 0,
          "23" => 0,
          "24" => 0,
          "25" => 0,
          "26" => 0,
          "27" => 0,
          "28" => 0,
          "29" => 0,
          "3" => 31,
          "30" => 0,
          "4" => 42,
          "5" => 49,
          "6" => 18,
          "7" => 27,
          "8" => 25,
          "9" => 24
        }
      }

      assert response == expected_response
    end
  end

  describe "fetch_higher_cost/2" do
    test "when the option is 'users', returns the user who spent the most" do
      file_name = "report_test.csv"

      response = file_name
      |> ReportsGenerator.build()
      |> ReportsGenerator.fetch_higher_cost("users")

      expected_response = {:ok, {"5", 49}}

      assert response == expected_response
    end

    test "when the option is 'foods', returns the most consumed food" do
      file_name = "report_test.csv"

      response = file_name
      |> ReportsGenerator.build()
      |> ReportsGenerator.fetch_higher_cost("foods")

      expected_response = {:ok, {"esfirra", 3}}

      assert response == expected_response
    end

    test "when the as invalid option is given, returns an error" do
      file_name = "report_test.csv"

      response = file_name
      |> ReportsGenerator.build()
      |> ReportsGenerator.fetch_higher_cost("bananas")

      expected_response = {:error, "Invalid option!"}

      assert response == expected_response
    end
  end
end
```

O `ExUnit` roda todos os testes em paralelo, o que faz com que a switch seja muito rápida mesmo que tenha muitos testes.

```bash
mix test
# .....

# Finished in 0.07 seconds
# 5 tests, 0 failures
```

---

## Medindo o tempo de execução

Medimos o tempo de execução no Elixir pelo módulo `timer` do Erlang. No `iex` podemos ver as funções digitando `:timer.` e a tecla `tab`:

```elixir
:timer.
#..> apply_after/4       apply_interval/4    cancel/1
#..> code_change/3       exit_after/2        exit_after/3
#..> get_status/0        handle_call/3       handle_cast/2
#..> handle_info/2       hms/3               hours/1
#..> init/1              kill_after/1        kill_after/2
#..> minutes/1           now_diff/2          seconds/1
#..> send_after/2        send_after/3        send_interval/2
#..> send_interval/3     sleep/1             start/0
#..> start_link/0        tc/1                tc/2
#..> tc/3                terminate/2
```

Iremos usar a função [tc/1](https://erlang.org/doc/man/timer.html#tc-1) com aridade 1, que espera uma função anônima com a função em si que queremos executar para medir o tempo de execução.

Vamos executar no `iex`:

```elixir
:timer.tc(fn -> ReportsGenerator.build("report_complete.csv") end)
#..>{2508696,
#..> %{
#..>   "foods" => %{
#..>     "açaí" => 37742,
#..>     "churrasco" => 37650,
#..>     "esfirra" => 37462,
#..>     "hambúrguer" => 37577,
#..>     "pastel" => 37392,
#..>     "pizza" => 37365,
#..>     "prato_feito" => 37519,
#..>     "sushi" => 37293
#..>   },
#..>   "users" => %{
#..>     "1" => 278849,
#..>     "10" => 268317,
#..>     "11" => 268877,
#..>     "12" => 276306,
#..>     "13" => 282953,
#..>     "14" => 277084,
#..>     "15" => 280105,
#..>     "16" => 271831,
#..>     "17" => 272883,
#..>     "18" => 271421,
#..>     "19" => 277720,
#..>     "2" => 271031,
#..>     "20" => 273446,
#..>     "21" => 275026,
#..>     "22" => 278025,
#..>     "23" => 276523,
#..>     "24" => 274481,
#..>     "25" => 274512,
#..>     "26" => 274199,
#..>     "27" => 278001,
#..>     "28" => 274256,
#..>     "29" => 273030,
#..>     "3" => 272250,
#..>     "30" => 275978,
#..>     "4" => 277054,
#..>     "5" => 270926,
#..>     "6" => 272053,
#..>     "7" => 273112,
#..>     "8" => 275161,
#..>     "9" => 274003
#..>   }
#..> }}
```

onde o primeiro elemento da tupla (2508696) é o tempo que levou a execução em microssegundos.

Ao invés de consumir o `report_complete.csv`, vamos consumir de forma paralela concorrente os três arquivos e agregar o resultado num import só.

---

# O que há por trás dos processos em Elixir?

Processos, concorrência, paralelismo no Elixir.

## Processos

- [x] Os desafios de um sistema web atual
- [x] Processos devem ser isolados

## Processos na BEAM (máquina virtual da Erlang)

- [x] Escalabilidade
- [x] Tolerância a falhas
- [x] Distribuição

- [x] Processo do sistema operacional vs Processo da BEAM
- [x] Os processos da BEAM são muito leves
- [x] Criados em questões de microssegundos e utilizam poucos kB. Processos do OS gastam alguns MB.

- [x] Cada thread executa de forma concorrente
- [x] Um scheduler por núcleo da CPU
  - [x] A máquina virtual roda em um único processo do sistema operacional
  - [x] Cada scheduler pode criar milhares de processos. O limite teórico é de 134 milhões!

<img src="./elixir.png" width="400" />

## Concorrência vs Paralelismo

- [x] Voce e sua irmã jogando videogame

📚 Livro: [Elixir in Action](https://livebook.manning.com/book/elixir-in-action/chapter-1)

---

## Processos na prática, um exemplo simples

No `iex`, podemos ver como criar um processo:

```elixir
spawn fn -> IO.puts("banana") end
#..> banana
#..> #PID<0.108.0>
```

Criamos um processo que executou alguma coisa. `PID` é o _process id_. Vamos armazenar esse processo e utilizar o módulo [Process](https://hexdocs.pm/elixir/Process.html) para ver se esse processo ainda existe:

```elixir
pid = spawn fn -> IO.puts("banana") end
#..> banana
#..> #PID<0.110.0>
Process.alive?(pid)
#..> false
```

Em geral, um processo nasce, executa e encerra. Alguns outros processos, como `iex` continuam executando em um processo.

```elixir
self()
#..> #PID<0.106.0>
pid = self()
#..> #PID<0.106.0>
Process.alive?(pid)
#..> true
```

Processos em elixir seguem um modelo de atores, ou seja, criamos um processo e cada um dos processos tem uma caixa de mensagens e eles se comunicam entre si a todo momento enviando e recebendo mensagens, como se fossem e-mails.

`self()` retorna o `PID` do processo do `iex`. E vamos mandar uma mensagem para o `iex` com o `send`. E a mensagem pode ser qualquer dado.

```elixir
pid_iex = self()
#..> #PID<0.106.0>
send pid_iex, {:ok, "mensagem de sucesso"}
#..> {:ok, "mensagem de sucesso"}
```

Mas todo processo tem que ter definida uma função `receive` para receber uma mensagem.

```elixir
receive do
{:ok, msg} -> msg
{:error, _} -> "Deu ruim!"
end
#..> "mensagem de sucesso"
```

Uma vez definida a função `receive`, já existia uma mensagem na caixinha do meu processo, então essa mensagem foi consumida e exibiu `"mensagem de sucesso"`.

Vamos mandar uma mensagem de erro e consumir essa mensagem:

```elixir
send pid_iex, {:error, "mensagem de erro"}
#..> {:error, "mensagem de erro"}
receive do
{:ok, msg} -> msg
{:error, _} -> "Deu ruim!"
end
#..> "Deu ruim!"
```

Criar um processo com `spawn`, envia mensagem pelo `send` e recebe pelo `receive`.

Vamos criar um processo novo:

```elixir
spawn fn -> send(pid_iex, {:ok, "deu certo?"}) end
#..> #PID<0.129.0>
```

Vamos definir o `receive` de novo:

```elixir
receive do
{:ok, msg} -> msg
end
#..> "deu certo?"
```

---

## Criando nossa primeira versão paralela

Vamos usar o módulo [Task](https://hexdocs.pm/elixir/Task.html), que são conveniências para usar `spawn`. Podemos iniciar a task com `Task.async` e usar o callback com `Task.await`. Porém, a função `async_stream` funcionará melhor para nosso caso, que espera um Enum como primeiro argumento. E como é `stream`, ele vai ser avaliado como `lazy` e não precisamos usar o `await` nesse caso.

```elixir
  def build_from_many(filenames) do
    filenames
    |> Task.async_stream(&build/1)
    # outra forma implícita
    # |> Task.async_stream(&build(&1))
    # forma explícita
    # |> Task.async_stream(fn filename -> build(filename))
    |> Enum.map(& &1)
  end
```

Vamos ver como fica o `stream`. Como é `lazy`, então precisamos colocar o `Enum.map(& &1)` somente para ver. Vamos passar uma lista de arquivos.

```elixir
ReportsGenerator.build_from_many(["report_1.csv", "report_2.csv", "report_3.csv"])
#..> [
#..>   ok: %{
#..>     "foods" => %{
#..>       "açaí" => 12543,
#..>       "churrasco" => 12585,
#..>       "esfirra" => 12468,
#..>       "hambúrguer" => 12593,
#..>       "pastel" => 12449,
#..>       "pizza" => 12426,
#..>       "prato_feito" => 12486,
#..>       "sushi" => 12450
#..>     },
#..>     "users" => %{
#..>       "1" => 94115,
#..>       "10" => 88751,
#..>       "11" => 89791,
#..>       "12" => 92811,
#..>       "13" => 92902,
#..>       "14" => 90163,
#..>       "15" => 95196,
#..>       "16" => 91100,
#..>       "17" => 93569,
#..>       "18" => 91691,
#..>       "19" => 90914,
#..>       "2" => 92473,
#..>       "20" => 90303,
#..>       "21" => 90351,
#..>       "22" => 94393,
#..>       "23" => 94706,
#..>       "24" => 90921,
#..>       "25" => 90969,
#..>       "26" => 89000,
#..>       "27" => 90489,
#..>       "28" => 93574,
#..>       "29" => 88447,
#..>       "3" => 90693,
#..>       "30" => 91738,
#..>       "4" => 91394,
#..>       "5" => 91247,
#..>       "6" => 90687,
#..>       "7" => 93026,
#..>       "8" => 92928,
#..>       "9" => 91770
#..>     }
#..>   },
#..>   ok: %{
#..>     "foods" => %{
#..>       "açaí" => 12695,
#..>       "churrasco" => 12468,
#..>       "esfirra" => 12386,
#..>       "hambúrguer" => 12520,
#..>       "pastel" => 12533,
#..>       "pizza" => 12462,
#..>       "prato_feito" => 12471,
#..>       "sushi" => 12465
#..>     },
#..>     "users" => %{
#..>       "1" => 90208,
#..>       "10" => 87816,
#..>       "11" => 89253,
#..>       "12" => 93040,
#..>       "13" => 94096,
#..>       "14" => 94690,
#..>       "15" => 91322,
#..>       "16" => 91196,
#..>       "17" => 88062,
#..>       "18" => 91552,
#..>       "19" => 93248,
#..>       "2" => 89401,
#..>       "20" => 91386,
#..>       "21" => 91871,
#..>       "22" => 91527,
#..>       "23" => 89308,
#..>       "24" => 91314,
#..>       "25" => 94039,
#..>       "26" => 91918,
#..>       "27" => 93380,
#..>       "28" => 89932,
#..>       "29" => 93527,
#..>       "3" => 90975,
#..>       "30" => 93089,
#..>       "4" => 93918,
#..>       "5" => 91428,
#..>       "6" => 91607,
#..>       "7" => 90088,
#..>       "8" => 89736,
#..>       "9" => 91542
#..>     }
#..>   },
#..>   ok: %{
#..>     "foods" => %{
#..>       "açaí" => 12504,
#..>       "churrasco" => 12597,
#..>       "esfirra" => 12608,
#..>       "hambúrguer" => 12464,
#..>       "pastel" => 12410,
#..>       "pizza" => 12477,
#..>       "prato_feito" => 12562,
#..>       "sushi" => 12378
#..>     },
#..>     "users" => %{
#..>       "1" => 94526,
#..>       "10" => 91750,
#..>       "11" => 89833,
#..>       "12" => 90455,
#..>       "13" => 95955,
#..>       "14" => 92231,
#..>       "15" => 93587,
#..>       "16" => 89535,
#..>       "17" => 91252,
#..>       "18" => 88178,
#..>       "19" => 93558,
#..>       "2" => 89157,
#..>       "20" => 91757,
#..>       "21" => 92804,
#..>       "22" => 92105,
#..>       "23" => 92509,
#..>       "24" => 92246,
#..>       "25" => 89504,
#..>       "26" => 93281,
#..>       "27" => 94132,
#..>       "28" => 90750,
#..>       "29" => 91056,
#..>       "3" => 90582,
#..>       "30" => 91151,
#..>       "4" => 91742,
#..>       "5" => 88251,
#..>       "6" => 89759,
#..>       "7" => 89998,
#..>       "8" => 92497,
#..>       "9" => 90691
#..>     }
#..>   }
#..> ]
```

Como estamos rodando em paralelo, o primeiro elemento é um `:ok`, ou seja, a operação aconteceu com sucesso. E o segundo é um map, que é o retorno da função `build_from_many`. Como executamos 3 arquivos ao mesmo tempo, precisamos aglutinar esse resultado. Ao invés de 3 maps, queremos apenas 1 map com o resultado completo. Então, vamos usar a função `Enum.reduce`.

Como a função sum_values pegava apenas os valores de um único map, temos que criar outra função `sum_reports1` que some o map que está no accumulator com o map atual que está sendo lido, sendo `report` aquele map inicialmente vazio.

```elixir
  def build_from_many(filenames) do
    filenames
    |> Task.async_stream(&build/1)
    |> Enum.reduce(report_acc(), fn {:ok, result}, report -> sum_reports(report, result) end)
  end
```

Essa função vai somar `foods1` com `foods2` e `users1` com `users2`. Felizmente, já existe uma função `Map.merge` que recebe 2 mapas e podemos falar para juntar os 2 mapas seguindo uma função determinada.

Não nos importamos com as chaves `_key` que seriam, por exemplo, `açaí`.

```elixir
  defp sum_reports(%{"foods" => foods1, "users" => users1}, %{"foods" => foods2, "users" => users2}) do
    foods = Map.merge(foods1, foods2, fn _key, value1, value2 -> value1 + value2 end)
    users = Map.merge(users1, users2, fn _key, value1, value2 -> value1 + value2 end)

    %{"users" => users, "foods" => foods}
  end
```

Para evitar repetir o `Map.merge`, vamos extraí-lo numa função anônima chamada `merge_maps`, que vai mergear 2 mapas somando seus valores.

```elixir
  defp sum_reports(%{"foods" => foods1, "users" => users1}, %{"foods" => foods2, "users" => users2}) do
    foods = merge_maps(foods1, foods2)
    users = merge_maps(users1, users2)

    %{"users" => users, "foods" => foods}
  end

  defp merge_maps(map1, map2) do
    Map.merge(map1, map2, fn _key, value1, value2 -> value1 + value2 end)
  end
```

Vamos recompilar no `iex` e verificar o resultado

```elixir
ReportsGenerator.build_from_many(["report_1.csv", "report_2.csv", "report_3.csv"])
#..> %{
#..>   "foods" => %{
#..>     "açaí" => 37742,
#..>     "churrasco" => 37650,
#..>     "esfirra" => 37462,
#..>     "hambúrguer" => 37577,
#..>     "pastel" => 37392,
#..>     "pizza" => 37365,
#..>     "prato_feito" => 37519,
#..>     "sushi" => 37293
#..>   },
#..>   "users" => %{
#..>     "1" => 278849,
#..>     "10" => 268317,
#..>     "11" => 268877,
#..>     "12" => 276306,
#..>     "13" => 282953,
#..>     "14" => 277084,
#..>     "15" => 280105,
#..>     "16" => 271831,
#..>     "17" => 272883,
#..>     "18" => 271421,
#..>     "19" => 277720,
#..>     "2" => 271031,
#..>     "20" => 273446,
#..>     "21" => 275026,
#..>     "22" => 278025,
#..>     "23" => 276523,
#..>     "24" => 274481,
#..>     "25" => 274512,
#..>     "26" => 274199,
#..>     "27" => 278001,
#..>     "28" => 274256,
#..>     "29" => 273030,
#..>     "3" => 272250,
#..>     "30" => 275978,
#..>     "4" => 277054,
#..>     "5" => 270926,
#..>     "6" => 272053,
#..>     "7" => 273112,
#..>     "8" => 275161,
#..>     "9" => 274003
#..>   }
#..> }
```

Comparado com o arquivo contendo a versão completa e ver se dá o mesmo resultado

```elixir
ReportsGenerator.build("report_complete.csv")
#..> %{
#..>   "foods" => %{
#..>     "açaí" => 37742,
#..>     "churrasco" => 37650,
#..>     "esfirra" => 37462,
#..>     "hambúrguer" => 37577,
#..>     "pastel" => 37392,
#..>     "pizza" => 37365,
#..>     "prato_feito" => 37519,
#..>     "sushi" => 37293
#..>   },
#..>   "users" => %{
#..>     "1" => 278849,
#..>     "10" => 268317,
#..>     "11" => 268877,
#..>     "12" => 276306,
#..>     "13" => 282953,
#..>     "14" => 277084,
#..>     "15" => 280105,
#..>     "16" => 271831,
#..>     "17" => 272883,
#..>     "18" => 271421,
#..>     "19" => 277720,
#..>     "2" => 271031,
#..>     "20" => 273446,
#..>     "21" => 275026,
#..>     "22" => 278025,
#..>     "23" => 276523,
#..>     "24" => 274481,
#..>     "25" => 274512,
#..>     "26" => 274199,
#..>     "27" => 278001,
#..>     "28" => 274256,
#..>     "29" => 273030,
#..>     "3" => 272250,
#..>     "30" => 275978,
#..>     "4" => 277054,
#..>     "5" => 270926,
#..>     "6" => 272053,
#..>     "7" => 273112,
#..>     "8" => 275161,
#..>     "9" => 274003
#..>   }
#..> }
```

---

## Medindo o tempo de execução da versão paralela

Vamos rodar o arquivo contendo 300 mil linhas:

```elixir
:timer.tc(fn -> ReportsGenerator.build("report_complete.csv") end)
#..> {4468670,
#..>  %{
#..>    "foods" => %{
#..>      "açaí" => 37742,
#..>      "churrasco" => 37650,
#..>      "esfirra" => 37462,
#..>      "hambúrguer" => 37577,
#..>      "pastel" => 37392,
#..>      "pizza" => 37365,
#..>      "prato_feito" => 37519,
#..>      "sushi" => 37293
#..>    },
#..>    "users" => %{
#..>      "1" => 278849,
#..>      "10" => 268317,
#..>      "11" => 268877,
#..>      "12" => 276306,
#..>      "13" => 282953,
#..>      "14" => 277084,
#..>      "15" => 280105,
#..>      "16" => 271831,
#..>      "17" => 272883,
#..>      "18" => 271421,
#..>      "19" => 277720,
#..>      "2" => 271031,
#..>      "20" => 273446,
#..>      "21" => 275026,
#..>      "22" => 278025,
#..>      "23" => 276523,
#..>      "24" => 274481,
#..>      "25" => 274512,
#..>      "26" => 274199,
#..>      "27" => 278001,
#..>      "28" => 274256,
#..>      "29" => 273030,
#..>      "3" => 272250,
#..>      "30" => 275978,
#..>      "4" => 277054,
#..>      "5" => 270926,
#..>      "6" => 272053,
#..>      "7" => 273112,
#..>      "8" => 275161,
#..>      "9" => 274003
#..>    }
#..>  }}
```

E agora paralelamente com 3 arquivos de 100 mil linhas cada

```elixir
:timer.tc(fn -> ReportsGenerator.build_from_many(["report_1.csv", "report_2.csv", "report_3.csv"]) end)
#..> {1360110,
#..>  %{
#..>    "foods" => %{
#..>      "açaí" => 37742,
#..>      "churrasco" => 37650,
#..>      "esfirra" => 37462,
#..>      "hambúrguer" => 37577,
#..>      "pastel" => 37392,
#..>      "pizza" => 37365,
#..>      "prato_feito" => 37519,
#..>      "sushi" => 37293
#..>    },
#..>    "users" => %{
#..>      "1" => 278849,
#..>      "10" => 268317,
#..>      "11" => 268877,
#..>      "12" => 276306,
#..>      "13" => 282953,
#..>      "14" => 277084,
#..>      "15" => 280105,
#..>      "16" => 271831,
#..>      "17" => 272883,
#..>      "18" => 271421,
#..>      "19" => 277720,
#..>      "2" => 271031,
#..>      "20" => 273446,
#..>      "21" => 275026,
#..>      "22" => 278025,
#..>      "23" => 276523,
#..>      "24" => 274481,
#..>      "25" => 274512,
#..>      "26" => 274199,
#..>      "27" => 278001,
#..>      "28" => 274256,
#..>      "29" => 273030,
#..>      "3" => 272250,
#..>      "30" => 275978,
#..>      "4" => 277054,
#..>      "5" => 270926,
#..>      "6" => 272053,
#..>      "7" => 273112,
#..>      "8" => 275161,
#..>      "9" => 274003
#..>    }
#..>  }}
```

Reduzimos o tempo.
