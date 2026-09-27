# MVP de Engenharia de Dados: Contratações Públicas do Pará (PNCP)

Pipeline de dados de ponta a ponta construído no **Databricks Free Edition**, partindo da coleta via API pública do Portal Nacional de Contratações Públicas até a resposta de perguntas de negócio sobre as contratações do estado do Pará.

**Arquitetura:** Bronze → Silver → Gold, em Delta Lake, com catálogo de dados no Unity Catalog.

| Notebook | Etapa |
|---|---|
| [MVP camada bronze.ipynb](MVP%20camada%20bronze.ipynb) | Coleta na API e carga do dado bruto |
| [MVP camada silver.ipynb](MVP%20camada%20silver.ipynb) | Limpeza, tipagem e padronização |
| [MVP camada gold.ipynb](MVP%20camada%20gold.ipynb) | Modelagem em esquema estrela e catálogo |
| [04_qualidade_dados.ipynb](04_qualidade_dados.ipynb) | Verificação de qualidade dos dados |
| [05_analise.ipynb](05_analise.ipynb) | Consultas e respostas às perguntas |

---

## 1. Contexto de Negócio e Perguntas

### O problema

A Lei 14.133/2021 obriga todos os órgãos da administração pública a publicar suas contratações no Portal Nacional de Contratações Públicas. Isso criou uma base nacional única, mas o portal foi desenhado para consulta caso a caso: quem quer entender o comportamento agregado de um estado precisa coletar, organizar e modelar esses dados por conta própria.

Este MVP monta esse caminho para o **estado do Pará**, no período de **janeiro a agosto de 2026**, cobrindo todas as modalidades de contratação. O objetivo é permitir que um órgão de controle, um pesquisador ou um fornecedor consiga responder perguntas sobre padrão de gasto, eficiência de negociação e uso de contratação direta sem precisar navegar registro a registro no portal.

### Perguntas de negócio

1. Quais municípios e órgãos apresentam a maior economia entre o valor estimado e o valor homologado?
2. A taxa de homologação de uma contratação depende mais da modalidade escolhida ou do valor envolvido?
3. Como as modalidades se distribuem em quantidade e em valor, e qual o peso da contratação direta?
4. Como o volume e o valor das contratações se distribuem ao longo dos meses?

As quatro perguntas foram respondidas. As respostas e suas limitações estão na seção 6.

---

## 2. Carga dos Dados

### Fonte

