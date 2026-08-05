<div align="right">
  🇺🇸 <a href="README.md">English</a> | 🇧🇷 <strong>Português</strong>
</div>

<br>

# BioMon - Sistema de Aquisição de Dados para Medição de Células de Bateria de Baixa Tensão

> **Resumo:** Sistema dedicado ao monitoramento contínuo, escalável e de baixo custo para aquisição de bioeletricidade na ordem de milivolts, focado em células a combustível microbianas (MFCs).
>
> **Contexto:** Projeto desenvolvido como trabalho da disciplina **Projeto de Sistemas Ubíquos e Embarcados (DEC0021)**, do curso de **Engenharia de Computação da Universidade Federal de Santa Catarina – Campus Araranguá**, sob orientação do **Prof. Dr. Jim Lau**.
>
> Autor: **Lucas Porto Ribeiro**
> 
> Semestre: **2026/1**

---

## 📖 Introdução

As células a combustível microbianas (MFCs) são tecnologias bioeletroquímicas que utilizam microrganismos como biocatalisadores para converter a energia química de substratos orgânicos diretamente em energia elétrica. Aplicações em geração de energia renovável, tratamento de efluentes e biossensores são vastas.

A atividade metabólica desses microrganismos faz com que a tensão gerada oscile ao longo do tempo, o que torna o **monitoramento contínuo** indispensável. O principal desafio é registrar múltiplas tensões de forma simultânea e precisa na escala de milivolts, uma vez que data loggers comerciais de alta precisão são onerosos e multímetros manuais não oferecem escalabilidade.

Este projeto propõe uma arquitetura baseada no microcontrolador ESP32 para medir **12 canais simultâneos**, superando limitações de custo e hardware e automatizando o envio de dados para a nuvem.

---

## 🛠️ Hardware Necessário

Para montar a solução, os componentes foram divididos em duas categorias. A subseção **Hardware Principal** lista os itens essenciais para o processamento de dados e funcionamento lógico do sistema. Já a subseção **Hardware de Integração** abrange os itens complementares necessários para alimentação, montagem e transformação do circuito em um equipamento físico funcional.

### Hardware Principal

* **Microcontrolador ESP32:** O cérebro do projeto. Fornece processamento, interface Wi-Fi e hospeda a página web local para acompanhamento em tempo real.
* **Conversor Analógico-Digital (ADC) ADS1115:** Garante uma resolução de 16 bits. É essencial porque o ADC interno do ESP32 possui comportamento não linear e resolução menor. Sua precisão varia conforme o ganho configurado, indo de 0,0078125 mV por bit (ganho de 16×) até 0,1875 mV por bit (ganho de 2/3×), permitindo adequar a resolução à faixa de tensão medida.
* **2× Multiplexadores CD74HC4067:** Multiplexadores analógicos de 16 canais que atuam como "chaves seletoras". Como o ADS1115 possui canais limitados, os multiplexadores expandem o sistema para os 12 pares (positivo e negativo) necessários.

### Hardware de Integração

* **Bateria de Lítio 18650 Recarregável:** Fornece a tensão autônoma necessária para a alimentação do sistema.
* **Suporte (Case) para Bateria 18650:** Utilizado para acoplar a bateria ao circuito.
* **Módulo TP4056:** Módulo responsável pelo carregamento da bateria de lítio.
* **Módulo Step-Up MT3608:** Regulador de tensão utilizado para elevar a tensão de saída da bateria para 5 V, mantendo-a constante e estável.
* **Botão (Chave Liga/Desliga):** Permite ligar e desligar o equipamento.
* **LED:** Utilizado para sinalizar visualmente o status (ligado/desligado) do sistema.
* **Resistor de 220 Ω:** Para limitar a corrente elétrica direcionada ao LED.
* **8× Conectores KRE (3 terminais):** Servem como portas de conexão seguras dos cabos provenientes dos 24 béqueres biológicos.
* **Conectores JST-XH (Macho e Fêmea):** Facilitam a conexão da alimentação na placa principal.
* **Barras de Pinos Fêmea (Cabeçotes):** Utilizadas para acoplar os componentes à placa. Quantidades utilizadas: 2 fileiras de 16 pinos, 2 fileiras de 15 pinos, 1 fileira de 10 pinos e 2 fileiras de 8 pinos.
* **Placa de Cobre:** Base para acomodação dos componentes e roteamento das trilhas elétricas.

---

## 🔌 Esquema de Conexão

A aquisição é baseada na leitura diferencial em pares. A conexão lógica se dá da seguinte forma:

1. As saídas de sinal dos béqueres entram diretamente nos pinos de entrada dos dois **CD74HC4067** — um gerencia a polaridade positiva e o outro a negativa.
2. O **ESP32** comanda a seleção de canal através dos pinos `S0`, `S1`, `S2` e `S3`, mapeados para os GPIOs 32, 33, 25 e 26, de modo síncrono nos dois multiplexadores.
3. As saídas comuns dos multiplexadores são conectadas ao GND e ao pino `A0` do **ADS1115**.
4. O **ADS1115** realiza a conversão com alta resolução e envia os dados digitais de volta ao **ESP32** utilizando o protocolo **I2C** (SDA e SCL).

### Modelagem do Circuito

Abaixo, é possível visualizar o diagrama esquemático do circuito, projetado e montado utilizando a ferramenta EasyEDA, detalhando as ligações elétricas entre os componentes:

<p align="center">
  <img src="images/circuit.svg" alt="Circuito montado no EasyEDA" width="80%">
</p>

