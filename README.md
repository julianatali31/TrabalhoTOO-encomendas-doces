# Sistema de Encomendas de Confeitaria

**Trabalho Prático — Tecnologia Orientada a Objetos (TOO)**
Bacharelado em Ciência da Computação · Prof. Mateus de Freitas Invernisi
Autora: Júlia Natali da Silva

---

## Tema e objetivo

O sistema controla as **encomendas de uma confeitaria**: cadastro de ingredientes com
custo e estoque, fichas técnicas de receita, catálogo de produtos com cálculo de
margem, descontos e o acompanhamento de cada encomenda do pedido até a entrega.

A pergunta que o sistema responde é simples de fazer e difícil de calcular à mão:
**quanto custa, de verdade, um cookie, e quanto sobra quando eu vendo uma dúzia com
desconto?**

Isso é difícil porque os ingredientes são comprados em embalagem (pacote de 5 kg,
cartela de 30 ovos) e usados em gramas e unidades; uma receita rende 15 cookies, então
o custo precisa ser dividido pelo rendimento; e quando o preço da manteiga sobe, toda
receita que leva manteiga muda de custo e todo produto feito com ela muda de margem.

O tema vem de uma confeitaria real, para a qual eu mantenho um sistema de gestão maior.
Os preços de compra e a receita do Cookie Tradicional usados na demonstração são os
verdadeiros — não são dados de exemplo. **O código deste repositório, no entanto, foi
escrito do zero para este trabalho:** é um subsistema novo, independente, em Python
puro, sem banco de dados e sem framework.

---

## Como executar

Requer apenas **Python 3.10 ou superior**. Nenhuma dependência externa, nenhum banco de
dados, nenhuma configuração.

Na raiz do projeto:

```bash
python main.py
```

A demonstração percorre, em ordem, cada pilar da POO e cada padrão de projeto,
imprimindo o resultado no terminal. Ela monta o estoque, calcula a ficha técnica do
Cookie Tradicional (custo de R$ 28,20 por fornada de 15 unidades, R$ 1,88 por cookie),
cria produtos pela fábrica, compara estratégias de desconto e leva uma encomenda do
pedido até a entrega.

### Testes

```bash
pip install pytest
python -m pytest testes/ -v
```

São **115 testes**, em arquivos que correspondem aos critérios de avaliação:

| Arquivo | O que verifica |
| :--- | :--- |
| `testes/test_pilares.py` | Os quatro pilares: encapsulamento, abstração, herança, polimorfismo |
| `testes/test_factory.py` | Padrão Factory: criação e extensão de tipos de produto |
| `testes/test_descontos.py` | Padrão Strategy: as quatro estratégias de desconto |
| `testes/test_estados.py` | Padrão State: o ciclo de vida completo da encomenda |

---

## Estrutura

```
doceria/
├── erros.py        ErroDeDominio e 8 subclasses
├── ingrediente.py  Ingrediente
├── receita.py      Receita, ItemReceita
├── produto.py      Produto, DoceDeReceita, DoceRevenda, ProdutoFactory
├── descontos.py    EstrategiaDesconto, SemDesconto, DescontoPercentual,
│                   DescontoPorQuantidade, CupomValorFixo
├── estados.py      EstadoPedido, Novo, EmProducao, Pronto, Entregue, Cancelado
└── pedido.py       Pedido, ItemPedido
```

São **389 linhas de código** e 29 classes: 20 de domínio e 9 de exceção. O enunciado
pede no mínimo 5 classes principais; as principais aqui são `Ingrediente`, `Receita`,
`Produto`, `Pedido` e `EstadoPedido`, e as outras são as subclasses e os objetos de
apoio delas. O restante dos arquivos é documentação explicando a origem de cada
decisão de projeto.

### Descrição de cada classe

**`Ingrediente`** — insumo com nome, unidade de medida, custo por unidade e estoque.
O construtor nomeado `comprado_em_embalagem()` recebe o preço pago e o tamanho da
embalagem ("R$ 20,00 no pacote de 5000 g") e calcula o custo por grama sozinho. O
estoque é privado e só muda por `baixar()` ou `repor()`.

**`ItemReceita`** — a ligação entre uma receita e um ingrediente, com a quantidade
usada. Imutável; sabe calcular o próprio custo.

