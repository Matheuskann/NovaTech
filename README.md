# NovaTech
Análise de vendas 2024 de uma empresa fictícia(NovaTech) de eletrônicos. Limpeza e tratamento de 5.000 registros com DuckDB (SQL) e visualização em dashboard no Power BI. Identificação de padrões de faturamento, cancelamentos, produtos e vendedores.

================================================================
       ANÁLISE DE VENDAS 2024 — NovaTech
       Ferramenta: DuckDB (SQL)
================================================================

INTRODUÇÃO
----------------------------------------------------------------
A NovaTech é uma empresa de varejo de produtos eletrônicos que
comercializa itens nas categorias de Informática, Fotografia,
Tablets, Eletrodomésticos, Celulares, Áudio, Wearables e
Acessórios. As vendas são realizadas por meio de diferentes
canais (Marketplace, Loja Física, E-commerce, Aplicativo e
Televendas) e atendidas por uma equipe de vendedores
distribuídos em 9 estados brasileiros.

O objetivo deste estudo é analisar o desempenho de vendas ao
longo de 2024, identificando padrões e oportunidades que
sirvam de base para decisões estratégicas de negócio.

A análise buscou responder às seguintes perguntas:
  - Qual foi o faturamento total de 2024?
  - Qual mês teve o maior e menor faturamento?
  - Qual a taxa de cancelamento e devolução?
  - Quais foram os 5 produtos mais vendidos em quantidade?
  - Quais categorias geraram mais receita?
  - Quais foram os 3 melhores vendedores em faturamento?
  - Algum estado se destacou em vendas?
  - Qual canal de venda trouxe mais receita?
  - Qual forma de pagamento é mais usada?

Os dados utilizados foram extraídos do arquivo
"vendas_2024_NovaTech.csv", contendo 5.000 registros de
pedidos realizados entre 01/01/2024 e 30/12/2024.


================================================================
PARTE 1 — IMPORTAÇÃO E LIMPEZA DOS DADOS
================================================================

----------------------------------------------------------------
PASSO 1: IMPORTAÇÃO DO CSV
----------------------------------------------------------------
Comando utilizado:

    CREATE TABLE vendas AS
    SELECT * FROM read_csv_auto('caminho/vendas_2024_NovaTech.csv');

Por quê:
  O read_csv_auto() detecta automaticamente os tipos de cada
  coluna e importa os dados diretamente para uma tabela DuckDB.
  Isso é mais eficiente do que usar SELECT * FROM 'arquivo.csv'
  diretamente, pois os dados ficam armazenados em memória e
  podem ser reutilizados e modificados sem releitura do arquivo.

Verificação da estrutura importada:

    DESCRIBE vendas;
    SELECT COUNT(*) FROM vendas;

Resultado: 5.000 linhas | 14 colunas


----------------------------------------------------------------
PASSO 2: VERIFICAÇÃO DE NULOS
----------------------------------------------------------------
Comando utilizado:

    SELECT
        COUNT(*) AS total_linhas,
        COUNT(*) - COUNT(id_pedido)       AS nulos_id_pedido,
        COUNT(*) - COUNT(data_venda)      AS nulos_data_venda,
        COUNT(*) - COUNT(produto)         AS nulos_produto,
        COUNT(*) - COUNT(categoria)       AS nulos_categoria,
        COUNT(*) - COUNT(preco_unitario)  AS nulos_preco_unitario,
        COUNT(*) - COUNT(quantidade)      AS nulos_quantidade,
        COUNT(*) - COUNT(desconto)        AS nulos_desconto,
        COUNT(*) - COUNT(total_venda)     AS nulos_total_venda,
        COUNT(*) - COUNT(id_vendedor)     AS nulos_id_vendedor,
        COUNT(*) - COUNT(nome_vendedor)   AS nulos_nome_vendedor,
        COUNT(*) - COUNT(estado_vendedor) AS nulos_estado_vendedor,
        COUNT(*) - COUNT(canal_venda)     AS nulos_canal_venda,
        COUNT(*) - COUNT(status_pedido)   AS nulos_status_pedido,
        COUNT(*) - COUNT(forma_pagamento) AS nulos_forma_pagamento
    FROM vendas;

