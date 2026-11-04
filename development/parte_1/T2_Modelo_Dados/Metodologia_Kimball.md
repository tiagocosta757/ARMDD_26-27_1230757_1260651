# Modelo Dimensional

O modelo segue a metodologia de Kimball, estruturada em quatro passos: identificar o processo de negócio, definir a granularidade, identificar as dimensões e identificar os factos.

## 1. Processo de negócio

Vendas e faturação a clientes.

O enunciado pede um data mart para "análises históricas às vendas efetuadas a clientes". Entram as faturas e respetivas linhas, com os produtos vendidos, clientes, empregados e custos de expedição.

Compras a fornecedores e gestão de stocks ficam de fora. São processos de negócio distintos, que dariam origem a tabelas de factos próprias. Aparecem apenas como contexto das vendas. O fornecedor, por exemplo, é pedido como eixo de análise sem deixar de ser um atributo do produto vendido.

## 2. Granularidade

Cada linha da tabela de factos representa uma linha de fatura: um produto, numa fatura, a um cliente, numa data.

O enunciado exige o "nível de granularidade mais elementar", e é este o nível mais fino que existe nos dados de venda.

Três das dez análises indicativas só funcionam com este detalhe:

- As análises 5 e 6 exigem drill down até ao nível do produto.
- A análise 7 pede quantidades por cliente e por produto.
- A análise 2 pede os custos de expedição por família de produto.
  Agregar ao nível da fatura obrigaria a perder o produto, a quantidade e o valor por produto, o que deixaria metade das análises sem resposta. Mas não se perde a perspetiva da fatura. O identificador da fatura é guardado na tabela de factos como dimensão degenerada, o que permite reagrupar por fatura e reconstituir os totais sempre que necessário.

Também não faz sentido fixar o grão ao dia. A granularidade define o que cada registo representa, não o nível a que as análises vão agregar. A data entra no modelo como dimensão, e cada linha transporta a data da sua fatura, o que já permite agregar por dia, semana, mês, trimestre, semestre ou ano.

Um grão mais fino custa espaço mas nunca retira flexibilidade, porque permite sempre agregar para cima. Um grão mais grosso é irreversível.

### Custos de expedição

O enunciado avisa que os custos de expedição estão ao nível da fatura e não de cada linha. A tabela `invoices` tem o campo `shipping_cost`, enquanto `invoice_lines` não tem qualquer campo equivalente. Temos portanto uma medida cuja granularidade natural é mais grosseira do que o grão escolhido, situação que obriga a um procedimento de alocação.

À primeira vista isto parece um argumento a favor do grão ao nível da fatura, mas é o contrário. A análise 2 pede o frete detalhado por família de produto. Numa fatura com produtos de três famílias diferentes esse valor não existe nos dados de origem. Só se obtém repartindo o frete da fatura pelas suas linhas. Se é preciso alocar o frete até à linha, então o grão tem de ser a linha.

O critério adotado é a repartição proporcional ao valor da linha:

```
frete_linha = shipping_cost × (line_total / products_total)
```

A observação dos dados mostra que o `shipping_cost` corresponde a 2,5 por cento do `products_total` da fatura. A repartição proporcional ao valor é assim a única que reproduz a lógica com que o valor foi calculado na origem, e distribui o custo de forma coerente entre linhas de valores muito diferentes. A repartição proporcional ao peso ou ao volume transportado não é viável, por não existirem esses atributos nos produtos.

A soma das parcelas alocadas tem de fechar exatamente com o `shipping_cost` da fatura. Com arredondamento a duas casas decimais sobra quase sempre um resto de cêntimos, que deve ser absorvido na última linha da fatura.

O frete é alocado sem IVA. Nas faturas observadas, o `vat_value` corresponde à soma do `line_vat` das linhas e não inclui IVA sobre o frete, apesar de o `shipping_cost` entrar no campo `total_for_vat`.

## 3. Dimensões

| Dimensão    | Responde a                      | Tabela de origem               |
| ------------ | ------------------------------- | ------------------------------ |
| Dim_Date     | Quando ocorreu a venda          | Gerada no ETL                  |
| Dim_Product  | O que foi vendido               | products, families, categories |
| Dim_Customer | A quem foi vendido              | customers, zone                |
| Dim_Supplier | Quem forneceu o produto vendido | suppliers, zone                |
| Dim_Employee | Quem efetuou a venda            | employees                      |
| Dim_Currency | Em que moeda foi faturado       | invoices                       |