**`Receita`** — ficha técnica com rendimento. Calcula `custo_total()`,
`custo_por_unidade()` e a `composicao()` ordenada pelo peso de cada ingrediente no
custo. `consumir()` e `devolver()` movimentam o estoque de todos os ingredientes de
uma vez.

**`Produto`** *(abstrata)* — qualquer item do catálogo. Implementa as contas de margem
uma única vez e **obriga** cada subclasse a declarar de onde vem o próprio custo.

**`DoceDeReceita`** — doce que a confeitaria produz; o custo vem da receita dividido
pelo rendimento.

**`DoceRevenda`** — item comprado pronto e revendido; o custo é o preço de compra.

**`ProdutoFactory`** — cria o produto certo a partir do tipo pedido, sem que quem
chama conheça as subclasses.

**`EstrategiaDesconto`** *(abstrata)* — uma regra de desconto. Recebe subtotal e
quantidade total e devolve quantos reais descontar.

**`SemDesconto`**, **`DescontoPercentual`**, **`DescontoPorQuantidade`**,
**`CupomValorFixo`** — as quatro regras implementadas.

**`EstadoPedido`** *(abstrata)* — o que a encomenda aceita no momento. Recusa tudo por
padrão.

**`Novo`**, **`EmProducao`**, **`Pronto`**, **`Entregue`**, **`Cancelado`** — os cinco
estados concretos.

**`ItemPedido`** — uma linha da encomenda, com preço e custo congelados no momento do
pedido. Imutável.

**`Pedido`** — a encomenda. Delega toda decisão ao estado atual.

**`ErroDeDominio`** e suas 8 subclasses — hierarquia de exceções que permite à camada
externa capturar qualquer violação de regra de negócio de uma vez.

---

## Os pilares da POO

### Abstração

`Receita` é o exemplo mais direto. Quem precisa saber quanto custa um cookie chama
`receita.custo_por_unidade()` e recebe um número. Não precisa saber que por trás disso
existe uma soma sobre uma lista de itens, uma multiplicação por ingrediente e uma
divisão pelo rendimento. A complexidade existe, mas fica do lado de dentro.

As quatro classes abstratas — `Produto`, `EstrategiaDesconto`, `EstadoPedido` e a base
`ErroDeDominio` — usam `ABC` e `@abstractmethod` para definir contratos sem
implementação. Elas dizem *o que* toda subclasse precisa responder, sem dizer *como*.

### Encapsulamento

O estoque do ingrediente é o caso central. `_estoque` é privado, exposto por uma
property **somente leitura**, e só muda dentro de `baixar()` ou `repor()`. Esses
métodos validam antes de alterar:

```python
farinha.estoque = -500      # AttributeError: property has no setter
farinha.baixar(99999)       # EstoqueInsuficiente
farinha.baixar(-500)        # QuantidadeInvalida
```

Sem isso, `baixar(-500)` seria uma forma de repor estoque sem registro, e
`estoque = -500` deixaria o objeto num estado que não existe no mundo real.
Encapsular não é esconder por esconder: é garantir que o objeto **nunca fique
inválido**.

`ItemReceita` e `ItemPedido` levam isso mais longe com `frozen=True`: não é que seja
difícil alterá-los, é impossível. E `Receita.itens` e `Pedido.itens` devolvem **tupla,
não lista** — assim ninguém insere um item por fora, driblando a validação do estado.

### Herança

Três hierarquias, cada uma por um motivo diferente:

- **`ErroDeDominio`** → 8 subclasses. Herança apenas de *classificação*: o que as
  subclasses ganham é poder ser capturadas juntas.
- **`Produto`** → `DoceDeReceita` e `DoceRevenda`. Herança de *comportamento*: as
  subclasses recebem `margem_valor()`, `margem_percentual()` e `no_prejuizo()`
  prontas, escritas uma única vez, e implementam apenas `custo_unitario()`.
- **`EstadoPedido`** → 5 estados. Herança de *política*: a base recusa todas as
  operações, e cada estado sobrescreve só o que permite.

### Polimorfismo

O teste `test_o_mesmo_laco_funciona_para_os_dois_tipos` percorre duas classes
diferentes chamando os mesmos métodos, **sem um único `isinstance` e sem um único
`if`**:

```python
for produto in [cookie, refrigerante]:
    print(produto.nome, produto.custo_unitario(), produto.margem_percentual())
```

