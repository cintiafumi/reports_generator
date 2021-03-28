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