Por quê:
  COUNT() ignora valores NULL, então COUNT(*) - COUNT(coluna)
  retorna exatamente o número de nulos de cada coluna. Isso
  permite verificar todas as colunas de uma vez sem precisar
  escrever um WHERE IS NULL para cada uma.

Resultado:
  - id_vendedor e nome_vendedor: 164 nulos
  - canal_venda: 46 nulos
  - Demais colunas: 0 nulos

Investigação dos nulos antes de decidir o que fazer:

    SELECT * FROM vendas WHERE id_vendedor IS NULL LIMIT 10;
    SELECT * FROM vendas WHERE canal_venda IS NULL LIMIT 10;

Por quê:
  Antes de remover ou preencher nulos, é essencial investigar
  se são registros válidos ou corrompidos. Neste caso, os
  registros sem vendedor tinham todos os outros campos
  preenchidos, indicando vendas legítimas sem vendedor
  associado. Removê-los cegamente distorceria a análise.

Correção:
  Os nulos foram preenchidos com valores descritivos ao criar
  a tabela limpa (ver Passo 5).


----------------------------------------------------------------
PASSO 3: VERIFICAÇÃO DE TIPOS DE DADOS
----------------------------------------------------------------
Problema identificado via DESCRIBE vendas:
  - data_venda: VARCHAR (deveria ser DATE)
  - desconto: VARCHAR com símbolo "%" (deveria ser DOUBLE)

Verificação do formato das datas:

    SELECT
        CASE
            WHEN data_venda LIKE '__/__/____' THEN 'DD/MM/YYYY'
            WHEN data_venda LIKE '____-__-__' THEN 'YYYY-MM-DD'
            WHEN data_venda LIKE '____/__/__' THEN 'YYYY/MM/DD'
            ELSE 'outro: ' || data_venda
        END AS formato,
        COUNT(*) AS quantidade
    FROM vendas
    GROUP BY formato;

Por quê:
  Antes de converter a coluna para DATE, é preciso saber quais
  formatos existem nos dados. O LIKE com wildcards (__) permite
  identificar o padrão de cada registro e contar quantos há de
  cada tipo. Isso evita erros de conversão.

Resultado: 3 formatos distintos encontrados:
  - DD/MM/YYYY : 4.782 registros
  - YYYY/MM/DD :   110 registros
  - DD-MM-YYYY :   108 registros


----------------------------------------------------------------
PASSO 4: VERIFICAÇÃO DE VALORES INVÁLIDOS
----------------------------------------------------------------
Comandos utilizados:

    -- Duplicatas completas
    SELECT COUNT(*) AS duplicadas
    FROM (
        SELECT *, COUNT(*) AS cnt
        FROM vendas
        GROUP BY ALL
        HAVING cnt > 1
    );

    -- id_pedido duplicado
    SELECT id_pedido, COUNT(*) AS cnt
        FROM vendas
        GROUP BY id_pedido
        HAVING cnt > 1;

    -- Células em branco (diferente de NULL)
    SELECT
        COUNT(*) FILTER (WHERE TRIM(id_pedido) = '')   AS branco_id_pedido,
        COUNT(*) FILTER (WHERE TRIM(produto) = '')     AS branco_produto,
        COUNT(*) FILTER (WHERE TRIM(categoria) = '')   AS branco_categoria
    FROM vendas;

    -- Valores negativos ou zerados
    SELECT
        COUNT(*) FILTER (WHERE preco_unitario <= 0) AS preco_invalido,
        COUNT(*) FILTER (WHERE quantidade <= 0)     AS quantidade_invalida,
        COUNT(*) FILTER (WHERE total_venda <= 0)    AS total_invalido,
        COUNT(*) FILTER (WHERE desconto < 0)        AS desconto_negativo
    FROM vendas;

    -- Intervalo de datas
    SELECT MIN(data_venda) AS data_minima, MAX(data_venda) AS data_maxima
    FROM vendas;

