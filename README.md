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
