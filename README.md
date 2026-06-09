# ⏱️ Ponto Fácil Inteligente

O **Ponto Fácil Inteligente** é uma ferramenta de folha de ponto eletrônica e gestão de banco de horas focada em **privacidade, autonomia e zero dependências externas**. O sistema roda inteiramente no lado do cliente (*client-side*), o que significa que nenhum dado sensível de horários ou rotina sai do navegador do usuário.

Este documento serve como manual arquitetural, especificação de lógica de negócios e guia passo a passo para desenvolvedores e entusiastas que desejam entender, estender ou reproduzir este projeto do zero.

---

## 🏛️ 1. Arquitetura do Projeto

O projeto foi concebido sob a filosofia de **Single-File Application (SFA)**. Todo o ciclo de vida da aplicação — desde a renderização da interface (UI), estilização responsiva, motor de cálculos matemáticos até a persistência de dados — ocorre dentro de um único arquivo `index.html`.

### Vantagens da Arquitetura SFA neste contexto:
* **Zero Configuração / Instalação:** O usuário final ou o desenvolvedor só precisa dar um duplo clique no arquivo ou hospedá-lo em servidores estáticos simples (como GitHub Pages, Vercel ou Netlify).
* **Portabilidade Total:** É fácil realizar backups manuais ou mover a ferramenta de servidor.
* **Segurança por Isolamento:** Sem servidores backend ou APIs de terceiros, mitigando vazamento de dados ou ataques de injeção na rede.

---

## 💻 2. Stack Tecnológica

O sistema utiliza a tríade fundamental da web em suas especificações modernas, sem frameworks pesados:

* **HTML5:** Utilização de tags semânticas e elementos nativos de controle de tempo.
* **CSS3 Moderno:**
    * **Variáveis CSS (`:root`)**: Centralização da paleta de cores.
    * **CSS Grid & Flexbox**: Layouts adaptáveis e fluidos.
    * **Media Queries (`@media`)**: Responsividade para telas de dispositivos móveis.
* **JavaScript (ES6+)**: Manipulação assíncrona do DOM, gerenciamento de estado local via JSON estruturado e manipulação avançada de inteiros para controle de tempo.

---

## 💾 3. Modelagem e Estrutura de Dados

A persistência é mantida no mecanismo de `localStorage` do navegador sob a chave única `pontoFacilDB`.

### Detalhes dos Campos:

* **`settings.modo`**: Define a quantidade de campos ativos na tela (Padrão: 4 pontos, expansível para 6).
* **`settings.cargaDiaria`**: Armazena a jornada padrão de trabalho convertida integralmente em minutos (Ex: 08:00 = 480 minutos).
* **`settings.saldoInicialMinutos`**: Banco de horas acumulado trazido de meses/anos anteriores (pode ser positivo ou negativo).
* **`records`**: Um dicionário (*hash map*) onde a chave é a data no formato ISO (`YYYY-MM-DD`).
* **`records["YYYY-MM-DD"]`**: Um array posicional fixo de strings.
    * **Índices [0] a [5]**: Strings de horários (`HH:MM`) representando Entradas e Saídas ordenadas.
    * **Índice [6]**: Um sub-array opcional contendo objetos de **Ajustes Manuais** pós-processados.

---

## 🧮 4. Motor de Lógica e Regras de Negócio

Trabalhar com tempo em JavaScript utilizando o objeto nativo `Date` pode gerar problemas crônicos de fuso horário (*timezones*) e complexidade desnecessária na manipulação de strings. O sistema resolve isso convertendo todo o fluxo de horários para **Inteiros Baseados em Minutos**.

### 4.1. Aritmética de Tempo Segura (Safe Time Math)
As funções `strParaMinutos(str)` e `minutosParaStr(min)` realizam as conversões de entrada e saída.

* **Conversão para Minutos:** A string "08:30" é dividida pelo caractere `:` em duas partes numéricas. O cálculo aplicado é:
    $$\text{Total em Minutos} = (\text{Horas} \times 60) + \text{Minutos}$$
    No exemplo: $(8 \times 60) + 30 = 510\text{ minutos}$.

