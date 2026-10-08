# Plano de Tarefas — Zona Azul Digital

## 1. Objetivo

Orientar a implementação completa da API Zona Azul Digital, seguindo os requisitos de `spec.md`, as regras de `constitution.md`, a arquitetura de `plan.md` e os critérios de verificação de `tests.md`.

A implementação deverá ser modular, funcional, testável e executável em container.

## 2. Instruções gerais de execução

Antes de iniciar:

1. Ler integralmente `constitution.md`, `spec.md`, `plan.md`, `tests.md`, `contrato.json` e `variante/params.json`.
2. Confirmar os parâmetros: tarifa 400 centavos, fração 30 minutos, teto 5000 centavos, tolerância 10 minutos e porta pública 8001.
3. Respeitar integralmente rotas, respostas, tipos, status HTTP e regras contratuais.
4. Utilizar a stack e a arquitetura definidas no `plan.md`.
5. Executar as tarefas na ordem de dependência.
6. Não alterar arquivos oficiais da prova, nem `scripts/`, `.github/` e `docs/`.
7. Não adicionar funcionalidades, endpoints ou dependências desnecessárias.

Em caso de divergência, priorizar `contrato.json` sobre exemplos ilustrativos e especificações contraditórias.

## 3. Tarefas de implementação

### T01 — Preparar o projeto

**Dependências:** nenhuma.

**Atividades:**

- Criar a estrutura NestJS com TypeScript.
- Configurar Node.js, npm, Prisma e SQLite.
- Declarar e fixar dependências compatíveis.
- Preparar `package.json`, lockfile, TypeScript e configurações necessárias.
- Organizar os módulos conforme `plan.md`.
- Configurar a inicialização da API e o tratamento global de erros.
- Manter as configurações reproduzíveis.

**Conclusão:** projeto inicial compila, a aplicação inicia e a estrutura corresponde ao plano técnico.

### T02 — Configurar persistência e modelo de dados

**Dependência:** T01.

**Requisitos:** RF-01, RF-02, RF-05, RF-08.

**Atividades:**

- Configurar a conexão Prisma com SQLite.
- Criar o modelo de Bilhete definido em `spec.md`.
- Definir identificadores únicos e estados válidos.
- Preparar as migrations.
- Garantir persistência de datas, estados e valores.
- Criar restrição de unicidade para placas com bilhete aberto.
- Prever operações atômicas e proteção contra concorrência.

**Testes relacionados:** UT-13 a UT-18, CT-01 a CT-04.

**Conclusão:** o banco pode ser inicializado, os registros são persistidos corretamente e a exclusividade de placa aberta é protegida.

### T03 — Implementar abertura de bilhetes

**Dependências:** T01, T02.

**Requisitos:** RF-01, RF-08, AC-01, AC-08.

**Atividades:**

- Implementar `POST /bilhetes`.
- Validar a placa obrigatória.
- Aceitar `entrada` opcional no formato contratual.
- Utilizar o relógio atual quando `entrada` estiver ausente.
- Impedir abertura duplicada para a mesma placa.
- Retornar HTTP 201 com os campos exigidos.
- Retornar erros 422 e 409 conforme o contrato.

**Testes relacionados:** UT-13, UT-14, IT-01 a IT-07, CT-01.

**Conclusão:** bilhetes válidos são criados, entradas inválidas são rejeitadas e uma placa não pode ter dois bilhetes abertos.

### T04 — Implementar cálculo de cobrança e encerramento

**Dependências:** T02, T03.

**Requisitos:** RF-02, RN-01, RN-02, AC-02, AC-07.

**Atividades:**

- Implementar `CobrancaService` com cálculo independente da persistência.
- Calcular a duração entre entrada e saída.
- Aplicar tolerância gratuita de 10 minutos.
- Arredondar frações de 30 minutos para cima.
- Cobrar 200 centavos por fração.
- Aplicar o teto máximo de 5000 centavos.
- Garantir resultado monetário inteiro.
- Implementar `POST /bilhetes/{id}/encerramento`.
- Validar existência e estado do bilhete.
- Persistir a saída, duração, cobrança e mudança de estado atomicamente.
- Retornar HTTP 200 e erros contratuais.

**Testes relacionados:** UT-01 a UT-12, UT-15, UT-17, UT-18, IT-08 a IT-10, IT-15, CT-02 e CT-03.

**Conclusão:** todos os cálculos e limites são respeitados, sem cobranças duplicadas ou estados inconsistentes.

### T05 — Implementar cancelamento

**Dependências:** T02, T03.

**Requisitos:** RF-05, AC-05.

**Atividades:**

- Implementar `POST /bilhetes/{id}/cancelamento`.
- Permitir cancelamento apenas de bilhetes abertos.
- Alterar o estado para `cancelado` atomicamente.
- Não gerar horário de saída nem cobrança.
- Liberar a placa para novas aberturas.
- Preservar o registro no histórico.
- Retornar HTTP 200, 404 ou 409 conforme o contrato.

**Testes relacionados:** UT-16, IT-11 a IT-15, IT-22, CT-03.

**Conclusão:** cancelamentos válidos não geram cobrança, preservam o histórico e impedem alterações de estados finais.

### T06 — Implementar consultas e histórico

**Dependências:** T02, T03, T04, T05.

**Requisitos:** RF-03, RF-06, AC-03, AC-06.

**Atividades:**

- Implementar `GET /bilhetes/ativos`.
- Retornar somente bilhetes abertos.
- Implementar `GET /bilhetes?placa=ABC1D23`.
- Validar a placa da consulta.
- Retornar o histórico completo, incluindo todos os estados.
- Ordenar registros do mais recente para o mais antigo.
- Retornar arrays vazios quando não houver resultados.