Todas as dimensões têm chave primária própria, uma chave surrogate inteira sem significado de negócio, atribuída sequencialmente. A chave natural do sistema operacional é guardada como atributo, o que permite a rastreabilidade à fonte e protege o data mart de alterações no sistema operacional.

Os atributos `row_effective_date`, `row_expiration_date` e `is_current` suportam a estratégia de Slowly Changing Dimension. A estratégia concreta a aplicar a cada atributo é definida no mapeamento entre a staging area e o armazém de dados.

### Dim_Date

| Atributo          | Tipo        | Descrição                          |
| ----------------- | ----------- | ------------------------------------ |
| date_key          | INT         | Chave surrogate, no formato AAAAMMDD |
| full_date         | DATE        | Data completa                        |
| day_number        | TINYINT     | Dia do mês                          |
| day_of_week       | VARCHAR(10) | Nome do dia da semana                |
| day_number_week   | TINYINT     | Número do dia na semana             |
| is_weekend        | BIT         | Indica se é sábado ou domingo      |
| week_number       | TINYINT     | Número da semana no ano             |
| month_number      | TINYINT     | Número do mês                      |
| month_name        | VARCHAR(10) | Nome do mês                         |
| quarter           | TINYINT     | Trimestre                            |
| four_month_period | TINYINT     | Quadrimestre                         |
| semester          | TINYINT     | Semestre                             |
| year              | SMALLINT    | Ano                                  |
| season            | VARCHAR(10) | Estação do ano                     |
| is_last_day_month | BIT         | Indica se é o último dia do mês   |

Esta dimensão não tem origem no sistema operacional. É construída por geração no processo de ETL, a partir da coluna `invoices.invoice_date`.

É a única dimensão em que a chave surrogate tem relação com a semântica dos dados, por ser atribuída sequencialmente pela ordem cronológica. É também a única que pode ser construída antecipadamente, sem depender do carregamento dos dados de origem.

Os atributos derivam das análises pedidas. O quadrimestre vem da análise 7, a semana da análise 6, a estação do ano da análise 8, o último dia do mês da análise 5 e o fim de semana da análise 10. O trimestre e o semestre vêm das análises 1, 3 e 9.

A hora não é modelada. Nenhuma das análises desce abaixo do dia, pelo que não existe necessidade de roll up ou drill down sobre agrupamentos horários. Separar a hora como dimensão própria ou juntá-la à data faria crescer a dimensão temporal em várias ordens de grandeza sem benefício analítico. Os campos `create_time` e `last_update_time` registados na fonte são usados no processo de ETL para o carregamento incremental e para a deteção de alterações, não como dimensão.

### Dim_Product

| Atributo            | Tipo          | Origem                      | Descrição                    |
| ------------------- | ------------- | --------------------------- | ------------------------------ |
| product_key         | INT           | Gerado                      | Chave surrogate                |
| product_reference   | VARCHAR(20)   | products.reference          | Chave natural                  |
| product_designation | VARCHAR(150)  | products.designation        | Designação do produto        |
| family_reference    | VARCHAR(20)   | products.family             | Código da família            |
| family_name         | VARCHAR(100)  | families.name               | Designação da família       |
| category_code       | INT           | products.category           | Código da categoria           |
| category_name       | VARCHAR(50)   | categories.Category         | Designação da categoria      |
| vat_rate            | DECIMAL(5,2)  | products.vat_rate           | Taxa de IVA aplicável         |
| current_sell_price  | DECIMAL(12,2) | products.current_sell_price | Preço de venda em vigor       |
| row_effective_date  | DATE          | Gerado                      | Início de validade do registo |
| row_expiration_date | DATE          | Gerado                      | Fim de validade do registo     |
| is_current          | BIT           | Gerado                      | Indica o registo em vigor      |

A família e a categoria não são dimensões próprias. Na fonte são tabelas separadas, ambas ligadas a produtos, mas no modelo dimensional ficam desnormalizadas dentro desta dimensão. É a opção que Kimball recomenda, guardando a informação hierárquica de forma redundante para favorecer a facilidade de uso e o desempenho das consultas. Resulta num esquema em estrela e não em floco de neve.

Confirma-se pelos dados que família e categoria são dois eixos de classificação independentes e não uma hierarquia entre si. O produto `PN6354` pertence à família `Partnum` e à categoria `Peças`, sem relação entre ambas.

As análises 2, 6 e 9 pedem agregação por família ou categoria com drill down até ao produto. Com estes atributos na mesma dimensão, as hierarquias produto para família e produto para categoria ficam disponíveis sem qualquer junção adicional.

