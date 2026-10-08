# Especificação Funcional — Zona Azul Digital

## 1. Objetivo e escopo

O projeto tem como objetivo especificar uma API REST para gerenciamento de bilhetes de estacionamento rotativo (Zona Azul Digital).

O sistema deve permitir abrir, encerrar e cancelar bilhetes, listar estacionamentos ativos, consultar o histórico por placa e gerar relatórios diários.

A API deve calcular os valores de estacionamento conforme a duração, respeitando a tarifa definida, as frações de cobrança, a tolerância gratuita e o teto máximo por bilhete.

O sistema deve validar os dados recebidos, garantir a integridade dos registros e respeitar as regras de negócio e transições de estado dos bilhetes.

O escopo contempla exclusivamente a API REST, sem desenvolvimento de interfaces gráficas, aplicativos ou back-office.
