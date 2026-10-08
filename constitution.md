# Constitution — Zona Azul Digital

## 1. Objetivo

Estabelecer as regras obrigatórias de desenvolvimento, arquitetura, qualidade e segurança da API Zona Azul Digital.

Estas diretrizes devem ser respeitadas durante toda a implementação, execução, correção e manutenção do projeto.

## 2. Hierarquia e fidelidade ao contrato

- O `contrato.json` e o `ENUNCIADO.md` são as referências oficiais dos requisitos da prova, com prevalência das regras contratuais sobre exemplos ilustrativos inconsistentes.
- O `spec.md` centraliza os requisitos funcionais e critérios de aceite.
- O `plan.md` define a arquitetura e as decisões tecnológicas.
- O `tests.md` define os cenários verificáveis.
- O `tasks.md` organiza a sequência de implementação.
- Em caso de conflito entre a documentação produzida e o contrato oficial, prevalece o contrato.
- Não alterar rotas, métodos HTTP, nomes de campos, tipos, formatos, códigos de status ou regras de negócio estabelecidos.

## 3. Princípios de arquitetura

- Utilizar Node.js, TypeScript, NestJS, Prisma e SQLite conforme `plan.md`.
- Organizar a aplicação em módulos com responsabilidades bem definidas.
- Controllers devem receber requisições e produzir respostas HTTP.
- Services devem executar regras de negócio e coordenar operações.
- Centralizar a persistência por meio do Prisma.
- Isolar os cálculos de cobrança das rotas e da persistência.
- Evitar duplicação de lógica, dependências desnecessárias e abstrações sem finalidade.
- Utilizar nomes claros e consistentes para módulos, classes, funções e variáveis.

## 4. Integridade e regras de negócio

- Utilizar exclusivamente os parâmetros da variante especificados em `spec.md`.
- Representar valores monetários em centavos inteiros nas respostas.
- Não utilizar valores monetários de ponto flutuante nos cálculos da cobrança.
- Respeitar o arredondamento das frações de tempo para cima.
- Aplicar a tolerância gratuita sem descontá-la quando ultrapassada.
- Nunca ultrapassar o teto de cobrança por bilhete.
- Garantir que uma placa possua no máximo um bilhete aberto.
- Impedir transições de estado não permitidas.
- Preservar o histórico de bilhetes encerrados e cancelados.
- Garantir operações atômicas de abertura e mudança de estado, inclusive sob concorrência.

## 5. Validação e tratamento de erros

- Validar as entradas antes de executar operações de negócio.
- Aplicar validações de formato (HTTP 422) antes de verificar conflitos (HTTP 409).
- Respeitar os códigos HTTP e os identificadores de erro estabelecidos em `spec.md`.
- Retornar erros contratuais no formato JSON `{"erro":"codigo_do_erro"}`.
- Não retornar mensagens genéricas do framework no lugar dos erros definidos.
- Não expor stack traces, detalhes internos do banco ou segredos nas respostas.
- Tratar falhas inesperadas sem comprometer a integridade dos registros.

## 6. Datas e horários

- Utilizar ISO-8601 com fuso `-03:00` nas respostas temporais.
- Interpretar corretamente instantes com offset informado.
- Centralizar a obtenção da data e horário atuais em um serviço de relógio substituível nos testes.
- Aplicar o fuso contratual nos relatórios diários.
- Não utilizar esperas reais para verificar cálculos temporais nos testes.

## 7. Persistência e segurança

- Utilizar SQLite conforme a arquitetura definida em `plan.md`.
- Manter registros consistentes e acessíveis entre as operações.
- Não armazenar segredos diretamente no código-fonte.
- Não versionar `.env` com segredos, banco local, `node_modules` ou arquivos temporários.
- Declarar todas as dependências e manter arquivo de lock.
- Não adicionar autenticação ou funcionalidades externas que alterem o contrato sem exigência explícita.

## 8. Qualidade e testes

A aplicação gerada deve possuir testes automatizados que verifiquem:

- Os oito casos de uso UC1–UC8.
- Entradas válidas e inválidas.
- Limites exatos, valores imediatamente anteriores e posteriores.
- Cálculos de frações, tolerância e teto.
- Erros HTTP e formatos de resposta.
- Transições de estado e conflitos.
- Consultas, ordenação e relatórios.
- Concorrência e integridade dos dados.

Os resultados esperados devem seguir `tests.md` e o contrato oficial.

## 9. Execução e reprodutibilidade

- A aplicação deve compilar e executar sem serviços externos desnecessários.
- O serviço deve estar acessível em `http://localhost:8001` na correção.
- Fornecer `Containerfile`, manifesto de dependências, lockfile e README.
- Preparar automaticamente os componentes necessários para iniciar a aplicação conforme `plan.md`.
- Os testes devem utilizar ambiente e dados isolados.
- A aplicação deve disponibilizar instruções reproduzíveis de instalação, execução e testes.

## 10. Verificação e conclusão

Antes de considerar a implementação concluída:

1. Verificar a compatibilidade com `contrato.json`.
2. Executar compilação, testes automatizados e verificações de qualidade.
3. Corrigir falhas identificadas e executar novamente as verificações afetadas.
4. Verificar a construção e execução do container.
5. Confirmar que os resultados respeitam os critérios de aceite.
6. Apresentar um resumo das funcionalidades implementadas, verificações realizadas e pendências existentes.

Não considerar a aplicação concluída quando houver falhas conhecidas sem registrá-las explicitamente.
