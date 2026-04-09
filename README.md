# CPU 8 bits

Implementação de uma CPU de arquitetura simplificada de 8 bits, desenvolvida como atividade ponderada do Módulo 5 - Engenharia de Computação (Inteli).

Este projeto implementa uma CPU funcional de 8 bits utilizando o simulador [Digital](https://github.com/hneemann/Digital). A CPU é composta por uma ALU completa com 8 operações aritméticas e lógicas, uma ROM de instruções, um Program Counter, um Instruction Register e registradores de saída AC e MQ.

O objetivo é demonstrar na prática como uma CPU executa instruções em ciclos de busca e execução, acumulando resultados sucessivos entre operações.

## Demonstração

> 📹 [Link do vídeo demonstrativo](https://drive.google.com/file/d/1AgupYC74_2X23gpcvvvOffLgk9-0rDC0/view?usp=sharing)


## Componentes

### ALU (Unidade Lógico-Aritmética)

A ALU recebe três entradas principais:

- **AC** (8 bits) — o acumulador, que guarda o resultado da operação anterior e serve de entrada para a próxima
- **N** (8 bits) — o operando externo, lido da ROM
- **OP** (3 bits) — o seletor de operação, que indica qual das 8 operações será executada

Além dessas, recebe **CK** (clock) e **EN** (enable) para controlar os registradores internos.

Todos os subcircuitos de operação rodam em paralelo o tempo todo — soma, subtração, multiplicação, divisão, shift left, shift right, NAND e XOR estão sempre calculando com os valores atuais de AC e N. O que muda é qual resultado é selecionado na saída.

O **Multiplexador de AC** recebe as 8 saídas e, controlado pelo OP, seleciona qual delas vai para o registrador AC. O **Multiplexador de MQ** faz o mesmo para o registrador MQ, apenas multiplicação e divisão produzem valores úteis nele e as demais entradas ficam em zero.

As saídas dos multiplexadores alimentam os **registradores Reg AC** e **Reg MQ**, que só salvam o resultado quando o clock pulsa e o enable está ativo. A saída Q do Reg AC vira o novo AC, retroalimentando a ALU e isso é o que permite acumular operações sucessivas, como uma calculadora.

#### Operações implementadas

| OP (binário) | Operação | Entradas | Saída AC | Saída MQ |
|---|---|---|---|---|
| 000 | Soma | AC + N | Resultado | — |
| 001 | Subtração | AC - N | Resultado | — |
| 010 | Multiplicação | AC × N | 8 LSBs | 8 MSBs |
| 011 | Divisão | AC ÷ N | Resto | Quociente |
| 100 | Shift Left | AC | AC << 1 | — |
| 101 | Shift Right | AC | AC >> 1 | — |
| 110 | NAND | AC NAND N | Resultado | — |
| 111 | XOR | AC XOR N | Resultado | — |

#### Subcircuitos da ALU

A ALU é construída a partir dos seguintes subcircuitos, todos desenvolvidos do zero:

- `somador_completo.dig` — somador de 1 bit com carry
- `somador_8bits.dig` — 8 somadores de 1 bit em cascata (ripple carry)
- `subtrator_8bits.dig` — somador com complemento de 2 (Cin=1 e inversão de B)
- `somador_16bits.dig` — usado internamente pela multiplicação
- `multiplicador_8bits.dig` — método shift-and-add, resultado de 16 bits dividido em AC (LSB) e MQ (MSB)
- `divisor_8bits.dig` — divisão com resto em AC e quociente em MQ
- `shift_left_8bits.dig` — deslocamento lógico à esquerda via redistribuição de bits
- `shift_right_8bits.dig` — deslocamento lógico à direita via redistribuição de bits
- Porta **NAND** nativa do Digital (8 bits)
- Porta **XOR** nativa do Digital (8 bits)


### CPU

#### Componentes

**Clk (Clock):**
Pulsa na frequência de 1 Hz. Cada pulso é o "batimento" da CPU. No simulador, pode ser acionado manualmente clicando no botão.

**En (Enable):**
Liga e desliga a CPU. Quando EN=1, a CPU executa normalmente. Quando EN=0, ela pausa na última operação executada sem perder o estado atual.

**Clk AND En (CLK_en):**
O clock efetivo da CPU. Só propaga o pulso quando os dois sinais estão ativos simultaneamente, o que garante que nenhum componente avance se o enable estiver desligado.

**PC (Program Counter):**
Contador que guarda o endereço da próxima instrução na ROM. A cada pulso de CLK_en ele incrementa automaticamente, apontando para a linha seguinte, fazendo com que as operações aconteçam em sequência.

**ROM:**
Memória somente leitura onde as instruções estão gravadas. Cada endereço contém um valor de 11 bits: os 3 bits mais significativos são o OP (qual operação) e os 8 bits menos significativos são o operando N. A ordem das instruções na ROM segue a mesma ordem dos canais do multiplexador da ALU.

**IR (Instruction Register):**
Registrador que captura e trava a instrução vinda da ROM enquanto ela é executada. Sem o IR, se o PC avançasse no meio da execução a instrução mudaria antes do término. Após travar, o IR separa os bits: 0–7 vão como N para a ALU e 8–10 vão como OP.

**Flip-flop D:**
Divide o clock em dois ciclos distintos usando suas duas saídas:
- **Q:** saída normal, atrasada 1 ciclo. Sinal de **execute**: quando Q sobe, a ALU executa e os registradores AC e MQ travam o resultado.
- **Q/:** saída invertida, oposta a Q (Q barrado). Sinal de **fetch**: quando Q/ está alto, o PC e o IR estão ativos buscando a próxima instrução da ROM.

Enquanto um está em 1, o outro está em 0. A CPU nunca busca e executa ao mesmo tempo. Esse é o ciclo fetch/execute clássico da arquitetura de von Neumann.

**AC_out:**
Saída do registrador AC, exibe o resultado principal da operação atual (soma, diferença, resto da divisão, resultado do shift, etc.).

**MQ_out:**
Saída do registrador MQ, exibe o resultado secundário, utilizado principalmente na multiplicação (8 bits mais significativos do produto) e na divisão (quociente).


#### Ciclo de funcionamento

A cada pulso de clock, a CPU executa uma operação completa em dois subciclos:

```
Pulso de clock
      │
      ├─ FETCH  (Q/ = 1)
      │    PC aponta para o próximo endereço da ROM
      │    IR trava a instrução (OP + N)
      │
      └─ EXECUTE  (Q = 1)
           ALU calcula com AC atual e N lido do IR
           Resultado é travado nos registradores AC e MQ
           PC incrementa → pronto para o próximo fetch
```


#### Exemplo de execução

A ROM foi configurada com os seguintes valores para demonstrar todas as operações em sequência:

| Ciclo | Operação | N (operando) | Cálculo | AC após | MQ após |
|---|---|---|---|---|---|
| 1 | Soma | 2 | 0 + 2 | **2** | 0 |
| 2 | Subtração | 1 | 2 - 1 | **1** | 0 |
| 3 | Multiplicação | 4 | 1 × 4 | **4** | 0 |
| 4 | Divisão | 3 | 4 ÷ 3 | **1** (resto) | **1** (quociente) |
| 5 | Shift Left | — | 1 << 1 = `00000010` | **2** | — |
| 6 | Shift Right | — | 2 >> 1 = `00000001` | **1** | — |
| 7 | NAND | 238 | 1 NAND 238 = NOT(00000000) | **255** | — |
| 8 | XOR | 252 | 255 XOR 252 = `00000011` | **3** | — |

> **Nota sobre Shift:** deslocar bits para a esquerda equivale a multiplicar por 2; deslocar para a direita equivale a dividir inteira por 2. O número continua sendo representado em 8 bits, o que muda é o valor numérico, não a quantidade de bits.

> **Nota sobre NAND:** NAND é o inverso do AND bit a bit. `00000001 AND 11101110 = 00000000`, e NOT disso = `11111111` = 255.

> **Nota sobre XOR:** XOR retorna 1 onde os bits diferem. `11111111 XOR 11111100 = 00000011` = 3.


## Como executar

1. Instale o [Digital](https://github.com/hneemann/Digital/releases)
2. Clone este repositório
3. Abra o arquivo `cpu_8bits.dig` no Digital
4. Clique em **Simulation → Start**
5. Ative o **Enable (En)**
6. Pressione o botão **Clk** manualmente ou use o clock automático para observar cada operação


## Ferramenta utilizada

[Digital](https://github.com/hneemann/Digital) — simulador de circuitos digitais de código aberto desenvolvido por Helmut Neemann.

## Autora

Sara Sbardelotto