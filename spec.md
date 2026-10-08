# Especificação Funcional — Zona Azul Digital

## 1. Objetivo e escopo

O projeto tem como objetivo especificar uma API REST para gerenciamento de bilhetes de estacionamento rotativo (Zona Azul Digital).

O sistema deve permitir abrir, encerrar e cancelar bilhetes, listar estacionamentos ativos, consultar o histórico por placa e gerar relatórios diários.

A API deve calcular os valores de estacionamento conforme a duração, respeitando a tarifa definida, as frações de cobrança, a tolerância gratuita e o teto máximo por bilhete.

O sistema deve validar os dados recebidos, garantir a integridade dos registros e respeitar as regras de negócio e transições de estado dos bilhetes.

O escopo contempla exclusivamente a API REST, sem desenvolvimento de interfaces gráficas, aplicativos ou back-office.

## 2. Parâmetros da variante

Os valores abaixo são obrigatórios e correspondem ao arquivo `variante/params.json`.

| Parâmetro | Valor | Descrição |
|---|---|---|
| `TARIFA_HORA_CENTAVOS` | 400 | Tarifa de 400 centavos por hora. |
| `FRACAO_MINUTOS` | 30 | Fração de 30 minutos, custando 200 centavos. |
| `TETO_DIARIO_CENTAVOS` | 5000 | Valor máximo de 5000 centavos por bilhete. |
| `TOLERANCIA_MINUTOS` | 10 | Até 10 minutos gratuitos, sem desconto da tolerância após ultrapassá-la. |
| `PORTA_SERVICO` | 8001 | Porta pública obrigatória para acesso à API. |

- Base URL: `http://localhost:8001`
- Fuso horário das respostas: `-03:00`.
- Datas e horários: ISO-8601 com fuso.
- Valores monetários: centavos inteiros, nunca ponto flutuante.

## 3. Modelo de dados

### 3.1. Entidade Bilhete

| Campo | Tipo | Descrição |
|---|---|---|
| `id` | Inteiro | Identificador único do bilhete. |
| `placa` | String | Placa com exatamente 7 caracteres alfanuméricos maiúsculos. |
| `entrada` | String ISO-8601 | Data e horário de abertura. |
| `saida` | String ISO-8601, opcional | Horário do encerramento. |
| `status` | String | Estado atual: aberto, encerrado ou cancelado. |
| `minutos` | Inteiro, opcional | Duração calculada no encerramento. |
| `valor_centavos` | Inteiro, opcional | Valor calculado no encerramento. |

Campos opcionais representam informações que ainda não foram produzidas. Os endpoints devem devolver apenas os campos previstos em seus respectivos contratos de resposta.

### 3.2. Estados do bilhete

| Estado atual | Operação | Novo estado |
|---|---|---|
| Inexistente | Abrir | Aberto |
| Aberto | Encerrar | Encerrado |
| Aberto | Cancelar | Cancelado |
| Encerrado | Encerrar | Conflito 409 |
| Encerrado | Cancelar | Conflito 409 |
| Cancelado | Cancelar | Conflito 409 |

Bilhetes encerrados ou cancelados não podem retornar ao estado aberto. Um novo estacionamento exige a criação de outro bilhete.

## 4. Funcionalidades e contrato REST

### RF-01 — Abrir bilhete (UC1)

**Endpoint:** `POST /bilhetes`

**Entrada:**

- `placa`: obrigatória, string de 7 caracteres alfanuméricos maiúsculos.
- `entrada`: opcional, data e hora ISO-8601 com fuso.
- Na ausência de `entrada`, utilizar a data e hora atual do serviço.

**Resposta de sucesso — HTTP 201:**

`{"id":1,"placa":"ABC1D23","entrada":"2026-10-05T08:00:00-03:00","status":"aberto"}`

**Erros:**

- 422 `{"erro":"placa_invalida"}` — placa ausente ou inválida.
- 422 `{"erro":"entrada_invalida"}` — entrada presente com formato inválido.
- 409 `{"erro":"bilhete_em_aberto"}` — placa já possui bilhete aberto.