Com base no esquemático, o design da placa de circuito impresso (PCB) foi elaborado. A seguir, apresentamos o planejamento das trilhas elétricas (roteamento) lado a lado com a renderização 3D da PCB desenvolvida, mostrando a disposição física final dos conectores e módulos:

<p align="center">
  <img src="images/traces.png" alt="Trilhas da PCB no EasyEDA" width="48%">
  <img src="images/pcb_3d.png" alt="Imagem da PCB 3D" width="48%">
</p>

---

## ⚙️ Software

O software foi desenvolvido em C++. Seu fluxo de funcionamento, desde a leitura dos sinais até a transmissão dos dados, é apresentado no diagrama de blocos a seguir.

```mermaid
flowchart LR

    subgraph Entradas
        A1["12× Sinal -"] --> MUXN["CD74HC4067"]
        A2["Seleção de Canal<br>(ESP32)"] --> MUXN

        B1["12× Sinal +"] --> MUXP["CD74HC4067"]
        B2["Seleção de Canal<br>(ESP32)"] --> MUXP
    end

    MUXN --> ADS["ADS1115"]
    MUXP --> ADS

    ADS -- "I²C" --> ESP["ESP32"]

    ESP -- "Wi-Fi" --> GS["Google Sheets"]

    ESP -- "Ponto de Acesso" --> WEB["Interface Web"]
```

### Bibliotecas e Ferramentas Necessárias

* Placa ESP32 instalada na Arduino IDE.
* Biblioteca `WiFi.h` (nativa).
* Biblioteca `Preferences.h` (nativa).
* Biblioteca `Adafruit_ADS1X15` (comunicação com o ADC).
* Biblioteca `ESP_Google_Sheet_Client` (acesso e autenticação segura com Google Cloud).
* Biblioteca `mongoose_glue.h` (servidor web embarcado — Mongoose Wizard).

### Estrutura do Código

O código principal do projeto está localizado em:

```text
src/
├── biomon_wizard/
│   └── biomon_wizard.ino
│
└── mongoose/
    ├── mongoose.c
    ├── mongoose.h
    ├── mongoose_config.h
    ├── mongoose_fs.c
    ├── mongoose_glue.c
    ├── mongoose_glue.h
    ├── mongoose_impl.c
    └── mongoose_wizard.json
```

O firmware principal está localizado em `src/biomon_wizard/biomon_wizard.ino`, enquanto os arquivos relacionados ao Mongoose estão organizados separadamente em `src/mongoose/`.

---

## 🚀 Instalação

### 1. Clone o repositório

Clone este repositório para o seu computador.

### 2. Abra o projeto

Abra o arquivo:

```text
src/biomon_wizard/biomon_wizard.ino
```

na Arduino IDE.

### 3. Configure o Google Sheets

Preencha as variáveis do Google Sheets:

```text
PROJECT_ID
CLIENT_EMAIL
PRIVATE_KEY
```

utilizando as credenciais da sua Google Service Account.

### 4. Compile e faça o upload

Selecione a placa ESP32 correspondente na Arduino IDE, compile o projeto e faça o upload do firmware para o microcontrolador.

---

## 📋 Instruções de Uso

A imagem a seguir contém o passo a passo e as instruções detalhadas para a correta utilização e configuração inicial do sistema:

<p align="center">
  <img src="images/instructions.png" alt="Instruções para utilização do sistema" width="60%">
</p>

---

## 🚀 Projeto Final

Após a integração de todo o hardware e software, o equipamento foi montado em sua estrutura definitiva. As imagens abaixo apresentam o sistema físico final desenvolvido e pronto para uso:

<p align="center">
  <img src="images/system_3.jpeg" alt="Sistema final físico desenvolvido 3" width="32%">
  <img src="images/system_2.jpeg" alt="Sistema final físico desenvolvido 2" width="32%">
  <img src="images/system_1.jpeg" alt="Sistema final físico desenvolvido 1" width="32%">
</p>

O firmware resultante (`biomon_wizard.ino`) cumpre com os requisitos funcionais estipulados:

* **Modo Access Point & Station:** O ESP32 cria sua própria rede Wi-Fi e simultaneamente tenta conectar à rede local de internet do laboratório para despachar pacotes.
* **Servidor Web Embarcado:** Permite acesso em tempo real via smartphone ou computador para visualizar um multímetro digital virtual, onde é possível configurar os parâmetros de conectividade e o intervalo da amostra.
* **Nuvem:** Processa lotes (*batching*) de amostras e despacha os dados para uma planilha do **Google Sheets** periodicamente, evitando limitações de cota de chamadas da API.
* **Memória:** Utiliza a biblioteca `Preferences` para armazenar persistentemente a rede e as senhas configuradas na interface.

### Interfaces e Visualização de Dados

Para o monitoramento local, o servidor web embarcado fornece uma interface amigável. As capturas de tela a seguir exibem a Interface Web, que atua como um multímetro digital em tempo real e painel de configuração:

<p align="center">
  <img src="images/ui_1.png" alt="Interface web 1" width="78%">
  <img src="images/ui_2.png" alt="Interface web 2" width="78%">
</p>

Os dados coletados são enviados automaticamente para a nuvem. A imagem abaixo mostra a planilha no Google Sheets recebendo as leituras contínuas do sistema:

<p align="center">
  <img src="images/table.png" alt="Planilha no Google Sheets" width="80%">
</p>

Com os dados estruturados na planilha, é possível gerar visualizações dinâmicas. A seguir, os gráficos no Google Sheets ilustrando o comportamento da tensão (bioeletricidade) gerada pelas células ao longo do tempo:

<p align="center">
  <img src="images/graph.png" alt="Gráficos no Google Sheets" width="80%">
</p>