Cada classe responde `custo_unitario()` do seu jeito: dividindo o custo da receita
pelo rendimento, ou devolvendo o preço de compra.

O mesmo acontece em `Pedido._reservar_insumos()`. O laço **não pergunta** se o produto
tem receita:

```python
for item in self._itens:
    item.produto.reservar_insumos(item.quantidade)
```

`DoceDeReceita` consome ingredientes; `DoceRevenda` não faz nada. Daqui, os dois
parecem iguais.

---

## Padrões de projeto

### Factory — criação dos produtos

**Visto em aula.** Implementado em `doceria/produto.py`.

Uma confeitaria vende duas coisas diferentes: o que ela produz e o que ela revende. As
duas têm nome, preço e margem, mas o custo vem de lugares completamente diferentes.
Sem fábrica, quem cadastra produto precisaria conhecer as classes concretas:

```python
if tipo == "receita":
    produto = DoceDeReceita(nome, preco, receita)
elif tipo == "revenda":
    produto = DoceRevenda(nome, preco, custo_compra)
```

Esse `if` apareceria em todo lugar que cria produto: no cadastro, na importação de
planilha, nos testes. Um tipo novo exigiria achar e editar todos eles.

A fábrica concentra a decisão em um lugar. Quem chama passa o tipo como texto e recebe
um `Produto` pronto:

```python
cookie = ProdutoFactory.criar("receita", "Cookie Tradicional", "5.00", receita=receita)
refri  = ProdutoFactory.criar("revenda", "Refrigerante", "6.00", custo_compra="3.20")
```

A fábrica também valida o que o `if` não validava: tipo inexistente é recusado com a
lista dos válidos, e um produto de receita criado sem receita é recusado com uma
mensagem que explica o problema, em vez de um `TypeError` do construtor.

`registrar_tipo()` permite um tipo novo **sem alterar nenhuma classe existente** — é o
**Princípio Aberto-Fechado**: aberto para extensão, fechado para modificação. O teste
`test_tipo_novo_sem_alterar_nenhuma_classe_existente` cria um `DoceDoado` e o registra
em tempo de execução, sem tocar em `Produto` nem na fábrica.

### State — ciclo de vida da encomenda

**Visto em aula.** Implementado em `doceria/estados.py`.

```
Novo --confirmar--> EmProducao --marcar_pronto--> Pronto --entregar--> Entregue
 |                      |                           |
 +----------------------+---------------------------+--> Cancelado
```

Dá para fazer isso com uma string e uns ifs:

```python
if pedido.status == "novo":
    pedido.itens.append(item)
elif pedido.status == "entregue":
    raise Exception("pedido ja entregue")
```

Dois problemas sérios:

1. **A regra fica espalhada.** Para saber o que uma encomenda "em produção" aceita, é
   preciso procurar todos os `if pedido.status` do projeto. A resposta não está em
   nenhum lugar: está distribuída.
2. **Esquecer um `if` não dá erro.** Se um caminho novo altera os itens e ninguém
   lembrou de checar o status, uma encomenda já entregue muda de valor em silêncio.

O padrão State resolve dando uma classe a cada etapa. A encomenda não pergunta "qual o
meu status?": ela delega.

```python
def confirmar(self):
    return self._estado.confirmar(self)
```

A decisão de projeto mais importante está na classe base: **todos os métodos recusam
por padrão**, e cada estado sobrescreve apenas o que permite. Esquecer de implementar
algo resulta numa operação bloqueada com mensagem clara, nunca numa operação indevida
autorizada. Negar por padrão é o mesmo raciocínio de uma regra de firewall.

| Estado | Aceita item | Confirma | Marca pronto | Entrega | Cancela |
| :--- | :---: | :---: | :---: | :---: | :---: |
| `Novo` | sim | sim | não | não | sim (sem estornar) |
| `EmProducao` | não | não | sim | não | sim (estorna) |
| `Pronto` | não | não | não | sim | sim (estorna) |
| `Entregue` | não | não | não | não | **não** |
| `Cancelado` | não | não | não | não | **não** |

As duas recusas em negrito são as mais interessantes de defender:

**`Entregue` recusa cancelamento.** Depois que o cliente levou os cookies, não existe
"devolver os ingredientes ao estoque": eles foram comidos. Cancelar aqui inflaria o
estoque com farinha que não existe, e todo custo calculado depois disso sairia errado.