Por quê:
  Cada verificação cobre um tipo diferente de problema:
  - GROUP BY ALL detecta linhas 100% idênticas
  - HAVING cnt > 1 no id_pedido detecta pedidos repetidos
  - TRIM() + '' detecta espaços em branco mascarados como dados
  - FILTER (WHERE ...) permite contar condições específicas em
    uma única passagem pela tabela, sem múltiplos SELECTs
  - MIN/MAX confirma se os dados estão dentro do período esperado

Resultados:
  - Duplicatas: 0
  - Células em branco: 0
  - Preços inválidos: 0
  - Quantidade <= 0: 105 registros
  - Total_venda = 0: 100 registros
  - Datas: 2024-01-01 a 2024-12-30 (dentro do esperado)

Investigação dos valores inválidos:

    SELECT quantidade, total_venda, status_pedido, COUNT(*) as cnt
    FROM vendas
    WHERE quantidade <= 0 OR total_venda <= 0
    GROUP BY quantidade, total_venda, status_pedido
    ORDER BY quantidade;

Resultado:
  - Quantidades negativas (-1 a -5) com total positivo:
    erro de digitação — corrigidos com ABS()
  - Total = 0 com quantidade positiva:
    registros corrompidos — removidos

Verificação de padronização de texto:

    SELECT status_pedido, COUNT(*) FROM vendas GROUP BY status_pedido;
    SELECT forma_pagamento, COUNT(*) FROM vendas GROUP BY forma_pagamento;
    SELECT categoria, COUNT(*) FROM vendas GROUP BY categoria;
    SELECT canal_venda, COUNT(*) FROM vendas GROUP BY canal_venda;
    SELECT produto, COUNT(*) FROM vendas GROUP BY produto;
    SELECT estado_vendedor, COUNT(*) FROM vendas GROUP BY estado_vendedor;

Por quê:
  Agrupar por colunas categóricas e contar os valores distintos
  é a forma mais eficiente de identificar erros de digitação,
  variações de grafia e inconsistências de padronização.

Problemas encontrados:
  - 59 registros com status "completed" (deveria ser "Concluído")
  - 54 registros com forma_pagamento em maiúsculo
    (BOLETO, CARTÃO CRÉDITO, CARTÃO DÉBITO)


----------------------------------------------------------------
PASSO 5: CRIAÇÃO DA TABELA LIMPA
----------------------------------------------------------------
Comando utilizado:

    CREATE TABLE vendas_limpa AS
    SELECT
        id_pedido,
        data_venda,
        produto,
        categoria,
        preco_unitario,
        quantidade,
        CAST(REPLACE(desconto, '%', '') AS DOUBLE) / 100  AS desconto,
        total_venda,
        COALESCE(id_vendedor, 'Desconhecido')             AS id_vendedor,
        COALESCE(nome_vendedor, 'Desconhecido')           AS nome_vendedor,
        estado_vendedor,
        COALESCE(canal_venda, 'Não Informado')            AS canal_venda,
        status_pedido,
        forma_pagamento
    FROM vendas;

Por quê:
  COALESCE() substitui NULL pelo valor padrão definido, apenas
  quando o valor for nulo. REPLACE() remove o símbolo "%" antes
  do CAST para DOUBLE. Dividir por 100 converte o percentual
  para fração decimal (13% → 0.13), facilitando cálculos.