* **Conversão para String:** O inteiro $510$ passa por uma divisão inteira e operador de resto:
    $$\text{Horas} = \lfloor 510 / 60 \rfloor = 8$$
    $$\text{Minutos} = 510 \pmod{60} = 30$$
    As saídas são formatadas com preenchimento de zeros à esquerda (`String.prototype.padStart(2, '0')`).

### 4.2. A Regra de Tolerância de 10 Minutos
De acordo com as convenções da CLT brasileira (Art. 58), pequenas variações no registro de ponto que não excedam 10 minutos diários (totais) não devem ser computadas como atraso nem como hora extra.

A função `aplicarTolerancia(totalTrabalhado, cargaHoraria)` traduz essa lógica programaticamente:

```javascript
function aplicarTolerancia(totalTrabalhado, cargaHoraria) {
    if (cargaHoraria === 0) return totalTrabalhado; // Dias livres (FDS) computam tudo
    const diferenca = totalTrabalhado - cargaHoraria;
    if (diferenca >= -10 && diferenca <= 10) return 0; // Neutraliza o saldo do dia
    return diferenca;
}

```

### 4.3. Fluxo de Processamento de Ajustes Manuais

Os ajustes manuais possuem uma regra matemática crucial: eles são aplicados estritamente após o processamento da tolerância.

O algoritmo de cálculo de saldo diário funciona da seguinte forma:

1. Soma-se o tempo real trabalhado nos pares de marcações.
2. Calcula-se o saldo bruto subtraindo a carga horária padrão.
3. Aplica-se a função de tolerância sobre este saldo bruto.
4. Soma-se o valor líquido (positivo ou negativo) dos ajustes manuais ao resultado obtido no passo 3.

---

## 🛠️ 5. Guia Passo a Passo de Desenvolvimento

* **Passo 1: O Esqueleto Semântico e Variáveis de Escopo**
Crie o arquivo HTML e defina no cabeçalho `<style>` as variáveis globais de interface. Isso garante consistência visual e facilita futuras manutenções. Monte containers claros para a seção de registros do dia atual, botões de ação rápidos e a tabela estruturada de histórico.
* **Passo 2: O Mecanismo de Estado Local**
Crie a variável de estado global `appData` no JavaScript. Escreva as funções `carregarDados()` e `salvarDados()`. Toda alteração efetuada nos campos de texto ou configurações deve disparar uma chamada para `salvarDados()`, que grava instantaneamente a string JSON atualizada no `localStorage`.
* **Passo 3: O Mecanismo de Loop do Histórico**
Desenvolva a função `atualizarTabela()`. Ela deve varrer todas as chaves do objeto `appData.records`, ordená-las cronologicamente e realizar o cálculo acumulado (banco de horas) de forma sequencial. Conforme itera por cada dia, a função soma os saldos diários ao saldo inicial de configurações, gerando uma linha dinâmica na tabela para cada data modificada.
* **Passo 4: Polimento Responsivo e Recursos Avançados**
Adicione Media Queries no CSS para gerenciar como os inputs e caixas de resumo se comportam em telas pequenas. Para os recursos de exportação (CSV, JSON, TXT), manipule o ecossistema do navegador criando elementos `<a>` temporários em memória com objetos do tipo `Blob` e acione o método programático `.click()` para disparar o download nativo de arquivos sem necessitar de bibliotecas de terceiros.

---

## 🚀 Como Executar o Projeto Localmente

1. Faça o clone deste repositório:

```bash
   git clone [https://github.com/seu-usuario/seu-repositorio.git](https://github.com/seu-usuario/seu-repositorio.git)

```

2. Navegue até a pasta do projeto.
3. Abra o arquivo `index.html` diretamente em qualquer navegador moderno de sua preferência.

---

## 📄 Licença

Este projeto é fornecido sob os termos da licença MIT. Sinta-se totalmente livre para clonar, modificar, distribuir ou utilizar comercialmente conforme suas necessidades.

```

```