**`Cancelado` recusa um segundo cancelamento.** Sem essa recusa, cancelar duas vezes
estornaria os insumos duas vezes, e o sistema passaria a mostrar mais ingrediente do
que existe na prateleira.

Um detalhe de ordem também importa. Em `Novo.confirmar()`, a reserva de insumos
acontece **antes** da troca de estado:

```python
pedido._exigir_itens()
pedido._reservar_insumos()
pedido._definir_estado(EmProducao())
```

Se faltar ingrediente, a reserva levanta exceção e a encomenda continua em "nova" —
nunca fica num estado intermediário inválido.

### Strategy — formas de dar desconto

**Padrão adicional, pesquisado.** Implementado em `doceria/descontos.py`.

Uma confeitaria dá desconto de mais de uma forma, e as regras não se parecem entre si:

| Estratégia | Regra | Depende de |
| :--- | :--- | :--- |
| `SemDesconto` | nada | — |
| `DescontoPercentual(0.10)` | 10% do subtotal | valor |
| `DescontoPorQuantidade(12, 0.15)` | 15% a partir de 12 unidades | **quantidade** |
| `CupomValorFixo(5.00)` | R$ 5,00, limitado ao subtotal | nada |

Se tudo isso virasse um método `calcular_desconto(tipo, ...)` dentro de `Pedido`, a
classe `Pedido` passaria a ter um motivo de mudar que não é dela: toda promoção nova
exigiria editá-la, e a classe que controla o ciclo de vida da encomenda ficaria cheia
de regra de promoção.

Strategy transforma o algoritmo em objeto. O `Pedido` guarda uma estratégia e pergunta
quanto descontar:

```python
pedido.aplicar_estrategia(DescontoPorQuantidade(12, "0.15"))
pedido.desconto()   # o Pedido só pergunta
```

Três decisões de projeto aqui:

**Todas recebem os mesmos dois parâmetros** — subtotal e quantidade total — e cada uma
usa o que precisa. Esse contrato único é o que permite trocar uma pela outra sem o
`Pedido` perceber. `CupomValorFixo` ignora a quantidade por completo, e isso é
deliberado: mostra que o contrato é "me diga quanto descontar", não "faça uma conta com
os dois parâmetros".

**`SemDesconto` existe como objeto.** Ter uma estratégia para o caso "nenhum desconto"
evita que o `Pedido` precise checar `if self._estrategia is None` antes de usar. O caso
de ausência passa a ser tratado como qualquer outro.

**O cupom é limitado ao subtotal.** Um cupom de R$ 20,00 numa encomenda de R$ 12,00 não
pode gerar troco.

O teste `test_estrategia_nova_funciona_sem_alterar_pedido` cria uma estratégia
`FreteGratisAcimaDe50` dentro do próprio teste e a aplica num pedido, sem tocar em
nenhuma classe do pacote.

---

## Funcionalidades implementadas

- **Cadastro de ingrediente por embalagem.** Informa-se o preço pago e o tamanho; o
  custo por unidade é calculado pelo próprio objeto.
- **Validação de unidade na receita.** Informar `kg` num ingrediente medido em `g` é
  recusado, em vez de produzir um custo mil vezes menor em silêncio.
- **Ficha técnica com custo e percentual por ingrediente,** ordenada pelo peso no
  custo. É o relatório que mostra que o chocolate e a manteiga são 74% do custo do
  cookie, e que economizar na farinha quase não muda nada.
- **Custo e margem calculados na hora, nunca copiados.** Se o preço da manteiga mudar,
  a próxima consulta já reflete — não existe campo guardado para ficar desatualizado.
- **Controle de estoque com validação,** sem permitir saldo negativo.
- **Consumo transacional de receita.** Valida todos os ingredientes antes de baixar
  qualquer um: se o terceiro não tiver saldo, nenhum dos três é consumido.
- **Catálogo com dois tipos de produto** e cálculo de margem polimórfico.
- **Quatro estratégias de desconto** intercambiáveis.
- **Encomenda com ciclo de vida completo:** montagem, confirmação com reserva de
  insumos, produção, entrega e cancelamento com estorno.