Os campos `stock` e `minimum_stock` não são incluídos. São medidas do processo de gestão de stocks, alheio ao processo de negócio modelado, e variam continuamente, o que provocaria um crescimento desnecessário da dimensão em caso de SCD de tipo 2.

### Dim_Customer

| Atributo            | Tipo         | Origem                      | Descrição                              |
| ------------------- | ------------ | --------------------------- | ---------------------------------------- |
| customer_key        | INT          | Gerado                      | Chave surrogate                          |
| customer_number     | INT          | customers.number            | Chave natural                            |
| customer_name       | VARCHAR(100) | customers.name              | Nome do cliente                          |
| customer_type       | VARCHAR(30)  | customers.type              | Tipo de cliente                          |
| customer_segment    | VARCHAR(20)  | customers.segment           | Segmento do cliente                      |
| taxpayer_number     | VARCHAR(20)  | customers.taxpayer_number   | Número de contribuinte                  |
| address             | VARCHAR(150) | customers.address           | Morada                                   |
| locality            | VARCHAR(50)  | customers.clity             | Localidade                               |
| postal_code         | VARCHAR(10)  | customers.zip_code          | Código postal                           |
| city                | VARCHAR(50)  | customers.zip_code          | Cidade, extraída da designação postal |
| zone_code           | INT          | customers.zone              | Código da zona                          |
| zone_name           | VARCHAR(50)  | zone.zone                   | Designação da zona                     |
| assigned_seller     | VARCHAR(100) | customers.seller            | Vendedor atribuído ao cliente           |
| payment_days        | INT          | customers.days_pay_invoices | Prazo de pagamento acordado              |
| row_effective_date  | DATE         | Gerado                      | Início de validade do registo           |
| row_expiration_date | DATE         | Gerado                      | Fim de validade do registo               |
| is_current          | BIT          | Gerado                      | Indica o registo em vigor                |

A hierarquia geográfica é cliente para localidade para cidade para zona.

A separação entre localidade e cidade resolve uma ambiguidade da fonte. O campo `clity` ora traz a cidade ora traz uma localidade de menor dimensão, como se vê na comparação entre "A-dos-Bispos" e "Lisboa". A designação postal presente no `zip_code` traz de forma consistente o concelho, como em "2600-011 VILA FRANCA DE XIRA". A cidade é por isso obtida por extração da parte textual do código postal, ficando o `clity` como atributo de nível inferior.

Esta decisão é necessária porque as análises 2 e 8 pedem detalhe por cidade do cliente, enquanto as análises 1, 9 e 10 pedem agregação ao nível da zona. Ambos os níveis são indispensáveis.

A análise 7 pede roll up ao nível do tipo de cliente, satisfeito pelo atributo `customer_type`.

O atributo `assigned_seller` identifica o vendedor atribuído ao cliente e não deve ser confundido com o empregado que emitiu a fatura, guardado em Dim_Employee. São conceitos distintos e correspondem a campos diferentes na fonte.

Esta é a dimensão afetada pelo problema de qualidade que o enunciado descreve, em que o mesmo cliente pode estar registado em duplicado com identificadores diferentes, de forma rigorosamente igual ou aproximada. O tratamento desses casos é definido no mapeamento entre a staging area e o armazém de dados.

Os campos de contacto telefónico, correio eletrónico e identificação bancária não são incluídos, por não constituírem eixos de análise.

### Dim_Supplier

| Atributo            | Tipo         | Origem                      | Descrição                    |
| ------------------- | ------------ | --------------------------- | ------------------------------ |
| supplier_key        | INT          | Gerado                      | Chave surrogate                |
| supplier_number     | INT          | suppliers.number            | Chave natural                  |
| supplier_name       | VARCHAR(100) | suppliers.name              | Nome do fornecedor             |
| supplier_type       | VARCHAR(30)  | suppliers.type              | Tipo de fornecedor             |
| taxpayer_number     | VARCHAR(20)  | suppliers.taxpayer_number   | Número de contribuinte        |
| address             | VARCHAR(150) | suppliers.address           | Morada                         |
| city                | VARCHAR(50)  | suppliers.city              | Cidade                         |
| postal_code         | VARCHAR(10)  | suppliers.zip_code          | Código postal                 |
| zone_code           | INT          | suppliers.zone              | Código da zona                |
| zone_name           | VARCHAR(50)  | zone.zone                   | Designação da zona           |
| nationality         | VARCHAR(50)  | suppliers.nationality       | Nacionalidade                  |
| payment_days        | INT          | suppliers.days_pay_invoices | Prazo de pagamento acordado    |
| row_effective_date  | DATE         | Gerado                      | Início de validade do registo |
| row_expiration_date | DATE         | Gerado                      | Fim de validade do registo     |
| is_current          | BIT          | Gerado                      | Indica o registo em vigor      |

