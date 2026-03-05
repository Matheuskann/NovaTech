# Análise de Vendas 2024 — NovaTech


A NovaTech é uma empresa fictícia de varejo de produtos eletrônicos que comercializa itens nas categorias de Informática, Fotografia, Tablets, Eletrodomésticos, Celulares, Áudio, Wearables e Acessórios. As vendas são realizadas por meio de diferentes canais (Marketplace, Loja Física, E-commerce, Aplicativo e Televendas) e atendidas por uma equipe de vendedores distribuídos em 9 estados brasileiros.
O objetivo deste estudo é analisar o desempenho de vendas ao longo de 2024, identificando padrões e oportunidades que sirvam de base para decisões estratégicas de negócio.
A análise buscou responder às seguintes perguntas:

Qual foi o faturamento total de 2024?
Qual mês teve o maior e menor faturamento?
Qual a taxa de cancelamento e devolução?
Quais foram os 5 produtos mais vendidos em quantidade?
Quais categorias geraram mais receita?
Quais foram os 3 melhores vendedores em faturamento?
Algum estado se destacou em vendas?
Qual canal de venda trouxe mais receita?
Qual forma de pagamento é mais usada?

Os dados utilizados foram extraídos do arquivo "vendas_2024_NovaTech.csv", contendo 5.000 registros de pedidos realizados entre 01/01/2024 e 30/12/2024.

# Metodologia

A análise foi realizada utilizando o DuckDB (SQL) como ferramenta principal para consulta, organização, limpeza e processamento dos dados.
Inicialmente, os dados brutos foram importados para o DuckDB via read_csv_auto(), criando a tabela base "vendas". Antes da análise, foi realizado um processo completo de limpeza e preparação, que incluiu:
Verificação de nulos para identificar registros incompletos — foram encontrados 164 registros sem id_vendedor/nome_vendedor e 46 sem canal_venda. Após investigação, os registros foram mantidos e preenchidos com "Desconhecido" e "Não Informado" respectivamente, pois tratavam-se de vendas legítimas.
Correção de tipos de dados — a coluna data_venda estava armazenada como VARCHAR e foi convertida para DATE. A coluna desconto estava em formato de texto com símbolo "%" e foi convertida para DOUBLE.
Normalização de formatos de data — foram identificados 3 formatos distintos no mesmo campo (DD/MM/YYYY, YYYY/MM/DD e DD-MM-YYYY), todos normalizados para o tipo DATE padrão utilizando CASE + STRPTIME com detecção automática do separador.
Correção de valores inválidos — identificados 105 registros com quantidade negativa, corrigidos com ABS(), e 100 registros com total_venda = 0, removidos por serem considerados corrompidos.

⚠️ Observação: Em um ambiente de produção, seria necessário validar com o time de negócio se os registros com quantidade negativa e total zerado representam estornos, devoluções ou erros de sistema antes de qualquer correção.

Padronização de texto — 59 registros com status "completed" em inglês foram substituídos por "Concluído", e 54 registros com forma_pagamento em letras maiúsculas foram normalizados para o formato título.
Após a limpeza, a tabela "vendas_limpa" ficou com 4.900 registros prontos para análise. Todas as análises de faturamento foram realizadas filtrando apenas os pedidos com status "Concluído", excluindo cancelamentos e devoluções.
Para a visualização e interpretação dos resultados, foi utilizado o Power BI, onde os dados processados no DuckDB foram exportados e organizados em um dashboard interativo com duas páginas. Os principais recursos visuais adotados foram gráficos de barras horizontais, gráficos de rosca e cartões de métricas, permitindo comparar de forma clara e intuitiva os resultados das análises realizadas.
Resultados Principais (Insights)
Faturamento total e mensal:
O faturamento total de 2024 considerando apenas pedidos Concluídos foi de R$ 10.645.968,50. O mês de maior faturamento foi Julho, com R$ 1.019.507,84 — único mês a superar R$ 1 milhão. O mês de menor faturamento foi Setembro, com R$ 721.345,49 e apenas 243 pedidos concluídos.
Taxa de cancelamento e devolução:
Do total de 4.900 pedidos, 3.452 foram Concluídos (70,45%), 761 foram Cancelados (15,53%) e 687 foram Devolvidos (14,02%). A taxa de perda total foi de 29,55% — quase 1 em cada 3 pedidos não convertido em receita efetiva.
Produtos mais vendidos em quantidade:
Os 5 produtos mais vendidos foram Caixa de Som JBL Flip 6 (761 unidades), Notebook Dell Inspiron (754), Smartwatch Xiaomi Band 8 (753), Webcam Logitech C920 (724) e Mouse Logitech MX Master (716). O Notebook Dell Inspiron, apesar de ser o 2º em quantidade, foi o mais rentável com R$ 2.029.930,01 em faturamento.
Categorias que geraram mais receita:
Informática liderou com R$ 3.420.048,53 (32,13% do total), seguida por Fotografia com R$ 2.110.028,41 (19,82%) e Tablets com R$ 1.652.462,75 (15,52%). Wearables e Acessórios juntos representaram menos de 3% da receita total.
Melhores vendedores:
Os 3 melhores vendedores em faturamento foram Amanda Costa (R$ 1.168.481,24), Camila Ferreira (R$ 1.115.028,12) e Roberto Silva (R$ 1.082.260,06), representando juntos 31,6% de todo o faturamento de 2024.
Destaque por estado:
SP se destacou com R$ 2.025.798,00 (19,03% do total), quase o dobro do 2º colocado RS (R$ 1.203.845,65). Os demais 8 estados apresentaram distribuição equilibrada, todos entre 9% e 11%.
Canal de venda e forma de pagamento:
O Marketplace liderou em receita entre os canais de venda. O PIX foi a forma de pagamento mais utilizada, com 1.169 pedidos (33,86%) e R$ 3.690.796,00 em faturamento, seguido pelo Cartão Crédito com 1.119 pedidos (32,42%).
Conclusão
A análise dos dados de 2024 revelou um faturamento sólido de R$ 10,6 milhões, com Julho como mês de pico e Setembro como principal ponto de atenção. A taxa de perda de 29,55% (cancelamentos + devoluções) é o indicador que mais merece atenção — quase 1 em cada 3 pedidos não se converte em receita efetiva.
Informática domina o portfólio com 32% da receita, enquanto Wearables e Acessórios somados representam menos de 3%, indicando categorias que precisam de revisão estratégica. SP concentra 19% das vendas, mas os demais estados mostram distribuição equilibrada. O PIX já é o principal meio de pagamento, consolidado como preferência dos clientes. Os 3 melhores vendedores respondem por quase 1/3 do faturamento total, evidenciando alta concentração de receita em poucos profissionais.
