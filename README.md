# 📊 Facilita BI — Ecossistema Analítico 0800

> Plataforma de BI que consolida a operação de atendimento 0800 de uma central entre as maiores do Brasil em três painéis vivos — Diretoria, Cliente e Operacional — a partir de uma única fonte de dados.

![Power BI](https://img.shields.io/badge/Power%20BI-Analytics-F2C811)
![DAX](https://img.shields.io/badge/DAX-Modeling-2b2b2b)
![Power Query](https://img.shields.io/badge/Power%20Query-M-2b9d94)
![Excel](https://img.shields.io/badge/Excel-Data%20Ops-217346)

---

# 🚀 Visão Geral

| Item            | Informação                                     |
| --------------- | ---------------------------------------------- |
| **Stack**       | Power BI · DAX · Power Query (M) · Excel       |
| **Atualização** | Semanal · automática                           |
| **Audiências**  | Diretoria · Gestão interna · Clientes externos |

---

# 🧠 O Problema

Uma central de atendimento 0800 servindo dezenas de associações de proteção veicular como clientes externos, além de gestores internos.

Cada associação precisava ver apenas seus próprios dados:

* sem RLS disponível no ambiente
* sem vazamento entre clientes
* sem duplicar modelos
* sem manter dezenas de PBIX diferentes manualmente

Três audiências completamente diferentes consumindo os mesmos dados:

| Audiência      | Necessidade                                  |
| -------------- | -------------------------------------------- |
| Diretoria      | Visão estratégica consolidada                |
| Gestão interna | Prestação de contas operacional e financeira |
| Operação       | Performance detalhada de atendimento         |

> Um modelo.
> Três perspectivas.
> Zero vazamento de dado.

---

# 🏗️ A Solução

## Arquitetura ponta a ponta

```txt
WebAssiste (sistema operacional — origem dos dados)
↓
Sheets / Excel (buffer de exportação)
↓
Power Query M (limpeza + filtro por cliente)
↓
Power BI (modelo tabular + medidas DAX)
↓
3 dashboards com audiências distintas
```

---

# 🔒 Multi-tenancy via Power Query

O isolamento de dados acontece no nível do modelo, não no visual.

O filtro `[Cliente] = "..."` é aplicado nas tabelas fato durante o carregamento.

Cada cliente recebe um arquivo com seus dados já isolados.

O dado simplesmente **não existe** no modelo do cliente errado.

## Filtro de isolamento por cliente (Power Query M)

```m
Table.SelectRows(Fonte, each [Cliente] = ParametroCliente)
```

---

# 📊 Os três dashboards

| Dashboard                 | Audiência                 | Visão                          |
| ------------------------- | ------------------------- | ------------------------------ |
| Diretoria                 | Diretor                   | KPIs estratégicos consolidados |
| Prestação de Contas       | Gestão interna + Clientes | Serviços prestados e valores   |
| Operacional — Atendimento | Gerente operacional       | Performance detalhada          |

---

# 🖼️ Screenshots

## Dashboard Diretoria — KPIs Executivo

![Dashboard Diretoria](./assets/diretor-kpis.png)

---

## Dashboard Prestação de Contas — Análise de Serviços

![Prestação de Contas](./assets/prestacao-contas.png)

---

## Dashboard Operacional — Relatório de Atendimento

![Operacional](./assets/operacional.png)

---

# 🧮 Medidas DAX principais

## Classificação de status de atendimento (Power Query M)

```m
#"Status Atendimento" =
Table.AddColumn(
    #"Linhas Classificadas",
    "STATUS ATENDIMENTO",
    each
        if [Tempo Atendimento Solicitante] = null or [Tempo Atendimento Solicitante] = 0 then "ZERADO"
        else if [Tempo Atendimento Solicitante] > (60 / 1440) then "ATIPICO"
        else if [Tempo Atendimento Solicitante] > (6 / 1440) then "EXCEDIDO"
        else "OK",
    type text
)
```

---

## TMA formatado em HH:MM

```dax
TMA Formatado =
VAR TotalSegundos =
    AVERAGE('TMA-2'[TEMPO ACIONAMENTO]) * 86400

VAR Horas =
    INT(TotalSegundos / 3600)

VAR Minutos =
    INT(MOD(TotalSegundos, 3600) / 60)

VAR Segundos =
    INT(MOD(TotalSegundos, 60))

RETURN
FORMAT(Horas, "00") & ":" &
FORMAT(Minutos, "00") & ":" &
FORMAT(Segundos, "00")
```

O tempo vem da fonte como fração de dia (padrão Excel).

Converter diretamente para texto distorce a leitura.

A medida reconverte para segundos antes de formatar, garantindo precisão independente da origem.

---

## Ranking DENSE de atendentes

```dax
Rank Atendente =
RANKX(
    ALL('TMA-2'[Atendente]),
    [TOTAL SERVIÇOS],
    ,
    DESC,
    DENSE
)
```

`DENSE` evita pular posições em caso de empate.

Importante em contextos onde posição impacta premiação.

---

## Flag Top 10

```dax
Top 10 Flag =
IF([Rank Atendente] <= 10, 1, 0)
```

---

# 📈 KPIs monitorados

## Diretoria

* Churn rate e cancelamentos
* Relação atendimento vs cancelamento
* NPS e LTV consolidados
* Base ativa e evolução mensal
* % atendimentos dentro do prazo
* % acionamentos dentro do prazo
* % placas com acionamento

---

## Cliente / Prestação de Contas

* Valor total e médio por serviço
* Quantidade e tipos de serviço
* % de placas com acionamento
* Km total percorrido
* Ranking de prestadores
* Distribuição geográfica
* Heatmap hora × dia da semana

---

## Operacional

* TMA por atendente
* TMA de acionamento
* TMP do prestador
* % OK / Excedido / Atípico
* Top 10 atendentes
* Top 10 acionadores
* Top 10 prestadores
* Atendimentos atípicos

---

# ⚙️ Decisões Técnicas

## Por que filtro M em vez de RLS?

Ambiente sem Power BI Service configurado para workspace por cliente.

Filtro no modelo é mais seguro que filtro visual.

Não pode ser contornado pelo usuário final.

---

## Por que duas versões de tempo?

DAX não possui tipo duração nativo.

* Para ranking → número
* Para exibição → texto formatado

São medidas diferentes com objetivos diferentes.

---

## Por que DENSE no RANKX?

Empates sem pular posição.

Em contextos onde posição impacta premiação, ranking com gaps cria conflito operacional.

A matemática parece pequena até RH precisar explicar porque existem dois terceiros lugares e ninguém ficou em quarto. O Excel vira tribunal. ☠️

---

# 📚 Glossário do domínio

| Termo        | Definição                  |
| ------------ | -------------------------- |
| TMA          | Tempo Médio de Atendimento |
| TMP          | Tempo Médio do Prestador   |
| LTV          | Lifetime Value             |
| Churn        | Taxa de cancelamento       |
| Base Ativa   | Veículos ativos            |
| Acionamento  | Despacho do prestador      |
| SLA          | Prazo prometido            |
| Antes/Depois | Momento do cancelamento    |

---

# ⏱️ SLA de atualização

| Item                    | Valor             |
| ----------------------- | ----------------- |
| Janela de processamento | Segunda a domingo |
| Liberação para clientes | Segunda-feira     |
| Periodicidade           | Semanal           |
| Histórico mantido       | 12 meses          |

---

# 📂 Estrutura sugerida no GitHub

```txt
/assets
├── dashboard-diretoria.png
├── dashboard-prestacao-contas.png
├── dashboard-operacional.png
└── heatmap-operacional.png
```

---

# 🔐 Observação

Dados, nomes de clientes e informações sensíveis foram removidos ou abstraídos.

O foco é demonstrar:

* arquitetura
* padrões técnicos
* modelagem
* governança
* lógica de negócio
* decisões analíticas

---

# 👨‍💻 Autor

Marcela Silva

---