O fornecedor liga-se ao produto de forma direta, através do campo `products.supplyer`. Cada produto tem um único fornecedor, pelo que a associação entre uma venda e o respetivo fornecedor é determinística e não exige qualquer critério de desambiguação.

Mantém-se como dimensão própria em vez de ser desnormalizada dentro de Dim_Product. Tem um número considerável de atributos, que engrossariam a dimensão de produto sem necessidade, e as análises 3 e 10 tratam-no como eixo de análise de pleno direito, incluindo com hierarquia geográfica própria e drill down até ao fornecedor individual.

O fornecedor e o cliente ficam em dimensões separadas, apesar de partilharem vários atributos. São três as razões.

Os atributos não coincidem. O tipo e o segmento de cliente não têm equivalente no fornecedor, e a nacionalidade não tem equivalente no cliente. Numa dimensão única, estes campos ficariam sistematicamente por preencher em metade dos registos.

Os papéis não são simétricos face ao facto. O cliente liga-se diretamente à venda, porque a fatura tem um cliente. O fornecedor chega à venda através do produto, enquanto atributo descritivo daquilo que foi vendido. Juntá-los sugeriria uma simetria que não existe.

A análise 10 pede, na mesma análise, detalhe por zona do fornecedor e por zona do cliente. Com uma dimensão única, a tabela de factos teria duas chaves estrangeiras a apontar para a mesma tabela, obrigando a referenciá-la duas vezes com nomes distintos em cada consulta.

### Dim_Employee

| Atributo            | Tipo         | Origem                | Descrição                       |
| ------------------- | ------------ | --------------------- | --------------------------------- |
| employee_key        | INT          | Gerado                | Chave surrogate                   |
| employee_number     | INT          | employees.number      | Chave natural                     |
| employee_code       | VARCHAR(20)  | employees.code        | Código do empregado              |
| employee_name       | VARCHAR(100) | employees.name        | Nome do empregado                 |
| employee_group      | VARCHAR(50)  | employees.group       | Função                          |
| department          | VARCHAR(50)  | employees.departament | Departamento                      |
| chief_name          | VARCHAR(100) | employees.chief       | Nome do responsável hierárquico |
| row_effective_date  | DATE         | Gerado                | Início de validade do registo    |
| row_expiration_date | DATE         | Gerado                | Fim de validade do registo        |
| is_current          | BIT          | Gerado                | Indica o registo em vigor         |

A tabela `invoices` tem uma ligação direta a `employees`, que identifica quem efetuou cada venda. A dimensão responde à pergunta sobre quem vendeu, complementar à do cliente, que responde a quem se vendeu.

As análises 3 e 4 pedem vendas detalhadas por empregado.

O campo `chief` da fonte é uma auto-relação ao próprio número de empregado. É resolvido no ETL para o nome do responsável, o que desnormaliza a hierarquia de chefia dentro da dimensão e permite analisar vendas por responsável de equipa além do vendedor individual.

O empregado está registado ao nível da fatura e não da linha, mas isso não exige qualquer alocação. Sendo um atributo descritivo e não uma medida, todas as linhas de uma fatura herdam o mesmo empregado. Só as medidas de granularidade mais grosseira, como o frete, obrigam a repartição.

Esta dimensão não dispõe de zona na fonte, ao contrário de clientes e fornecedores. A análise geográfica de vendas por empregado, caso venha a ser necessária, terá de ser feita através da zona do cliente.

### Dim_Currency

| Atributo        | Tipo        | Origem            | Descrição           |
| --------------- | ----------- | ----------------- | --------------------- |
| currency_key    | INT         | Gerado            | Chave surrogate       |
| currency_code   | CHAR(3)     | invoices.currency | Código ISO da moeda  |
| currency_name   | VARCHAR(50) | Gerado            | Designação da moeda |
| currency_symbol | VARCHAR(5)  | Gerado            | Símbolo              |

O enunciado determina que o data mart tem de contemplar a expansão a outros países da União Europeia e a abertura de uma sucursal no Reino Unido, considerando o Euro como moeda de uniformização para análises que envolvam moedas diferentes.

