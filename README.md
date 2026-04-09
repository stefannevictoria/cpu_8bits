# 🖥️ CPU 8 bits 

Este repositório contém a implementação de uma **CPU de arquitetura simplificada de 8 bits**, desenvolvida como atividade ponderada da disciplina de Computação.

A CPU integra uma ALU completa com todas as operações aritméticas clássicas, um circuito de controle com Program Counter, memórias ROM para instruções e operandos, e um Instruction Register para separar o ciclo de busca do ciclo de execução. Tudo implementado na ferramenta de simulação de circuitos digitais **Digital**.

---

## 🎥 Vídeo de apresentação

[![Vídeo da ponderada](https://img.youtube.com/vi/443v5LUBIkY/0.jpg)](https://www.youtube.com/watch?v=443v5LUBIkY)

---

## 📁 Estrutura dos arquivos

```
📁 CPU/
├── cpu_simples.dig          ← CPU com fetch e execute no mesmo clock
├── cpu_com_ir.dig           ← CPU com Instruction Register (fetch separado)
├── ALU.dig                  ← ALU completa com todas as operações
├── divisor_8bits.dig        ← divisor Restoring Division
├── multiplicador_8bits.dig  ← multiplicador array (produtos parciais)
├── shift_left.dig           ← shift lógico para a esquerda
├── shift_right.dig          ← shift lógico para a direita
├── somador_1bit.dig         ← Full Adder de 1 bit (bloco base)
├── somador_8bits.dig        ← somador de 8 bits (8x Full Adder encadeados)
├── somador_16bits.dig       ← somador de 16 bits (usado na multiplicação)
├── subtrator_8bits.dig      ← subtrator usando complemento de 2
├── register_8bits.dig       ← registrador de 8 bits com flip-flops D
└── README.md
```

---

## 📐 Arquitetura

A CPU é composta por três grandes blocos: a **ALU**, o **circuito de controle** e as **memórias**.

![CPU](https://res.cloudinary.com/dwewomj84/image/upload/v1775706784/Screenshot_2026-04-08_220558_ohdesk.png)

---

## ⚙️ ALU — Unidade Lógica e Aritmética

A ALU recebe dois operandos: o acumulador **AC** e o valor **N** vindo da memória, e produz resultados nos registradores **AC** e **MQ**. Todas as operações rodam em paralelo e um MUX de 6 entradas seleciona o resultado correto conforme o **SEL** (3 bits).

| SEL | Operação      | Entrada  | Saída                       |
|-----|---------------|----------|-----------------------------|
| 0   | Soma          | AC + N   | AC (8 bits)                 |
| 1   | Subtração     | AC - N   | AC (8 bits)                 |
| 2   | Multiplicação | AC × N   | AC (8 LSB) e MQ (8 MSB)     |
| 3   | Divisão       | AC ÷ N   | AC (Resto) e MQ (Quociente) |
| 4   | Shift Left    | AC       | AC (8 bits)                 |
| 5   | Shift Right   | AC       | AC (8 bits)                 |

![ALU](https://res.cloudinary.com/dwewomj84/image/upload/v1775706964/Screenshot_2026-04-09_005527_pzoea6.png)

#### Soma
Implementada a partir do zero com um **Full Adder de 1 bit**. Recebe A, B e Cin, produz S (XOR dos três) e Co (gerado quando pelo menos dois bits de entrada são 1). O **somador de 8 bits** encadeia 8 Full Adders, onde o Co de cada um entra como Cin do próximo.

#### Subtração
Reutiliza o somador com **complemento de 2**. Para calcular AC - N, o circuito faz AC + (~N) + 1: os bits de N são invertidos por portas XOR e o sinal SUB é conectado no Cin do primeiro somador, fornecendo o +1 sem circuito extra.

#### Multiplicação
Implementada com **array multiplier**. Para cada bit de N, calcula-se um Produto Parcial (PP) com 8 portas AND com os bits de AC. Os PPs são somados em cascata com somadores de 16 bits. O resultado de 16 bits é dividido: bits 0–7 vão para AC e bits 8–15 vão para MQ.

#### Divisão
Implementada com o algoritmo **Restoring Division**. A cada passo: o parcial sofre shift left e recebe o próximo bit do dividendo; o divisor é subtraído; se positivo o bit do quociente é 1 e o resultado é mantido; se negativo o bit é 0 e o parcial é restaurado. São encadeados 8 instâncias — uma por bit. O resto final vai para AC e o quociente para MQ.

#### Shift Left

Implementado pela conexão direta de fios: cada bit vai para a posição imediatamente acima, o bit 0 recebe constante 0 (inserido) e o bit 7 é descartado. Equivale a multiplicar o valor por 2.

#### Shift Right

Implementado pela conexão direta de fios: cada bit vai para a posição imediatamente abaixo, o bit 7 recebe constante 0 (inserido) e o bit 0 é descartado. Equivale a dividir o valor por 2.

#### Registradores AC e MQ

Implementados com 8 flip-flops D em paralelo, todos compartilhando o mesmo clock. A saída Q mantém o valor anterior até um pulso de clock, quando captura o valor atual da entrada D. O resultado de AC realimenta a entrada AC_in da ALU, formando o acumulador.

---

## 🔄 Circuito de Controle

O circuito de controle gerencia o fluxo de execução da CPU, coordenando busca e execução das instruções.


### Program Counter (PC)
Um **contador** que incrementa automaticamente a cada pulso de clock, apontando para o próximo endereço de instrução. O enable está fixo em 1 (conta sempre) e o clear em 0 (nunca reseta).

### Endereçamento por Pilha
O endereçamento é **sequencial**, o próximo operando está sempre no endereço imediatamente após o anterior. O PC incrementa sozinho a cada clock, sem necessidade de endereço explícito na palavra de comando.

### Memórias ROM

Duas ROMs compartilham o mesmo endereço vindo do PC:

- **ROM de Operações:** armazena o opcode de cada instrução (3 bits, SEL de 0 a 5)
- **ROM de Operandos:** armazena o valor N correspondente a cada instrução

### Palavra de Comando

Com 6 operações distintas, o opcode usa **3 bits**. Não há campo de endereço na palavra de comando pois o endereçamento é por pilha.

| Endereço | Opcode (SEL) | Operação      |
|----------|--------------|---------------|
| 0        | 0            | Soma          |
| 1        | 1            | Subtração     |
| 2        | 2            | Multiplicação |
| 3        | 3            | Divisão       |
| 4        | 4            | Shift Left    |
| 5        | 5            | Shift Right   |

### Instruction Register (IR)

Registrador de **3 bits** que guarda o opcode buscado na ROM antes de enviá-lo à ALU. Separa formalmente o ciclo de fetch do ciclo de execute: no clock N o IR captura o opcode; no clock N+1 a ALU executa.

---

## 📦 Versões do projeto

### `cpu_simples.dig` — CPU sem IR
Fetch e execute acontecem no **mesmo clock**. Mais simples e com resultados diretos e previsíveis. Boa para entender o funcionamento geral da CPU.

```
CLK → Counter → ROMs → ALU → AC/MQ
```

![CPU simples](https://res.cloudinary.com/dwewomj84/image/upload/v1775707215/Screenshot_2026-04-08_211056_tt7tac.png)

### `cpu_com_ir.dig` — CPU com Instruction Register
Implementação mais completa com **IR separando fetch de execute**. O opcode é buscado na ROM em um ciclo e executado pela ALU no ciclo seguinte, adicionando um delay de 1 clock. Arquitetura mais próxima de uma CPU real.

```
CLK → Counter → ROMs → IR → ALU → AC/MQ
```

---

## ▶️ Como executar

1. Baixe e instale o **Digital**: [https://github.com/hneemann/Digital/releases](https://github.com/hneemann/Digital/releases)
2. Clone este repositório
3. Abra o arquivo `cpu_simples.dig` ou `cpu_com_ir.dig` no Digital
4. Clique em **Simulação → Iniciar** (ou pressione `F5`)
5. Use o botão de **clock manual** para avançar um ciclo por vez
6. Observe os valores de **AC** e **MQ** mudando a cada instrução executada
7. Para testar operações diferentes, clique duas vezes na **ROM de Operandos** durante a simulação e altere os valores na tabela (cada posição corresponde ao operando N daquela instrução)