Em seguida, todos os demais problemas foram corrigidos
iterativamente com CREATE OR REPLACE TABLE vendas_limpa:

    -- Correção 1: tipos de data e desconto
    CREATE OR REPLACE TABLE vendas_limpa AS
    SELECT
        id_pedido,
        CASE
            WHEN data_venda LIKE '__/__/____' THEN STRPTIME(data_venda, '%d/%m/%Y')::DATE
            WHEN data_venda LIKE '____/__/__' THEN STRPTIME(data_venda, '%Y/%m/%d')::DATE
            WHEN data_venda LIKE '__-__-____' THEN STRPTIME(data_venda, '%d-%m-%Y')::DATE
        END AS data_venda,
        produto, categoria, preco_unitario,
        ABS(quantidade) AS quantidade,
        desconto, total_venda, id_vendedor, nome_vendedor,
        estado_vendedor, canal_venda,
        CASE
            WHEN status_pedido = 'completed' THEN 'Concluído'
            ELSE status_pedido
        END AS status_pedido,
        CASE
            WHEN forma_pagamento = 'BOLETO'         THEN 'Boleto'
            WHEN forma_pagamento = 'CARTÃO CRÉDITO' THEN 'Cartão Crédito'
            WHEN forma_pagamento = 'CARTÃO DÉBITO'  THEN 'Cartão Débito'
            ELSE forma_pagamento
        END AS forma_pagamento
    FROM vendas_limpa
    WHERE total_venda > 0;

Por quê:
  - STRPTIME() converte string para DATE com formato explícito
  - ::DATE faz o cast final para o tipo DATE do DuckDB
  - O CASE no data_venda detecta o formato pelo separador
    (/ ou -) e aplica o STRPTIME correto para cada registro
  - ABS() converte quantidades negativas para positivo
  - WHERE total_venda > 0 remove os 100 registros corrompidos
  - CASE no status e forma_pagamento padroniza os valores

Resultado final: 4.900 registros limpos e validados.

Verificação final:

    -- Consistência entre colunas
    SELECT COUNT(*) AS inconsistentes
    FROM vendas_limpa
    WHERE ABS(total_venda - (preco_unitario * quantidade * (1 - desconto))) > 1;

    -- Formatos de ID
    SELECT COUNT(*) FROM vendas_limpa WHERE id_pedido NOT LIKE 'PED-%';
    SELECT COUNT(*) FROM vendas_limpa
        WHERE id_vendedor NOT LIKE 'V%' AND id_vendedor != 'Desconhecido';

Resultado: 0 inconsistências em todas as verificações finais.


================================================================
PARTE 2 — ANÁLISES
================================================================

----------------------------------------------------------------
ANÁLISE 1: FATURAMENTO TOTAL DE 2024
----------------------------------------------------------------
Comando utilizado:

    SELECT ROUND(SUM(total_venda), 2) AS faturamento_total
    FROM vendas_limpa
    WHERE status_pedido = 'Concluído';

Por quê:
  Filtramos apenas pedidos Concluídos pois Cancelados e
  Devolvidos não representam receita efetiva para a empresa.
  ROUND(..., 2) garante o resultado com 2 casas decimais.

Resultado: R$ 10.645.968,50


----------------------------------------------------------------
ANÁLISE 2: FATURAMENTO POR MÊS
----------------------------------------------------------------
Comando utilizado:

    SELECT
        MONTH(data_venda) AS mes_numero,
        CASE MONTH(data_venda)
            WHEN 1  THEN 'Janeiro'   WHEN 2  THEN 'Fevereiro'
            WHEN 3  THEN 'Março'     WHEN 4  THEN 'Abril'
            WHEN 5  THEN 'Maio'      WHEN 6  THEN 'Junho'
            WHEN 7  THEN 'Julho'     WHEN 8  THEN 'Agosto'
            WHEN 9  THEN 'Setembro'  WHEN 10 THEN 'Outubro'
            WHEN 11 THEN 'Novembro'  WHEN 12 THEN 'Dezembro'
        END AS mes_nome,
        COUNT(*) AS qtd_pedidos,
        ROUND(SUM(total_venda), 2) AS faturamento
    FROM vendas_limpa
    WHERE status_pedido = 'Concluído'
    GROUP BY mes_numero
    ORDER BY mes_numero;

Por quê:
  MONTH() extrai o número do mês da coluna DATE. O CASE traduz
  o número para o nome em português, pois STRFTIME('%B') retorna
  nomes em inglês no DuckDB. GROUP BY mes_numero agrupa os
  registros por mês para somar o faturamento de cada período.

