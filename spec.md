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