- **Cálculo de subtotal, desconto, total, custo, lucro e margem** da encomenda, com
  arredondamento a centavo.
- **Custo congelado no item do pedido:** se o insumo subir amanhã, o lucro da encomenda
  de hoje não muda.

---

## Limitações conhecidas

Coisas que o sistema deliberadamente não faz, e por quê.

- **Não persiste nada.** Toda a execução vive em memória. É intencional: o objetivo era
  um domínio que roda e é testado sem infraestrutura. Persistência seria uma camada de
  repositório, fora do escopo do trabalho.
- **Consome ingredientes fracionados.** Uma encomenda de 6 cookies de uma receita que
  rende 15 consome 0,8 ovo. A conta está correta em termos de custo, mas não
  corresponde à cozinha, onde se produz a fornada inteira. O caminho certo seria
  separar estoque de insumo de estoque de produto acabado — o que exigiria um estado
  novo de produção e uma classe de lote.
- **Não trata devolução.** `Entregue` é terminal. Uma devolução real precisaria de um
  estado próprio, com estorno financeiro e **sem** estorno de insumo, já que o doce foi
  consumido. Preferi recusar a operação a implementá-la errado.
- **Não registra histórico de movimentação.** O estoque sabe o saldo atual, não como
  chegou nele. Um kardex (data, tipo, quantidade, saldo anterior e novo, motivo) seria
  o próximo passo natural.
- **Não há concorrência.** Duas encomendas simultâneas do mesmo ingrediente poderiam
  passar da validação de saldo antes de qualquer uma baixar. Resolver isso exigiria
  transação de banco ou travamento, que dependem da camada de persistência.
- **O desconto não valida se o pedido fica no prejuízo.** `no_prejuizo()` apenas
  informa; nada impede aplicar um cupom que zera a margem.

---

## Detalhamento de Aprendizado

> **PREENCHER COM MINHAS PRÓPRIAS PALAVRAS ANTES DE ENTREGAR.**
>
> Material real desta etapa, para eu desenvolver: a diferença entre uma classe que só
> guarda dados e uma que tem comportamento; por que decidi que a classe base de
> `EstadoPedido` deveria recusar tudo por padrão em vez de permitir; o import circular
> entre `pedido.py` e `estados.py` e a solução com `TYPE_CHECKING`; por que a reserva
> de insumos tem que vir antes da troca de estado; por que `Receita.consumir()` valida
> todos os ingredientes antes de baixar o primeiro.

**Dificuldades encontradas:**

**Como resolvi:**

**Principal aprendizado:**

---

## Declaração de Uso de IA

> **CONFERIR E AJUSTAR ANTES DE ENTREGAR.** Esta seção precisa descrever exatamente o
> que aconteceu. O texto abaixo é o registro do que foi feito nesta etapa; qualquer
> trabalho que eu fizer depois disso precisa ser acrescentado aqui.

**Ferramenta e modelo:** Claude (Anthropic), modelos Opus 5.5 e Opus 5, acessados pelo
Claude Code.

**Em quais partes do trabalho a IA foi utilizada:**

- **Definição do escopo.** A IA analisou o sistema de gestão maior que eu mantenho e
  propôs recortar as encomendas como subsistema do trabalho, por ser a parte com ciclo
  de vida natural para o padrão State. A escolha dos três padrões (Factory, State,
  Strategy) e a decisão de fazer um sistema novo em vez de extrair o existente foram
  minhas.
- **Escrita do código do pacote `doceria/`, dos testes e deste README.** Gerados pela
  IA a partir do escopo acordado, com comentários explicando a origem de cada decisão
  de projeto.
- **Identificação das decisões de projeto.** Pontos como "a classe base deve recusar
  por padrão", "validar o consumo antes de baixar qualquer ingrediente" e "`Entregue`
  não pode aceitar cancelamento" foram propostos e justificados pela IA.

**Qual foi o meu aprendizado com o uso:**

> **PREENCHER.** Pontos para eu desenvolver com honestidade: o que eu entendi sobre
> abstração e encapsulamento que eu não entendia antes; por que um sistema pode
> funcionar e ainda ter modelagem ruim; o que eu precisei estudar para conseguir
> defender cada decisão deste código; e onde eu discordei ou mudei o que a IA propôs.
#   T r a b a l h o T O O - e n c o m e n d a s - d o c e s  
 