Resultado:
  Maior: Julho — R$ 1.019.507,84
  Menor: Setembro — R$ 721.345,49


----------------------------------------------------------------
ANÁLISE 3: TAXA DE CANCELAMENTO E DEVOLUÇÃO
----------------------------------------------------------------
Comando utilizado:

    SELECT
        status_pedido,
        COUNT(*) AS qtd_pedidos,
        ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER(), 2) AS percentual
    FROM vendas_limpa
    GROUP BY status_pedido
    ORDER BY qtd_pedidos DESC;

Por quê:
  SUM(COUNT(*)) OVER() é uma window function que calcula o
  total geral de pedidos sem agrupar, permitindo dividir cada
  contagem pelo total e obter o percentual de cada status em
  uma única query, sem subconsultas.

Resultado:
  Concluído : 3.452 — 70,45%
  Cancelado :   761 — 15,53%
  Devolvido :   687 — 14,02%
  Taxa de perda total: 29,55%


----------------------------------------------------------------
ANÁLISE 4: TOP 5 PRODUTOS MAIS VENDIDOS
----------------------------------------------------------------
Comando utilizado:

    SELECT
        produto,
        SUM(quantidade)            AS total_quantidade,
        COUNT(*)                   AS qtd_pedidos,
        ROUND(SUM(total_venda), 2) AS faturamento
    FROM vendas_limpa
    WHERE status_pedido = 'Concluído'
    GROUP BY produto
    ORDER BY total_quantidade DESC
    LIMIT 5;

Por quê:
  SUM(quantidade) soma todas as unidades vendidas por produto,
  diferente de COUNT(*) que contaria apenas o número de pedidos.
  Um pedido pode ter 5 unidades, então somar a quantidade é
  mais preciso para medir volume real de vendas.

Resultado:
  1º Caixa de Som JBL Flip 6   — 761 unidades — R$   420.667,87
  2º Notebook Dell Inspiron    — 754 unidades — R$ 2.029.930,01
  3º Smartwatch Xiaomi Band 8  — 753 unidades — R$   209.588,06
  4º Webcam Logitech C920      — 724 unidades — R$   301.504,92
  5º Mouse Logitech MX Master  — 716 unidades — R$   231.329,30


----------------------------------------------------------------
ANÁLISE 5: FATURAMENTO POR CATEGORIA
----------------------------------------------------------------
Comando utilizado:

    SELECT
        categoria,
        COUNT(*) AS qtd_pedidos,
        SUM(quantidade) AS total_itens,
        ROUND(SUM(total_venda), 2) AS faturamento,
        ROUND(SUM(total_venda) * 100.0 / SUM(SUM(total_venda)) OVER(), 2) AS percentual
    FROM vendas_limpa
    WHERE status_pedido = 'Concluído'
    GROUP BY categoria
    ORDER BY faturamento DESC;

Por quê:
  SUM(SUM(total_venda)) OVER() é uma window function aninhada
  que soma todos os faturamentos agrupados, retornando o total
  geral. Isso permite calcular o percentual de cada categoria
  sobre o total sem precisar de uma subquery separada.

Resultado:
  1º Informática      — R$ 3.420.048,53 — 32,13%
  2º Fotografia       — R$ 2.110.028,41 — 19,82%
  3º Tablets          — R$ 1.652.462,75 — 15,52%
  4º Eletrodomésticos — R$ 1.386.117,66 — 13,02%
  5º Celulares        — R$   990.794,03 —  9,31%
  6º Áudio            — R$   779.192,79 —  7,32%
  7º Wearables        — R$   209.588,06 —  1,97%
  8º Acessórios       — R$    97.736,27 —  0,92%


