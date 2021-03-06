# Krawler MPI

Web crawler distribuído para sites de e-commerce

## Objetivo

Um *Web Crawler* tem como objetivo recuperar informações de uma ou mais páginas web de maneira inteligente, percorrendo caminhos criados pelos links de cada página. Neste projeto, o objetivo é criar um [Web Crawler](https://en.wikipedia.org/wiki/Web_crawler) capaz de recuperar informações de produtos disponíveis em sites de e-commerce e disponibilizá-las para o usuário, semelhante ao funcionamento do site [Buscapé](https://www.buscape.com.br/). Este *crawler* deve ser implementado de maneira distribuída, afim de diminuir o tempo de execução do programa, criando assim um sistema amigável para o usuário.

A biblioteca [MPI](https://www.open-mpi.org/) será utilizada para realizar esta implementação distribuída, mais especificamente a versão disponibilizada pelo framework [Boost](https://www.boost.org/). A distribuição do programa se dá por meio da utilização de diversos processos e troca de mensagens entre estes processos, ações facilitadas pelo MPI. O intuito deste tipo de implementação é permitir a execução do programa em sistemas distribuídos, como por exemplo um *cluster* no [AWS](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_clusters.html).

## Dependências
- Boost
- MPI
- GCC/G++
- CMake
- Make
- Python 3.7+
- matplotlib (Python)

## MPI

O MPI (Message Passing Interface) é uma biblioteca criada para permitir o desenvolvimento de programas que utilizam diversos processos de um ou mais computadores para executar tarefas paralelamente. Este protocolo funciona com trocas de mensagens (dados) entre processos a partir de um processo mestre. Neste projeto, queremos fazer com que diversas URLs sejam acessadas ao mesmo tempo, portanto o objetivo do MPI será dividir URLs para cada processo, que por sua vez irão realizar a coleta de produtos e enviá-los a um processo principal (normalmente o 0 (zero)) para que sejam mostrados. os processos podem ser criados na mesma máquina ou em máquinas diferentes. Caso esteja rodando num *cluster*, um arquivo contendo hosts pode ser especificado. A utilização com *clusters* não será detalhada neste relatório, mas pode ser facilmente encontrada na Internet.

## Funcionamento

As tarefas realizadas por cada processo são bastante simples:
1. Realizar o download de páginas web.
2. Coletar informações de produtos utilizando [Expressões Regulares](https://en.wikipedia.org/wiki/Regular_expression).
3. Mostrar as informações de cada produto em formato [JSON](https://www.json.org/).

Primeiramente, o processo 0 (zero) recebe um lista com diversas URLs. Esta lista será quebrada quase igualmente entre N processos, definidos pelo usuário que executa o programa. Cada lista menor menor será enviada para um outro processo, onde as URLs serão percorridas e utilizadas para baixar diferentes produtos.

##### Exemplo
Suponha que a lista inicial de URLs possua 40 links distintos.
A lista inicial tem o seguinte formato:

```python
[url1, url2, url3, url4, ..., url40]
```

Suponha também que colocaremos 4 processos para serem executados. Uma nova lista será criada contendo 4 elementos. Cada elemento é um novo vetor com 10 URLs:

```python
[
    [url1,  ..., url10],
    [url11, ..., url20],
    [url21, ..., url31],
    [url31, ..., url40]
]
```

Após esta separação, utilizamos a função `boost::mpi::scatter()` para enviar cada elemento a um processo diferente. O primeiro vetor (URLs 1-10) será enviado para o processo de mesmo índice de sua posição na lista, ou seja, 0 (zero). Já o segundo vetor (URLs 11-20) será enviado para o processo 1, e assim por diante.

Cada processo utilizará seu conjunto de URLs para coletar os produtos, podendo navegar para outros links afim de facilitar a coleta. Cada produto será composto pelas seguintes informações: nome, descrição, foto, preço, preço parcelado, número de parcelas, categoria e URL específica do produto, formatados em um JSON conforme abaixo:

```json
{
    "nome": "Produto",
    "descricao": "Descrição do Produto",
    "foto": "http://...",
    "preco": 1000,
    "preco_parcelado": 1000,
    "preco_num_parcelas": 10,
    "categoria": "Categoria",
    "url": "http://..."
}
```

As informações são obtidas por meio do uso de expressões regulares que as detectam em meio ao conteúdo HTML de cada página. Após a finalização de todas as URLs de todos os processos, a função `boost::mpi::gather()` é utilizada para agrupar novamente os produtos gerados, em um processo escolhido pelo usuário (novamente, normalmente o 0 (zero)). Após agrupados, os produtos serão mostrados ao usuário via saída padrão (terminal). Abaixo est um esquema da troca de mensagens entre processos:

![MPI Schema](https://i.imgur.com/9BagN78.png)

## Métricas

Afim de explorar a performance deste programa, algumas métricas de tempo são coletadas ao longo do *crawling*. São elas:

- Tempo ocioso (TOTAL_IDLE_TIME): tempo de espera pelo download de páginas
- Tempo médio ocioso (AVG_IDLE_TIME): tempo médio de espera pelo download de páginas
- Tempo por produto (PROD_TIME): tempo gasto para baixar uma página específica de um produto e criar sua visualização em JSON. Cada produto baixado possui uma linha com o tempo gasto.
- Tempo médio por produto (AVG_TIME_PER_PRODUCT): tempo total de execução dividido pelo número de produtos coletados.

Outras métricas coletadas:
- Tempo total de execução (TOTAL_EXEC_TIME).
- Número de produtos coletados (TOTAL_PROD_COUNT).

##### Nota: a métrica AVG_IDLE_TIME foi introduzida porque a métrica TOTAL_IDLE_TIME mostra o tempo total de downloads de páginas de maneira sequencial (soma de todos os tempos), o que não demonstra a realidade numa execução paralela (páginas sendo baixadas ao mesmo tempo). Como não é possível identificar com precisão o tempo de downloads *paralelos*, foi introduzido um cálculo de tempo médio de download. 

Todas as métricas são enviadas para a saída de erro padrão do sistema, que pode ser redirecionada para um arquivo de texto a gosto do usuário, como no exemplo a seguir (todos os valores estçao em segundos, com exceção de TOTAL_PROD_COUNT que é uma contagem):

```
1.2
0.85
0.50
...
TOTAL_PROD_COUNT: 200
AVG_IDLE_TIME: 15
TOTAL_IDLE_TIME: 180
TOTAL_EXEC_TIME: 30
AVG_TIME_PER_PRODUCT: 0.7
```

## Utilização

#### Sites suportados
- [Magazine Luiza](https://www.magazineluiza.com.br/)

O programa utiliza categorias de produtos como entrada. A categoria é uma URL de um site suportado. No caso do Magazine Luiza, a URL deve ser obitda navegando para o site, clicando em **Todos os Departamentos** na parte superior esquerda, e escolhendo um dos links destacados pela área vermelha:

<img src="https://i.imgur.com/ief76w1.png" alt="" width="50%" height="50%" />

Na próxima página clique em um dos links destacados novamente pela área vermelha:

<img src="https://i.imgur.com/FNuQU2P.png" alt="" width="50%" height="50%" />

A URL da págna que abrirá é a que deve ser utilizada:

<img src="https://i.imgur.com/NmOzgTQ.png" alt="" width="50%" height="50%" />

O teste acima foi realizado clicando em: **Todos os Departamentos > Celulares > Smartphones** obtendo a URL **https://www.magazineluiza.com.br/smartphone/celulares-e-smartphones/s/te/tcsp/**

Após selecionada a URL, o programa deve ser compilado da seguinte maneira:

```sh
$ cd src/build/
$ /usr/bin/cmake -DCMAKE_EXPORT_COMPILE_COMMANDS=ON -DCMAKE_BUILD_TYPE=Debug -G 'Unix Makefiles' </caminho/até/krawler_mpi/src>
$ make
```

Um programa em python foi criado para executar o programa (localizado em `test/`), para realizar diversos testes e criar gráficos de tempo. Para utilizá-lo, navegue até a pasta `test/` e rode o seguinte comando:
```sh
$ python run.py <max_procs> <iter> <url>
```

em que `<max_procs>` é o número máximo de processos a serem utilizados - serão feitos testes utilizando de 1 até `<max_procs>` processos - `<iter>` é o número de iterações com a mesma quantidade de processos, afim de obter medidas mais precisas e `<url>` é a URL da categoria, como explicado acima.

Este programa está preparado para rodar num cluster especifico criado para este projeto. Caso deseje rodar num cluster próprio, edite as lista `HOSTNAMES` e `HOSTDESCS` no arquivo `run.py` e coloque os dados correspondentes às máquinas de seu cluster.

#### Arquivos de teste

As métricas estarão na pasta `test/files/`, com cada arquivo estando no formato `iters_mX_pY_iZ`, onde X é a quantidade de máquinas utilizadas para aquele teste, Y é a quantidade de processos utilizada para aquele teste e Z é a iteração. Por exemplo:

> O arquivo `iters_m2_p4_i0` mostra os resultados da primeira iteração de uma execução que utilizou 2 máquinas e 4 processos. O arquivo `iters_m2_p4_i1` mostra os resultados da segunda iteração da mesma execução.

Cada máquina suporta um máximo de dois processos, sendo esta uma escolha do desenvolvedor. O arquivo `hosts`, presente na pasta `test/`, mostra os *hosts* (máquinas) utilizados pela última execução e a quantidade de processos (*slots*) disponíveis em cada um. Por algum motivo ainda não descoberto, a execução com mais de 7 processos não é possvel, gerando um erro do tipo *Segmentation Fault*.

#### Gráficos

Caso o usuário possua a biblioteca de python `matplotlib`, 5 gráficos podem ser gerados com o programa `make_charts.py`, passando a quantidade de processos máximos a serem analisado (e.g. se o usuário passar 6, haverá 6 pontos de dado em cada gráfico). É esperado que a métrica TOTAL_PROD_COUNT se mantenha constante independentemente do número de processos, indicando que a mesma quantidade de produtos foi recuperada em todos os testes.

Utilização:
```sh
$ python make_charts.py <max_proc>
```

## Resultados
Para demonstrar o programa foram gerados alguns testes utilizando a URL https://www.magazineluiza.com.br/artesanato/armarinhos/s/am/arsa/, categoria que não possui quantidade exagerada de produtos mas é grande o suficiente para o teste. Os testes foram realizados em um cluster AWS com instância rodando sistema Ubuntu 18.04.1, utilizando de 1 até 7 processos e de 1 até 4 máquinas, com máximo de 2 processos por máquina. Cada execução com um determinado número de processos foi realizada com 5 iterações, resultando em 42 iterações no total.

O programa apresentará erros se a quantidade de páginas a serem buscadas for menor do que a quantidade de processos passados.

Os gŕaficos gerados estão abaixo, sendo que o eixo horizontal representa número de processos e o eixo vertical representa tempo em segundos, exceto no gráfico TOTAL_PROD_COUNT, em que o eixo vertical representa quantidade:

![TOTAL_PROD_COUNT](https://i.imgur.com/Pg3t1rj.png)

Como mencionado, o gráfico é constante, mostrando que a mesma quantidade de produtos foi coletada em todos os testes.

![TOTAL_EXEC_TIME](https://i.imgur.com/T39FKeL.png)
![AVG_TIME_PER_PRODUCT](https://i.imgur.com/Dwig3nH.png)
![AVG_IDLE_TIME](https://i.imgur.com/QLRFjvn.png)

Os três gráficos acima mostram uma melhora bastante significativa conforme a quantidade de processos aumenta. No foi possvel conferir se um número maior de processos diminuiria ainda mais os tempos, por conta dos erros presentes ao utilizar mais de 7 processos.