A dimensão permite cumprir esse requisito sem perder informação. A tabela de factos guarda os valores na moeda original e os mesmos valores convertidos a Euro, mais a taxa de câmbio aplicada. A conversão garante que as medidas são aditivas e comparáveis entre países, e a manutenção dos valores originais permite responder a questões sobre a faturação em cada moeda.

A taxa de câmbio é guardada na tabela de factos e não na dimensão, por variar ao longo do tempo e ser específica de cada transação. A dimensão guarda apenas os atributos descritivos da moeda, que são estáveis.

O campo `invoices.currency` existe na fonte, mas contém o valor 'EURO', que não corresponde ao código ISO. A normalização para 'EUR' é feita na fase de transformação. No momento atual a empresa opera em Portugal e pontualmente em Espanha, ambos na zona Euro, pelo que a dimensão terá um único registo nos dados disponibilizados. O modelo fica no entanto preparado para acomodar novas moedas sem alterações estruturais.

A conversão é efetuada na fase de transformação, dentro da staging area, em conjunto com as restantes operações de transformação e limpeza.

## 4. Factos

### Fact_Sales

| Atributo              | Tipo          | Origem                              | Descrição                         |
| --------------------- | ------------- | ----------------------------------- | ----------------------------------- |
| date_key              | INT           | invoices.invoice_date               | Chave estrangeira para Dim_Date     |
| product_key           | INT           | invoice_lines.product               | Chave estrangeira para Dim_Product  |
| customer_key          | INT           | invoices.customer                   | Chave estrangeira para Dim_Customer |
| supplier_key          | INT           | products.supplyer                   | Chave estrangeira para Dim_Supplier |
| employee_key          | INT           | invoices.employee                   | Chave estrangeira para Dim_Employee |
| currency_key          | INT           | invoices.currency                   | Chave estrangeira para Dim_Currency |
| invoice_stamp_id      | VARCHAR(50)   | invoices.invoice_stamp_id           | Dimensão degenerada                |
| invoice_line_stamp_id | VARCHAR(50)   | invoice_lines.invoice_line_stamp_id | Dimensão degenerada                |
| quantity              | INT           | invoice_lines.quantity              | Quantidade vendida                  |
| unit_price            | DECIMAL(12,2) | invoice_lines.unit_price            | Preço unitário                    |
| value_without_vat     | DECIMAL(12,2) | invoice_lines.line_total            | Valor da linha sem IVA              |
| vat_amount            | DECIMAL(12,2) | invoice_lines.line_vat              | Valor do IVA da linha               |
| value_with_vat        | DECIMAL(12,2) | Calculado                           | Valor da linha com IVA              |
| allocated_freight     | DECIMAL(12,2) | Calculado                           | Frete alocado à linha              |
| exchange_rate         | DECIMAL(12,6) | Gerado                              | Taxa de câmbio aplicada            |
| value_without_vat_eur | DECIMAL(12,2) | Calculado                           | Valor sem IVA em Euro               |
| vat_amount_eur        | DECIMAL(12,2) | Calculado                           | Valor do IVA em Euro                |
| value_with_vat_eur    | DECIMAL(12,2) | Calculado                           | Valor com IVA em Euro               |
| allocated_freight_eur | DECIMAL(12,2) | Calculado                           | Frete alocado em Euro               |

A chave primária é composta pelo conjunto das chaves estrangeiras, acrescido do `invoice_line_stamp_id`, que garante a unicidade de cada registo.

Existe uma única tabela de factos porque existe um único processo de negócio. Todas as medidas se situam ao mesmo grão, a linha de fatura, conforme a regra de que uma tabela de factos não pode conter medidas de granularidades diferentes.

Os valores sem IVA, do IVA e com IVA são guardados em separado em vez de se guardar apenas um deles. A análise 4 pede valores sem IVA e valores de IVA, a análise 1 pede valores com IVA e a análise 8 pede apenas o IVA. Guardando os três componentes, todas as análises se resolvem por simples soma.

Pela mesma razão o frete é guardado isolado das restantes medidas. A análise 9 pede valores de vendas sem IVA e sem custos de expedição, o que só é possível se o frete constituir uma medida própria.

Todas as medidas são aditivas ao longo de todas as dimensões, com exceção da taxa de câmbio, que é um valor de referência e não uma medida a somar. Indicadores como a margem não são guardados por serem não aditivos, devendo ser calculados no momento da consulta a partir das medidas aditivas.