----------------------------------------------------------------
ANÁLISE 6: TOP 3 VENDEDORES EM FATURAMENTO
----------------------------------------------------------------
Comando utilizado:

    SELECT
        id_vendedor,
        nome_vendedor,
        COUNT(*) AS qtd_pedidos,
        SUM(quantidade) AS total_itens,
        ROUND(SUM(total_venda), 2) AS faturamento
    FROM vendas_limpa
    WHERE status_pedido = 'Concluído'
    AND id_vendedor != 'Desconhecido'
    GROUP BY id_vendedor, nome_vendedor
    ORDER BY faturamento DESC
    LIMIT 3;

Por quê:
  Excluímos os registros com id_vendedor = 'Desconhecido' pois
  não representam um vendedor real e distorceriam o ranking.
  GROUP BY nos dois campos garante que cada vendedor apareça
  uma única vez com seu nome correto.

Resultado:
  1º Amanda Costa    (V004) — 373 pedidos — R$ 1.168.481,24
  2º Camila Ferreira (V010) — 342 pedidos — R$ 1.115.028,12
  3º Roberto Silva   (V003) — 339 pedidos — R$ 1.082.260,06


----------------------------------------------------------------
ANÁLISE 7: FATURAMENTO POR ESTADO
----------------------------------------------------------------
Comando utilizado:

    SELECT
        estado_vendedor,
        COUNT(*) AS qtd_pedidos,
        SUM(quantidade) AS total_itens,
        ROUND(SUM(total_venda), 2) AS faturamento,
        ROUND(SUM(total_venda) * 100.0 / SUM(SUM(total_venda)) OVER(), 2) AS percentual
    FROM vendas_limpa
    WHERE status_pedido = 'Concluído'
    GROUP BY estado_vendedor
    ORDER BY faturamento DESC;

Resultado:
  1º SP — R$ 2.025.798,00 — 19,03%
  2º RS — R$ 1.203.845,65 — 11,31%
  3º GO — R$ 1.169.613,35 — 10,99%
  ...
  9º RJ — R$   966.192,29 —  9,08%


----------------------------------------------------------------
ANÁLISE 8: FORMA DE PAGAMENTO MAIS USADA
----------------------------------------------------------------
Comando utilizado:

    SELECT
        forma_pagamento,
        COUNT(*) AS qtd_pedidos,
        ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER(), 2) AS percentual,
        ROUND(SUM(total_venda), 2) AS faturamento
    FROM vendas_limpa
    WHERE status_pedido = 'Concluído'
    GROUP BY forma_pagamento
    ORDER BY qtd_pedidos DESC;

Resultado:
  1º PIX            — 1.169 pedidos — 33,86% — R$ 3.690.796,00
  2º Cartão Crédito — 1.119 pedidos — 32,42% — R$ 3.489.663,84
  3º Cartão Débito  —   585 pedidos — 16,95% — R$ 1.662.067,52
  4º Boleto         —   579 pedidos — 16,77% — R$ 1.803.441,14


================================================================
CONCLUSÃO
================================================================
A análise dos dados de 2024 revelou um faturamento sólido de
R$ 10,6 milhões, com Julho como mês de pico (R$ 1.019.507,84)
e Setembro como ponto de atenção (R$ 721.345,49).

A taxa de perda de 29,55% (cancelamentos + devoluções) é o
principal indicador de melhoria identificado — quase 1 em cada
3 pedidos não se converte em receita efetiva.

Informática domina o portfólio com 32% da receita. SP concentra
19% das vendas. PIX já é o principal meio de pagamento. Os 3
melhores vendedores respondem por 31,6% do faturamento total.


RECOMENDAÇÕES
----------------------------------------------------------------
1. Investigar causas da alta taxa de cancelamento e devolução,
   especialmente por produto e canal de venda.

2. Entender a queda de Setembro — sazonalidade, problemas
   operacionais ou ausência de campanhas promocionais.

3. Desenvolver estratégias para Wearables e Acessórios,
   categorias com menos de 3% de participação na receita.

4. Equilibrar o time de vendas para reduzir a concentração
   de receita nos top 3 vendedores.

5. Explorar o potencial do RJ, último no ranking de estados
   apesar de ser um grande centro econômico.
