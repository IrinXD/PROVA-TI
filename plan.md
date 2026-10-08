# Plano Técnico — Zona Azul Digital

## 1. Objetivo

Definir a arquitetura, tecnologias, responsabilidades e configurações necessárias para implementar a API REST especificada em `spec.md`.

A implementação deve priorizar simplicidade, organização modular, confiabilidade, testabilidade e fidelidade ao contrato.

## 2. Stack tecnológica

| Componente | Tecnologia | Justificativa |
|---|---|---|
| Runtime | Node.js 24 LTS | Ambiente estável para execução da API. |
| Linguagem | TypeScript | Tipagem estática e melhor manutenção. |
| Framework | NestJS 12 | Estrutura modular com controllers, services e injeção de dependências. |
| Servidor HTTP | Express via NestJS | Adaptador padrão, sem configuração de servidor independente. |
| Persistência | SQLite | Banco relacional em arquivo, sem serviço externo. |
| ORM | Prisma 7 | Modelagem, consultas tipadas e migrations. |
| Testes | Vitest, Supertest e @nestjs/testing | Testes unitários e integração HTTP. |
| Dependências | npm | Instalação reproduzível através de package-lock.json. |
| Container | Node.js 24 em imagem Debian slim | Execução reproduzível e compatível com dependências nativas do SQLite. |

Adotar ESM de forma consistente entre Node.js, TypeScript, NestJS e Prisma. Fixar versões compatíveis das dependências e gerar o Prisma Client antes da compilação.

## 3. Estrutura da aplicação

```text
src/
├── main.ts
├── app.module.ts
├── common/relogio.service.ts
├── database/
│   ├── database.module.ts
│   └── prisma.service.ts
└── modules/
    ├── bilhetes/
    │   ├── bilhetes.module.ts
    │   ├── bilhetes.controller.ts
    │   └── bilhetes.service.ts
    ├── cobranca/
    │   ├── cobranca.module.ts
    │   └── cobranca.service.ts
    └── relatorios/
        ├── relatorios.module.ts
        ├── relatorios.controller.ts
        └── relatorios.service.ts
```

Os DTOs de entrada e validação devem ficar próximos aos módulos responsáveis. O serviço de relógio deve ser registrado como provider compartilhado e substituível nos testes.

## 4. Responsabilidades

### 4.1. Inicialização

- `main.ts`: inicializar a aplicação, configurar porta, validação e tratamento de erros.
- `app.module.ts`: registrar os módulos e suas dependências.
- Não criar `app.ts` ou `server.ts` separados.

### 4.2. Bilhetes

**Controller:** expor exatamente as rotas e métodos HTTP de UC1, UC2, UC3, UC5 e UC6.

**Service:** implementar as operações:

| Função conceitual | Responsabilidade |
|---|---|
| `abrirBilhete` | Validar dados, impedir placa duplicada e persistir bilhete aberto. |
| `encerrarBilhete` | Verificar estado, registrar saída, calcular cobrança e encerrar. |
| `cancelarBilhete` | Cancelar somente bilhetes abertos, sem cobrança. |
| `listarAtivos` | Consultar bilhetes abertos, mais recentes primeiro. |
| `historicoPorPlaca` | Consultar todos os registros da placa solicitada. |

### 4.3. Cobrança

O `CobrancaService` implementa `calcularCobranca`, recebendo os instantes de entrada e saída e retornando duração e valor em centavos inteiros.

Deve aplicar integralmente as regras RN-01 e RN-02 do `spec.md`, sem acessar diretamente o banco.

### 4.4. Relatórios

O `RelatoriosController` recebe a requisição UC4.

O `RelatoriosService` implementa `gerarRelatorioDiario`, recebendo a data solicitada e retornando os campos e cálculos estabelecidos no contrato.

### 4.5. Banco de dados

O `PrismaService` centraliza a conexão com o SQLite e disponibiliza acesso aos módulos.

A entidade Bilhete deve contemplar os campos definidos em `spec.md`, preservando as informações após encerramento ou cancelamento.

Garantir atomicidade nas alterações de estado e exclusividade de bilhetes abertos por placa. Utilizar índice único parcial para placas com status aberto e tratar conflitos de persistência como HTTP 409 quando corresponderem à regra UC8.

## 5. Datas e horários

- Centralizar a obtenção do horário atual em um serviço de relógio injetável.
- Permitir substituição do relógio nos testes automatizados.
- Persistir instantes sem perder informações de fuso.
- Serializar datas e horários no formato ISO-8601 com offset `-03:00`.
- Realizar agrupamentos diários considerando o fuso definido no contrato.
- Não depender de esperas reais para testar tolerância, frações e teto.

## 6. Validação e respostas

- Validar formatos antes das regras de negócio.
- Respeitar a precedência 422 antes de 409.
- Personalizar a validação padrão do NestJS para retornar HTTP 422 com os códigos de erro do contrato.
- Não expor mensagens internas, erros do Prisma ou stack traces nas respostas contratuais.
- Configurar explicitamente HTTP 200 nos endpoints de encerramento e cancelamento e HTTP 201 na abertura.

## 7. Configuração e persistência

Utilizar `DATABASE_URL` configurável, com padrão local funcional sem necessidade de segredo externo.

O SQLite deve persistir os registros em arquivo, com diretório gravável no container e possibilidade de volume persistente.

A porta interna documentada no contrato é 8080, enquanto a porta pública da variante é 8001. Configurar a execução em container com mapeamento 8001:8080 e permitir configuração explícita da porta de escuta para execução fora do container.

Documentar ambas as formas de inicialização no README.

## 8. Arquivos adicionais da aplicação

A geração deverá incluir:

- `package.json` e `package-lock.json`
- `tsconfig.json` e configurações necessárias do NestJS
- `prisma/schema.prisma` e migrations
- Configuração do Prisma 7 e Prisma Client gerado no processo de build
- `Containerfile` e `.dockerignore`
- `.gitignore` e `.env.example`
- `README.md`
- Testes automatizados unitários e HTTP

Nunca versionar o banco local, arquivos `.env` com segredos, dependências instaladas ou artefatos temporários.

## 9. Execução e verificações

O processo de geração deve:

1. Instalar dependências declaradas de modo reproduzível.
2. Preparar o banco, aplicar migrations e gerar o Prisma Client.
3. Compilar a aplicação TypeScript.
4. Executar testes unitários e testes HTTP com dados isolados.
5. Verificar os comportamentos e erros definidos em `spec.md` e `tests.md`.
6. Construir e verificar o container.
7. Documentar instruções de execução e possíveis pendências.

Evitar dependências externas e funcionalidades não solicitadas pelo contrato.