Os dados vêm da **API de Consulta do PNCP**, endpoint `GET /v1/contratacoes/publicacao`, documentado no [Swagger oficial](https://pncp.gov.br/api/consulta/swagger-ui/index.html).

O PNCP expõe quatro APIs distintas. A escolha pela API de Consulta foi deliberada:

| API | Finalidade | Uso neste trabalho |
|---|---|---|
| `/api/consulta` | Dados abertos para terceiros, documentada e sem autenticação | **Escolhida** |
| `/api/pncp` | Integração para órgãos publicarem dados; escrita exige credenciamento | Descartada |
| `/api/search` | Backend da busca do portal, não documentada como API pública | Descartada |
| Compras.gov.br | Sistema distinto, cobre apenas a esfera federal | Descartada |

### Licença de uso

Os dados são públicos e de divulgação obrigatória por força do art. 174 da Lei 14.133/2021. A API de consulta é aberta e não exige autenticação. O uso neste trabalho é acadêmico e não comercial, e os dados brutos não são redistribuídos neste repositório. Os termos completos estão em [PNCP em Dados Abertos](https://www.gov.br/pncp/pt-br/acesso-a-informacao/dados-abertos).

### Como a coleta foi feita

O notebook da camada Bronze percorre o período de forma sistemática, em três níveis aninhados:

1. **Por modalidade.** A API não aceita consulta de todas as modalidades de uma vez; o parâmetro `codigoModalidadeContratacao` é obrigatório. O notebook itera sobre os 13 códigos da tabela de domínio.
2. **Por mês.** O período é dividido em janelas mensais. Consultas menores são mais estáveis e permitem repetir apenas o mês que falhou.
3. **Por página.** Cada resposta traz no máximo 50 registros. O loop avança até atingir o `totalPaginas` informado pela API ou até receber uma página incompleta.

O tratamento de falhas usa nova tentativa com espera exponencial para erros 429 e 5xx, trata HTTP 204 como ausência de resultados e registra em log qualquer janela que não tenha sido coletada.

Cada execução grava os registros em arquivos JSON Lines dentro de um Volume do Unity Catalog, numa pasta identificada por timestamp, preservando o histórico de coletas.

![Arquivos brutos no Volume](imagens/bronze%20json%20raw_pncp%20mvp.png)

**Limitação encontrada:** o Databricks Free Edition restringe o acesso de saída à internet a um conjunto de domínios confiáveis. O domínio do PNCP se mostrou acessível, o que permitiu fazer a extração diretamente do notebook. Caso estivesse bloqueado, a alternativa seria executar a coleta localmente e subir os arquivos para o Volume.

---

## 3. Modelagem e Catálogo de Dados

### Esquema estrela

A camada Gold organiza os dados em uma tabela fato e quatro dimensões:

```
                    dim_tempo
                        |
    dim_orgao ---- fato_contratacao ---- dim_modalidade
                        |
                  dim_municipio
```

| Tabela | Granularidade | Conteúdo |
|---|---|---|
| `fato_contratacao` | Uma linha por contratação publicada | Métricas de valor, economia, prazo e flags de qualidade |
| `dim_orgao` | Um CNPJ por linha | Razão social, esfera e poder |
| `dim_municipio` | Um código IBGE por linha | Nome do município e UF |
| `dim_modalidade` | Um código por linha | Nome e classificação entre licitação e contratação direta |
| `dim_tempo` | Uma data por linha | Ano, mês, trimestre e dia da semana |

**Decisão de chaves:** o modelo usa as chaves naturais da fonte, como CNPJ, código IBGE e código de modalidade, em vez de chaves substitutas. Elas já são estáveis e únicas no PNCP, e isso mantém as consultas legíveis sem perder integridade. As chaves primárias e estrangeiras foram declaradas no Unity Catalog como restrições informativas, o que documenta o modelo e habilita o diagrama de relacionamentos.

**Decisão de modelagem:** a dimensão de modalidade recebeu o atributo derivado `contratacao_direta`, marcando dispensa e inexigibilidade. Essa classificação não existe na fonte e foi atribuída com base na Lei 14.133/2021, para permitir responder diretamente à pergunta 3.

### Catálogo de dados

Todas as tabelas e colunas receberam comentários descritivos aplicados via `COMMENT ON TABLE` e `ALTER TABLE ... ALTER COLUMN ... COMMENT`. Cada coluna documenta seu significado, o domínio de valores esperado e a linhagem, ou seja, de onde veio e qual transformação sofreu.

![Catálogo no Unity Catalog](imagens/print%20catalog%20mvp.png)

![Colunas da fato_contratacao com chaves e comentários](imagens/print%20gold%20fato%201%20mvp.png)

![Colunas da fato_contratacao, continuação](imagens/print%20gold%20fato%202%20mvp.png)

![Colunas da fato_contratacao, continuação](imagens/print%20gold%20fato%203%20mvp.png)

---

## 4. Pipeline de Dados

### Visão geral

```
API PNCP → Volume (JSON Lines) → bronze.pncp_contratacoes
                                          ↓
                                 silver.contratacoes
                                          ↓
              gold.fato_contratacao + 4 dimensões
```

A linhagem completa é rastreada automaticamente pelo Unity Catalog:

![Lineage da cadeia completa](imagens/print%20lineage%20mvp.png)

### Camada Bronze

Preserva o dado exatamente como veio da API, sem nenhuma transformação. Acrescenta apenas colunas de controle de ingestão: arquivo de origem, timestamp da carga, URL da fonte e identificador da execução. Essa camada existe para garantir rastreabilidade, permitindo reprocessar as camadas seguintes sem precisar consultar a API de novo.

### Camada Silver

Aplica cinco grupos de transformação:

1. **Achatamento** dos campos aninhados que a API retorna como objetos JSON: `orgaoEntidade`, `unidadeOrgao` e `amparoLegal`.
2. **Padronização de nomes** para snake_case, com descarte das colunas sem uso analítico.
3. **Tipagem**: datas em texto ISO convertidas para `timestamp`, valores monetários para `decimal(18,2)`, o que evita imprecisão de ponto flutuante em somas.
4. **Deduplicação** por `numero_controle_pncp`, mantendo a versão com data de atualização mais recente.
5. **Colunas derivadas**: economia em valor e em percentual, prazo de propostas e a classificação de desfecho.

**Decisão central desta camada:** nenhuma linha é excluída. Os problemas de qualidade são marcados em colunas de flag e filtrados apenas no momento da análise. Isso mantém os registros problemáticos mensuráveis a qualquer momento e evita que o descarte silencioso de dados produza resultados otimistas.

### Camada Gold

Monta o esquema estrela descrito na seção 3, aplica os comentários do catálogo e declara as restrições de chave. Ao final, verifica a integridade referencial conferindo que nenhuma linha do fato aponta para chave inexistente nas dimensões.

---

## 5. Qualidade de Dados

A verificação cobre as cinco dimensões de qualidade, atributo por atributo, e está documentada em [04_qualidade_dados.ipynb](04_qualidade_dados.ipynb).

### Problemas detectados e tratamentos

| # | Problema | Dimensão | Tratamento |
|---|---|---|---|
| 1 | Campos aninhados em JSON | Consistência | Achatados em colunas próprias na Silver |
| 2 | Datas e valores como texto | Consistência | Convertidos para `timestamp` e `decimal(18,2)` |
| 3 | `valor_total_homologado` ausente em parte dos registros | Completude | Preservado como nulo, marcado em `tem_homologado` e filtrado nas análises |
| 4 | `valor_total_estimado` igual a zero | Acurácia | Marcado em `estimado_valido` e excluído dos cálculos de economia |
| 5 | Homologado idêntico ao estimado em proporção alta | Acurácia | Marcado em `valor_identico`, com análises reportadas com e sem esses casos |
| 6 | Valores estimados em ordem de grandeza impossível | Outliers | Uso de mediana em vez de média e exigência de mínimo de 30 contratações por grupo |
| 7 | Risco de duplicatas entre janelas e execuções | Unicidade | Deduplicação por `numero_controle_pncp` |
| 8 | Colunas integralmente nulas, como `orgaoSubRogado` | Completude | Descartadas na Silver |
| 9 | Códigos de esfera e poder fora da tabela de domínio documentada | Consistência | Quantificados e reportados; domínio real mais amplo que o manual indica |
| 10 | Prazo de proposta com valores impossíveis, chegando a mais de 9 mil dias | Acurácia | Marcado em `prazo_proposta_invalido` |

### O achado mais relevante

Uma proporção alta de contratações registra o valor homologado exatamente igual ao valor estimado. Dificilmente isso significa que todas essas licitações fecharam no preço previsto. O mais provável é que o órgão tenha repetido o valor estimado ao preencher o campo, sem registrar o que de fato saiu da disputa. Não é economia zero, é ausência de informação registrada como número.

Esses registros foram mantidos, porque fazem parte do dado real do PNCP. A camada Gold apenas os identifica com a flag `valor_identico`, e as análises de economia os deixam de fora. Se fossem descartados sem aviso, a economia média apareceria bem maior do que é.

### Validações que passaram sem problema

CNPJ com 14 dígitos, código IBGE com 7 dígitos, UF dentro do recorte e código de modalidade dentro do intervalo válido não apresentaram nenhuma inconsistência. A verificação foi executada e documentada mesmo assim, para evidenciar que a checagem ocorreu.

---

## 6. Análise de Dados

As consultas e os resultados estão em [05_analise.ipynb](05_analise.ipynb).

### Pergunta 1: economia por município

Entre os 34 municípios com pelo menos 30 contratações, a mediana de economia varia de 0,4% em Ipixuna do Pará a 38,5% em Aurora do Pará. Marabá e Santarém dão os resultados mais confiáveis, com 257 contratações a 31,9% e 145 a 30,3%, porque têm volume suficiente para que o número não dependa de poucos casos.

Belém tem 1.652 contratações e mediana de 23%, mas a soma da economia é negativa em quase R$ 29 bilhões. Na maior parte dos processos o município fecha abaixo do valor estimado, e alguns contratos de valor muito alto fecham acima o bastante para inverter o total.

As somas de valor estão comprometidas por registros com valores impossíveis, como Itaituba com R$ 360 bilhões de economia. A resposta usa a mediana, e o total aparece apenas como indicativo.

### Pergunta 2: modalidade ou valor

A modalidade influencia mais. A amplitude entre modalidades é de 51,6 pontos percentuais, contra 19,9 pontos entre as faixas de valor.

A diferença vem do tipo de processo. Dispensa e pregão eletrônico, com 71,0% e 66,6%, têm rito rápido e padronizado, e chegam ao fim em poucas semanas. O credenciamento, com 19,4%, funciona como chamamento aberto por longos períodos, então a ausência de homologação nele é esperada. O pregão presencial aparece com 32,9%, mas tem só 73 registros, base pequena demais para conclusão.

O valor também afeta o resultado, embora de forma irregular. O quartil mais baixo tem a maior taxa, 77,9%. O segundo cai para 58,0% e depois a taxa volta a subir até 64,9% no quartil mais alto. Não há relação direta entre valor maior e menor conclusão.

### Pergunta 3: peso da contratação direta

Dispensa e inexigibilidade somam 57,2% das contratações do período, com 6.438 e 3.534 registros de um total aproximado de 17.450. A maior parte das contratações públicas do estado não passa por licitação competitiva. O pregão eletrônico responde por 31,9%.

Em valor, a leitura exige cuidado. A inexigibilidade aparece com 86,4% do total financeiro, mas o número está dominado pelos mesmos registros anômalos da pergunta 1. Os R$ 373 bilhões atribuídos a ela superam em muito o orçamento anual do estado, o que indica erro de preenchimento em poucos registros.

O padrão da dispensa se sustenta: 36,9% das contratações e 0,8% do valor, uso compatível com o que a Lei 14.133 prevê para compras pequenas e frequentes. O pregão eletrônico tem perfil mais equilibrado, com 31,9% das contratações e 8,1% do valor.

### Pergunta 4: distribuição mensal

O volume cresce ao longo do período, de 1.630 contratações em janeiro para 2.544 em agosto, alta de 56%. Parte disso é sazonalidade orçamentária e parte pode ser aumento da adesão dos órgãos ao PNCP.

A taxa de homologação sobe até 65,2% em abril e cai de forma contínua depois, chegando a 42,3% em agosto. A queda não indica piora na gestão. É o efeito do tempo de maturação dos processos, que justificou o corte temporal usado na pergunta 2: contratações recentes ainda não tiveram prazo para ser concluídas e registradas.

Abril também concentra o valor estimado anômalo de R$ 370 bilhões, provavelmente o mesmo registro que distorce a inexigibilidade na pergunta 3.

### Discussão geral

O conjunto das respostas aponta três coisas. A contratação direta responde por mais da metade dos processos no Pará, ainda que concentrada em valores baixos. O que determina se uma contratação chega ao fim registrado é principalmente o rito escolhido, e não o valor envolvido. E o PNCP serve bem para contagens e medidas de tendência central, mas não para somas financeiras sem tratamento prévio de outliers, além de ter uma parcela relevante de registros que apenas repetem o valor estimado no campo do homologado.

Esse último ponto é também um resultado do trabalho: o pipeline respondeu às perguntas e, no processo, delimitou até onde a fonte permite respondê-las.

---

## 7. Autoavaliação

> - **Objetivos atingidos.** Consegui analisar com clareza os diferentes dados do pncp, deixando claro que 
> - **Dificuldades encontradas.** A maior dificuldade foi decidir qual das apis utilizar, pois existe a pncp/pncp, a pncp/search, a pncp/consulta e a do comprasgov. Além disso, organizar os dados em um formato que favorecesse as analises na camada ouro também foi desafiador, criar um design funcional e eficiente exige conhecer bem os dados que se está trabalhando.
> - **Decisões que tomaria diferente.** Eu gostaria de ter incluido outros endpoints além do de contratacões, visando possibilitar analises mais completas e úteis.
> - **O que aprendeu.** Aprendi a organizar dados externos de forma a atingir os objetivos de negócio que desejo, além de aprender sobre a plataforma Databricks, data lakes e modelagem de dados.
> - **Trabalhos futuros.** Carga incremental via `/v1/contratacoes/atualizacao`, cruzamento com `/v1/contratos` para analisar fornecedores, extensão para outras UFs.

---

## Tecnologias

Databricks Free Edition, Apache Spark (PySpark), Delta Lake, Unity Catalog, SQL e Python.