**Testes relacionados:** IT-16 a IT-22, CT-04.

**Conclusão:** consultas apresentam dados corretos, ordenados e sem registros indevidos.

### T07 — Implementar relatório diário

**Dependências:** T02, T04.

**Requisitos:** RF-04, AC-04.

**Atividades:**

- Implementar `GET /relatorios/diario`.
- Validar o parâmetro obrigatório `data`.
- Utilizar a data civil no fuso `-03:00`.
- Calcular `total_bilhetes`, `faturamento_centavos` e `tempo_medio_minutos`.
- Considerar somente bilhetes encerrados no dia para o tempo médio.
- Arredondar o tempo médio ao inteiro mais próximo, com 0,5 para cima.
- Retornar os quatro campos obrigatórios.
- Retornar HTTP 422 para datas inválidas.

**Testes relacionados:** UT-19, IT-23 a IT-26.

**Conclusão:** o relatório apresenta indicadores consistentes, com tipos e arredondamentos corretos.

### T08 — Implementar testes e qualidade

**Dependências:** T03 a T07.

**Requisitos:** UC1–UC8 e AC-G01 a AC-G08.

**Atividades:**

- Configurar Vitest e Supertest.
- Criar testes unitários com Prisma simulado.
- Criar testes HTTP com banco SQLite isolado.
- Utilizar relógio controlável.
- Implementar os cenários descritos em `tests.md`.
- Testar limites exatos, valores adjacentes e conflitos.
- Verificar integridade em requisições concorrentes.
- Validar erros JSON, formatos e códigos HTTP.
- Executar verificações de tipagem, compilação e qualidade do código.

**Testes relacionados:** UT-01 a UT-19, IT-01 a IT-26, CT-01 a CT-04.

**Conclusão:** suíte automatizada executável, com resultados verificáveis e sem dependência de banco externo ou tempo real.

### T09 — Preparar container e documentação

**Dependências:** T01 a T08.

**Requisitos:** AC-G08 e qualidade SDLC.

**Atividades:**

- Criar `Containerfile` funcional.
- Declarar instalação de dependências e preparação do Prisma.
- Aplicar migrations necessárias antes da inicialização.
- Configurar porta interna 8080 e acesso público pela porta 8001.
- Garantir funcionamento sem variáveis secretas obrigatórias.
- Preparar `.dockerignore`, `.gitignore` e `.env.example`.
- Criar README com instalação, configuração, execução, testes e container.
- Evitar arquivos temporários, segredos e dependências instaladas no repositório.

**Testes relacionados:** EX-01 a EX-07.

**Conclusão:** aplicação inicializa em container, responde pela porta esperada e possui documentação reproduzível.

### T10 — Validar e corrigir a aplicação

**Dependências:** T01 a T09.

**Atividades:**

1. Executar instalação, geração do Prisma Client e migrations.
2. Executar compilação, testes e verificações de qualidade.
3. Construir e iniciar o container.
4. Testar os endpoints com a variante configurada.
5. Comparar as respostas com `contrato.json`, `spec.md` e `tests.md`.
6. Identificar falhas e corrigir suas causas.
7. Reexecutar os testes afetados e a suíte completa.
8. Verificar que as correções não introduziram regressões.

**Conclusão:** aplicação funcional e verificações aprovadas, ou pendências explicitamente identificadas.

## 4. Regras para correções

Durante as verificações:

- Priorizar erros que impedem inicialização, compilação ou execução.
- Corrigir primeiro incompatibilidades com o contrato REST.
- Em seguida, corrigir regras de negócio, limites e inconsistências de estado.
- Corrigir falhas sem reestruturar módulos que já funcionam corretamente.
- Não alterar os resultados esperados dos testes apenas para aprovar uma implementação incorreta.
- Não remover testes para ocultar falhas.
- Preservar a compatibilidade com os documentos de especificação.

Divergências ou comportamentos não definidos expressamente pelo contrato devem ser identificados e tratados como decisões adicionais, sem modificar as exigências oficiais.

## 5. Economia de contexto

Durante a implementação e as verificações:

- Não reproduzir arquivos inteiros sem necessidade.
- Não apresentar logs extensos quando um resumo do erro for suficiente.
- Informar apenas os trechos relevantes de mensagens de falha.
- Evitar repetir requisitos já definidos nos documentos.
- Trabalhar nas tarefas pendentes sem refazer etapas concluídas corretamente.
- Manter as explicações compactas, preservando os dados necessários para diagnosticar problemas.

A economia de contexto não deve impedir verificações ou ocultar falhas.

## 6. Critérios finais de entrega

A aplicação deve possuir:

- Todos os endpoints e comportamentos UC1–UC8.
- Estrutura modular NestJS.
- Persistência SQLite com Prisma.
- Validação e tratamento de erros contratuais.
- Cálculos corretos e integridade dos estados.
- Testes unitários e de integração.
- Dependências declaradas e lockfile.
- Container funcional.
- README com instruções reproduzíveis.
- Ausência de segredos e artefatos indevidos.

## 7. Resumo final obrigatório

Ao concluir, apresentar um relatório compacto contendo:

| Item | Informação esperada |
|---|---|
| Implementação | Funcionalidades e módulos concluídos |
| Verificações | Compilação, testes e container |
| Resultado | Quantidade de testes aprovados e falhos |
| Correções | Principais falhas encontradas e resolvidas |
| Pendências | Problemas restantes, se existirem |
| Execução | Como inicializar e acessar a API |

Não declarar testes aprovados ou execução bem-sucedida sem realmente realizar as verificações.
