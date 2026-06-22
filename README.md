# Bancada de Teste de Fim de Linha (Fogões)

Este repositório contém o projeto de software para a **Bancada de Teste de Fim de Linha (EOL) funcional para fogões**. O sistema realiza a validação de parâmetros como estanqueidade, vazão, segurança elétrica e temperatura de forma totalmente simulada dentro do CLP, sem necessidade de conexões físicas de processo para os testes.

## Especificações de Hardware e Rede

### Controlador (CLP)
* **Modelo:** CPU 1212C DC/DC/DC
* **Código de Referência:** 6ES7 212-1AE31-0XB0
* **Versão do Firmware:** V3.0
* **Memória de Trabalho:** 50 KB
* **Endereço IP:** `172.28.4.5`
* **Máscara de Sub-rede:** `255.255.255.0`

### Interface Homem-Máquina (IHM)
* **Modelo:** KTP900 Basic PN (9'' TFT, 800 x 480, Touch e teclas físicas)
* **Endereço IP:** `172.28.4.6`
* **Máscara de Sub-rede:** `255.255.255.0`

## Ambiente de Desenvolvimento
* **Software:** Siemens TIA Portal
* **Idioma do Projeto:** pt-BR (tags, blocos e telas em português)

## Estrutura Geral do Projeto
* **Sequenciador (SCL):** Controla a lógica de passos, transições de estados e construção do plano de execução com base na receita selecionada.
* **Modelos de Simulação (OB30 - 100 ms):** Executa os cálculos de resposta térmica, decaimento de pressão e comportamento elétrico em tempo real.
* **Interface (IHM KTP900):** Telas para operação, acompanhamento dos testes dinâmicos e exibição do relatório de aprovação/retrabalho.