O `invoice_stamp_id` e o `invoice_line_stamp_id` são dimensões degeneradas, chaves sem tabela de dimensão associada. Guardam a referência direta ao sistema operacional e permitem reagrupar os registos ao nível da fatura, recuperando designadamente o valor total do frete.

A fonte não dispõe de numeração sequencial das linhas dentro de cada fatura. Caso se revele necessária para apresentação, pode ser gerada no processo de ETL pela ordenação do `invoice_line_stamp_id` dentro de cada fatura.

## Dimensões consideradas e não criadas

### Zona

A zona existe na fonte como tabela própria, com cinco registos, e está associada a clientes, fornecedores e compras. Optou-se por desnormalizar os seus atributos dentro das dimensões de cliente e de fornecedor, em vez de criar uma dimensão própria.

Uma dimensão de zona obrigaria a tabela de factos a conter duas chaves estrangeiras para a mesma tabela, com a consequente necessidade de a referenciar duas vezes em consultas como a da análise 10. A desnormalização resolve as hierarquias geográficas dentro de cada dimensão e torna o drill down imediato nas ferramentas de análise.

A tabela `zone` tem apenas duas colunas, o código e a designação, pelo que nada se perde em capacidade descritiva. A redundância resultante é intencional. As dimensões representam uma fração reduzida do espaço ocupado pela base de dados dimensional, pelo que o custo de armazenamento é irrelevante face ao ganho em simplicidade e desempenho.

### Família e categoria de produto

Desnormalizadas em Dim_Product, pelas razões indicadas na descrição dessa dimensão.

### Hora

Não modelada, pelas razões indicadas na descrição de Dim_Date.

### Data de pagamento

A tabela `invoices` dispõe do campo `payment_date`, que permitiria uma segunda ligação a Dim_Date no papel de data de pagamento. Nenhuma das análises pedidas o exige, pelo que não é incluído. Caso venha a ser necessário, a Dim_Date suporta essa utilização sem alterações, na qualidade de role playing dimension.

## Questões a colocar ao docente

**Taxa de câmbio.** O enunciado determina que o data mart considere o Euro como moeda de uniformização para análises que envolvam moedas diferentes, mas o sistema operacional não dispõe de qualquer campo com taxas de câmbio. O campo `invoices.currency` contém apenas a designação da moeda. Importa esclarecer se a taxa de câmbio deve ser assumida como unitária enquanto a totalidade da faturação for em Euro, se deve ser obtida de uma fonte externa a definir, ou se basta que o modelo demonstre estar preparado para a acomodar.

## Verificações pendentes sobre os dados

**Cálculo do IVA ao nível da fatura.** Na primeira fatura observada, o `vat_value` corresponde a 23 por cento do `products_total` e não do `total_for_vat`, o que confirma que o frete não é tributado. Na segunda fatura, porém, o `vat_value` de 32,20 não corresponde a 23 por cento do `products_total` de 145. É necessário verificar se existem produtos com taxas de IVA distintas ou isentos.

**Escala dos valores em products.** Os campos `current_sell_price` e `last_cost_price` apresentam valores da ordem de 1000000 e 500000 para um fusível, enquanto o `unit_price` praticado nas linhas de fatura é da ordem das centenas. É necessário verificar se existe fator de escala ou casas decimais implícitas.

**Codificação de carateres.** Os dados apresentam corrupção sistemática de carateres acentuados, com ocorrências como "Alpiarзa", "Mйdio", "Peзas" e "nє9". O problema afeta nomes de cidades, designações de categorias e segmentos de cliente, que são atributos usados para agrupamento nas análises. Exige tratamento na fase de limpeza.

**Datas de alteração anteriores às de criação.** Nos registos de fornecedores observados, o `create_date` é de 2018 e o `last_update_date` de 2017. Trata-se de uma violação de integridade em colunas que suportam o carregamento incremental.

**Cliente genérico de vendas a dinheiro.** O cliente número 2, designado "Vendas a Dinheiro", apresenta contribuinte a zeros e a generalidade dos campos por preencher. Não é um registo defeituoso mas um cliente genérico do negócio, pelo que deve ser tratado como membro especial da dimensão e não rejeitado na fase de limpeza.

**Duplicação entre classificação e categoria do produto.** O campo `products.classification` apresenta o mesmo valor da designação da categoria correspondente. É necessário confirmar se é sempre assim, caso em que se trata de redundância na fonte e o campo pode ser dispensado.
