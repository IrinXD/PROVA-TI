# Plano de Testes — Zona Azul Digital

## 1. Objetivo

Definir os testes automatizados necessários para verificar a API Zona Azul Digital, conforme `contrato.json`, `spec.md` e os parâmetros da variante.

Os testes devem ser organizados em unitários e de integração, com descrições em português, claras e objetivas.

Cada teste deve ter preparação, ação e resultado esperado. Não utilizar esperas reais, serviços externos ou banco de produção.

## 2. Estratégia de testes

| Tipo | Ferramentas | Objetivo |
|---|---|---|
| Unitário | Vitest e mocks | Verificar funções e regras de negócio isoladamente. |
| Integração HTTP | Vitest, Supertest e NestJS | Verificar rotas, validações, respostas e persistência. |
| Concorrência | Supertest e SQLite isolado | Garantir integridade em operações simultâneas. |
| Execução | Build e container | Verificar inicialização e configuração reproduzível. |

### 2.1. Preparação

- Utilizar `vi.mock` ou mocks injetáveis nos testes unitários do Prisma.
- Não acessar o banco real nos testes unitários.
- Utilizar banco SQLite temporário e isolado nos testes de integração.
- Limpar os registros entre cenários.
- Substituir o relógio por um relógio controlável.
- Congelar ou avançar o tempo diretamente, sem utilizar esperas.
- Utilizar as configurações da variante: tarifa 400, fração 30, teto 5000 e tolerância 10.
- Validar respostas JSON, códigos HTTP e efeitos persistidos.
- Não depender da ordem de execução dos testes.

## 3. Testes unitários

### 3.1. Cálculo de cobrança — RN-01, RN-02 e AC-07

Preparação comum: chamar `calcularCobranca` com horários controlados, considerando entrada em `2026-10-05T08:00:00-03:00` e saída no intervalo indicado.

| ID | Descrição do teste | Duração | Valor esperado |
|---|---|---|---|
| UT-01 | Não cobra por permanência zero | 0 min | 0 |
| UT-02 | Não cobra dentro da tolerância | 9 min | 0 |
| UT-03 | Não cobra no limite da tolerância | 10 min | 0 |
| UT-04 | Cobra ao ultrapassar a tolerância | 11 min | 200 |
| UT-05 | Cobra uma fração completa | 30 min | 200 |
| UT-06 | Cobra duas frações ao passar do limite | 31 min | 400 |
| UT-07 | Cobra uma hora corretamente | 60 min | 400 |
| UT-08 | Arredonda a fração adicional para cima | 61 min | 600 |
| UT-09 | Cobra a fração anterior ao teto | 720 min | 4800 |
| UT-10 | Cobra exatamente o teto | 750 min | 5000 |
| UT-11 | Não ultrapassa o teto | 751 min | 5000 |
| UT-12 | Mantém o teto em duração prolongada | 1440 min | 5000 |

Em todos esses testes, verificar também que `valor_centavos` é inteiro e nunca negativo.

### 3.2. Operações de negócio

| ID | Preparação | Ação | Resultado esperado |
|---|---|---|---|
| UT-13 | Prisma simulado, placa sem bilhete aberto | Abrir bilhete | Bilhete criado com status `aberto` |
| UT-14 | Prisma simulado, placa já ocupada | Abrir outro bilhete | Conflito `bilhete_em_aberto` |
| UT-15 | Bilhete aberto no Prisma simulado | Encerrar bilhete | Cobrança calculada e estado encerrado |
| UT-16 | Bilhete aberto no Prisma simulado | Cancelar bilhete | Estado cancelado, sem saída ou cobrança |
| UT-17 | Bilhete já encerrado | Encerrar novamente | Conflito `bilhete_ja_encerrado` |
| UT-18 | Simulação de falha durante a persistência | Executar encerramento | Operação não deixa estado parcialmente atualizado |
| UT-19 | Duas durações encerradas no mesmo dia: 20 e 21 min | Calcular média | Resultado inteiro igual a 21 minutos |

