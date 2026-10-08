# Especificação Funcional — Zona Azul Digital

## 1. Objetivo e escopo

O projeto tem como objetivo especificar uma API REST para gerenciamento de bilhetes de estacionamento rotativo (Zona Azul Digital).

O sistema deve permitir abrir, encerrar e cancelar bilhetes, listar estacionamentos ativos, consultar o histórico por placa e gerar relatórios diários.

A API deve calcular os valores de estacionamento conforme a duração, respeitando a tarifa definida, as frações de cobrança, a tolerância gratuita e o teto máximo por bilhete.

O sistema deve validar os dados recebidos, garantir a integridade dos registros e respeitar as regras de negócio e transições de estado dos bilhetes.

O escopo contempla exclusivamente a API REST, sem desenvolvimento de interfaces gráficas, aplicativos ou back-office.

## 2. Parâmetros da variante

Esta seção apresenta os valores obrigatórios da variante, definidos em `variante/params.json`, que devem ser utilizados pela aplicação.

| Parâmetro | Valor | Descrição |
|---|---|---|
| `TARIFA_HORA_CENTAVOS` | 400 | Tarifa de 400 centavos (R$ 4,00) por hora de estacionamento. |
| `FRACAO_MINUTOS` | 30 | Cobrança em frações de 30 minutos, equivalentes a 200 centavos cada. Frações incompletas são arredondadas para cima. |
| `TETO_DIARIO_CENTAVOS` | 5000 | Valor máximo de 5.000 centavos (R$ 50,00) por bilhete. |
| `TOLERANCIA_MINUTOS` | 10 | Permanências de até 10 minutos são gratuitas. Após esse limite, cobra-se o tempo integral desde o primeiro minuto, sem descontar a tolerância. |
| `PORTA_SERVICO` | 8001 | Porta em que a API deve estar acessível para a suíte de correção. |

A aplicação deve respeitar exatamente esses parâmetros em seus cálculos e configurações.

Todos os valores monetários devem ser representados em centavos inteiros, evitando o uso de números de ponto flutuante nas respostas da API.