**Aceite AC-01:** uma abertura válida cria um bilhete único com status `aberto`, retorna HTTP 201 e permite posterior consulta.

### RF-02 — Encerrar bilhete (UC2)

**Endpoint:** `POST /bilhetes/{id}/encerramento`

**Entrada:** identificador do bilhete na rota.

**Comportamento:**

1. Localizar o bilhete.
2. Verificar se está aberto.
3. Registrar a saída utilizando o relógio atual.
4. Calcular a duração e o valor conforme RN-01 e RN-02.
5. Alterar seu estado para encerrado.
6. Persistir as informações.

**Resposta de sucesso — HTTP 200:**

`{"id":1,"placa":"ABC1D23","entrada":"2026-10-05T08:00:00-03:00","saida":"2026-10-05T09:00:00-03:00","minutos":60,"valor_centavos":400}`

**Erros:**

- 404 `{"erro":"bilhete_nao_encontrado"}` — bilhete inexistente.
- 409 `{"erro":"bilhete_ja_encerrado"}` — bilhete já encerrado.

**Aceite AC-02:** um encerramento válido registra a saída, calcula corretamente a duração e a cobrança, retorna HTTP 200 e impede outro encerramento bem-sucedido.

### RF-03 — Listar bilhetes ativos (UC3)

**Endpoint:** `GET /bilhetes/ativos`

**Comportamento:** retornar todos os bilhetes cujo status seja `aberto`, ordenados do mais recente para o mais antigo pelo horário de entrada.

**Resposta:** HTTP 200 com array JSON de bilhetes abertos.

**Aceite AC-03:** nenhum bilhete encerrado ou cancelado aparece na listagem. Quando não existem ativos, retornar `[]`.

### RF-04 — Relatório diário (UC4)

**Endpoint:** `GET /relatorios/diario?data=AAAA-MM-DD`

**Entrada:** parâmetro obrigatório `data`.

**Resposta de sucesso — HTTP 200:**

`{"data":"2026-10-05","total_bilhetes":12,"faturamento_centavos":8400,"tempo_medio_minutos":47}`

**Regras:**

- O campo `data` identifica o dia solicitado.
- `tempo_medio_minutos` considera somente bilhetes encerrados no dia.
- A média deve ser arredondada ao inteiro mais próximo, com 0,5 para cima.
- O faturamento deve ser apresentado em centavos inteiros.
- O filtro temporal deve considerar o fuso `-03:00`.

**Erro:**

- 422 `{"erro":"data_invalida"}` — data ausente ou inválida.

**Aceite AC-04:** uma data válida retorna os quatro campos obrigatórios com os tipos corretos. O tempo médio considera os encerramentos do dia e aplica o arredondamento especificado.
### RF-05 — Cancelar bilhete (UC5)

**Endpoint:** `POST /bilhetes/{id}/cancelamento`

**Comportamento:**

- Somente bilhetes abertos podem ser cancelados.
- Alterar o estado para `cancelado`.
- Não gerar cobrança ou horário de saída.
- Manter o bilhete disponível no histórico.

**Resposta:** HTTP 200 com os dados do bilhete e `status: "cancelado"`, sem `saida` ou `valor_centavos`.

**Erros:**

- 404 `{"erro":"bilhete_nao_encontrado"}`.
- 409 `{"erro":"bilhete_nao_aberto"}`.

**Aceite AC-05:** cancelar um bilhete aberto retorna 200, registra o estado cancelado, não gera cobrança e libera a placa para um novo estacionamento.

### RF-06 — Histórico por placa (UC6)

**Endpoint:** `GET /bilhetes?placa=ABC1D23`

**Entrada:** placa obrigatória no formato definido.

**Comportamento:** localizar todos os bilhetes associados à placa, independentemente do estado, ordenados do mais recente ao mais antigo.

**Resposta:** HTTP 200 com array JSON.

**Erro:**

- 422 `{"erro":"placa_invalida"}`.

**Aceite AC-06:** retornar todos os registros da placa, incluindo abertos, encerrados e cancelados. Uma placa válida sem histórico retorna `[]`.