Os mocks devem simular respostas e falhas do Prisma sem acessar arquivos de banco.

## 4. Testes de integração HTTP

Os testes devem utilizar requisições reais à aplicação NestJS inicializada em ambiente de teste, com banco SQLite isolado e relógio controlado.

### 4.1. Abertura de bilhetes — UC1 e UC8

| ID | Preparação e ação | Resultado esperado |
|---|---|---|
| IT-01 | POST `/bilhetes` com placa `ABC1D23` válida | HTTP 201; campos `id`, `placa`, `entrada`, `status: aberto` |
| IT-02 | POST com placa válida, sem `entrada` | HTTP 201; entrada igual ao instante do relógio controlado |
| IT-03 | POST com corpo `{}` | HTTP 422; `{"erro":"placa_invalida"}` |
| IT-04 | POST com placa minúscula, curta, longa ou com símbolos | HTTP 422; `{"erro":"placa_invalida"}` |
| IT-05 | POST com entrada sem fuso ou data/hora malformada | HTTP 422; `{"erro":"entrada_invalida"}` |
| IT-06 | Abrir duas vezes a placa `ABC1D23` sem encerramento | Primeira resposta 201; segunda 409 `bilhete_em_aberto` |
| IT-07 | Placa já aberta; nova solicitação com a mesma placa e `entrada` inválida | HTTP 422 `entrada_invalida`, antes de verificar conflito |

### 4.2. Encerramento e cancelamento — UC2 e UC5

| ID | Preparação e ação | Resultado esperado |
|---|---|---|
| IT-08 | Bilhete aberto às 08:00; encerrar às 08:31 | HTTP 200; `minutos: 31`, `valor_centavos: 400` |
| IT-09 | Encerrar o mesmo bilhete novamente | HTTP 409; `bilhete_ja_encerrado` |
| IT-10 | Encerrar ID inexistente | HTTP 404; `bilhete_nao_encontrado` |
| IT-11 | Cancelar bilhete aberto | HTTP 200; status `cancelado`, sem `saida` ou `valor_centavos` |
| IT-12 | Cancelar bilhete já cancelado | HTTP 409; `bilhete_nao_aberto` |
| IT-13 | Cancelar bilhete encerrado | HTTP 409; `bilhete_nao_aberto` |
| IT-14 | Cancelar ID inexistente | HTTP 404; `bilhete_nao_encontrado` |
| IT-15 | Encerrar bilhete cancelado | HTTP 409; `bilhete_nao_aberto`, conforme convenção adicional adotada no projeto |

No IT-08, verificar também `id`, `placa`, `entrada` e `saida`, respeitando os nomes contratuais.

### 4.3. Listagens e histórico — UC3 e UC6

| ID | Preparação e ação | Resultado esperado |
|---|---|---|
| IT-16 | Consultar ativos sem nenhum bilhete | HTTP 200; `[]` |
| IT-17 | Criar dois bilhetes abertos em horários diferentes e consultar ativos | HTTP 200; ambos, mais recente primeiro |
| IT-18 | Criar bilhetes abertos, encerrados e cancelados; consultar ativos | Somente os abertos aparecem |
| IT-19 | Consultar histórico de uma placa com bilhetes em três estados | HTTP 200; todos os registros da placa, mais recentes primeiro |
| IT-20 | Consultar uma placa válida sem histórico | HTTP 200; `[]` |
| IT-21 | Consultar histórico com placa ausente ou inválida | HTTP 422; `{"erro":"placa_invalida"}` |
| IT-22 | Encerrar ou cancelar um bilhete e abrir outro para a mesma placa | Nova abertura recebe HTTP 201 |

### 4.4. Relatório diário — UC4

| ID | Preparação e ação | Resultado esperado |
|---|---|---|
| IT-23 | Encerrar dois bilhetes no dia 2026-10-05, com durações de 20 e 21 minutos | HTTP 200; `total_bilhetes: 2`, `faturamento_centavos: 400`, `tempo_medio_minutos: 21` |
| IT-24 | GET `/relatorios/diario` sem parâmetro `data` | HTTP 422; `{"erro":"data_invalida"}` |
| IT-25 | Consultar com data `05-10-2026`, `2026-13-05` ou formato inválido | HTTP 422; `{"erro":"data_invalida"}` |
| IT-26 | Encerrar um bilhete após a meia-noite no fuso `-03:00` | O tempo médio considera o dia local do encerramento, não o da abertura |

O relatório deve retornar os campos `data`, `total_bilhetes`, `faturamento_centavos` e `tempo_medio_minutos`, utilizando os tipos definidos no contrato.

## 5. Testes de integridade e concorrência

| ID | Preparação | Ação | Resultado esperado |
|---|---|---|---|
| CT-01 | Placa sem bilhete aberto | Duas aberturas simultâneas | Uma resposta 201 e outra 409; somente um bilhete aberto |
| CT-02 | Bilhete aberto | Duas solicitações de encerramento simultâneas | Uma resposta 200 e outra 409; cobrança registrada uma vez |
| CT-03 | Bilhete aberto | Encerramento e cancelamento concorrentes | Apenas uma mudança de estado é efetivada; estado final consistente |
| CT-04 | Banco temporário com histórico | Consultar histórico após mudanças de estado | Registros preservados, sem duplicação ou perda de dados |

## 6. Validações gerais

Todos os testes de integração devem verificar, quando aplicável:

- Os métodos e caminhos HTTP exatamente como definidos no contrato.
- Os nomes dos campos, sem substituição por nomes alternativos.
- Os valores monetários como inteiros em centavos.
- As datas de resposta em ISO-8601 com fuso `-03:00`.
- As respostas de erro com o campo `erro` e o identificador exato.
- A precedência da validação de formato sobre conflitos de estado.
- A ausência de stack traces e detalhes internos do banco nas respostas.

## 7. Verificações de execução

| ID | Verificação | Resultado esperado |
|---|---|---|
| EX-01 | Instalação pelas dependências declaradas | Instalação reproduzível |
| EX-02 | Geração do Prisma Client e aplicação das migrations | Banco preparado corretamente |
| EX-03 | Compilação TypeScript | Build concluído sem erros |
| EX-04 | Execução dos testes automatizados | Suíte executada e resultados reportados |
| EX-05 | Construção e inicialização do Containerfile | Aplicação inicializa sem configuração secreta obrigatória |
| EX-06 | Acesso pela porta pública 8001 | Endpoints acessíveis em `http://localhost:8001` |
| EX-07 | Verificação de higiene do repositório | Ausência de segredos, banco local e `node_modules` versionados |

## 8. Pontos de atenção

Os seguintes comportamentos não estão integralmente determinados pelo contrato oficial e precisam ser documentados como convenções, sem serem confundidos com exigências expressas:

- Arredondamento do campo `minutos` quando a duração contém segundos.
- Tratamento de saída anterior ao horário de entrada.
- Abrangência de `total_bilhetes` e `faturamento_centavos` em relatórios com registros de estados diferentes.
- Resultado de um relatório sem bilhetes encerrados.

Até a definição dessas convenções, não criar testes com expectativas arbitrárias para esses casos.

## 9. Critério de conclusão

A implementação será considerada verificada quando os testes aplicáveis estiverem aprovados, o build e o container funcionarem e os comportamentos dos UC1–UC8 respeitarem o contrato.

As falhas identificadas deverão ser corrigidas e verificadas novamente. Falhas remanescentes devem constar no resumo final da implementação.

Não substituir as verificações funcionais por testes que apenas confirmem a existência de arquivos ou funções.