## 5. Regras de negócio

### RN-01 — Cálculo da cobrança (UC2 e UC7)

Para bilhetes com permanência superior à tolerância:

1. Calcular o tempo total entre entrada e saída.
2. Dividir a duração por `FRACAO_MINUTOS`.
3. Arredondar a quantidade de frações para cima.
4. Multiplicar pelo valor de cada fração.
5. Limitar o resultado a `TETO_DIARIO_CENTAVOS`.
6. Retornar o resultado em centavos inteiros.

O valor da fração é `400 / (60 / 30) = 200` centavos.

> [!WARNING]
> A tolerância não é descontada. Permanências de até 10 minutos são gratuitas. A partir do momento em que o limite é ultrapassado, cobra-se desde o primeiro minuto.

**Exemplos verificáveis:**

| Duração | Valor esperado |
|---|---|
| 0 minutos | 0 centavos |
| 10 minutos | 0 centavos |
| 11 minutos | 200 centavos |
| 30 minutos | 200 centavos |
| 31 minutos | 400 centavos |
| 60 minutos | 400 centavos |
| 61 minutos | 600 centavos |
| 750 minutos | 5000 centavos |
| 751 minutos | 5000 centavos |

> [!WARNING]
> O teto de 5000 centavos aplica-se individualmente a cada bilhete, mesmo quando a cobrança calculada ultrapassa esse valor.

### RN-02 — Integridade da cobrança

- O valor calculado não pode superar o teto.
- Bilhetes cancelados não produzem cobrança.
- O encerramento não pode cobrar duas vezes o mesmo bilhete.
- Valores monetários devem permanecer inteiros.

**Aceite AC-07:** todos os limites e resultados definidos na tabela de RN-01 são respeitados.

### RN-03 — Exclusividade de placa (UC8)

Uma placa só pode possuir um bilhete com status aberto simultaneamente.

- Uma segunda abertura para a mesma placa deve retornar HTTP 409 com `{"erro":"bilhete_em_aberto"}`.
- Bilhetes encerrados ou cancelados não impedem novas aberturas.
- A regra deve permanecer válida mesmo diante de requisições concorrentes.

**Aceite AC-08:** duas tentativas de abertura simultâneas para uma mesma placa não podem resultar em dois bilhetes abertos.

## 6. Validação e tratamento de erros

Todos os erros definidos no contrato devem retornar um objeto JSON contendo a propriedade `erro`.

| Situação | HTTP | Código |
|---|---|---|
| Placa ausente ou inválida | 422 | `placa_invalida` |
| Entrada inválida | 422 | `entrada_invalida` |
| Data inválida | 422 | `data_invalida` |
| Bilhete inexistente | 404 | `bilhete_nao_encontrado` |
| Encerramento duplicado | 409 | `bilhete_ja_encerrado` |
| Cancelamento de bilhete não aberto | 409 | `bilhete_nao_aberto` |
| Placa com bilhete aberto | 409 | `bilhete_em_aberto` |

**Precedência obrigatória:** validações de formato (422) devem ocorrer antes das verificações de conflito de negócio (409).

Não substituir os códigos de erro contratuais por mensagens personalizadas.

## 7. Critérios gerais de aceite

A API será considerada funcional quando:

- **AC-G01:** todos os endpoints RF-01 a RF-06 responderem nos caminhos e métodos definidos.
- **AC-G02:** os comportamentos UC7 e UC8 forem respeitados.
- **AC-G03:** todos os códigos HTTP e nomes de campos corresponderem ao contrato.
- **AC-G04:** horários de resposta estiverem em ISO-8601 com fuso `-03:00`.
- **AC-G05:** os cálculos respeitarem os parâmetros reais da variante.
- **AC-G06:** dados e estados permanecerem consistentes entre operações e consultas.
- **AC-G07:** respostas de erro utilizarem os identificadores exatos.
- **AC-G08:** as operações funcionarem independentemente de interface gráfica.

A arquitetura, persistência, dependências e procedimentos de execução serão definidos no `plan.md`. Os cenários detalhados de verificação serão definidos no `tests.md`.
