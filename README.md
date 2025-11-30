## 1. Objetivo

Desenvolver um **sistema ponta-a-ponta** em VHDL capaz de:

1. **Enviar** 64 amostras de 16 bits (armazenadas em ROM) através de uma UART,
2. **Criptografar** cada byte no lado do emissor (Sender) com um *stream cipher* (LFSR + XOR),
3. **Receber, descriptografar** e reconstituir as amostras no lado do receptor (Receiver),
4. **Filtrar** o sinal reconstruído num FIR de 21 taps,
5. Disponibilizar os dados filtrados para gravação/observação (RAM de debug ou DAC externo).

---

## 2. Arquitetura de Alto Nível

```
   ROM -> Encryptor -> UART-Tx  ======  UART-Rx -> Decryptor -> FIR-21t -> RAM/DAC
                     (Sender)           (Receiver)
```

* **Clock único** (50 MHz nas simulações).
* Comunicação **8 N 1**: 1 start, 8 dados, 1 stop.
* **Cipher** simples (XOR com LFSR-8, polinómio x⁸+x⁶+x⁵+x⁴+1, semente 0xB5).
* **Filtro FIR** parametrizado (16 bits, 21 coef.) – reset ativo‐baixo.

---

## 3. Implementação VHDL

### 3.1 Blocos reutilizados

| Bloco              | Função                              | Observações                   |
| ------------------ | ----------------------------------- | ----------------------------- |
| `UART`             | Tx + Rx 8N1                         | Param. `BIT_TICKS`            |
| `Encryptor`        | LFSR + XOR (8 bits)                 | Pulso `start`, saída em 1 clk |
| `Decryptor`        | Algoritmo idêntico (simétrico)      | Usa a mesma chave inicial     |
| `FIR_Filter_21tap` | Filtro digital de 21 coef. (16 bit) | Reset ativo‐baixo             |

### 3.2 Entidade **Sender**

* **ROM** interna (`ROM_DATA`) com 64 amostras de 16 bits.
* **FSM** em oito estados: carrega amostra ➜ divide em bytes ➜ cifra ➜ transmite ➜ incrementa índice.
* Gera sinal `done` ao finalizar o envio.

### 3.3 Entidade **Receiver**

* **UART-Rx** reconstrói cada byte cifrado.
* **FSM** em seis estados: espera 1.º byte ➜ decifra ➜ espera 2.º ➜ decifra ➜ envia ao FIR ➜ armazena resultado.
* Gere `data_out_valid` por amostra e `done` ao processar as 64.

### 3.4 Test-bench `tb_FinalProject`

* Gera clock de 50 MHz e pulso de reset de 50 ns.
* Conecta `tx_out` → `rx_in` (loopback interno).
* Regista cada saída filtrada em `ram_output.txt`.
* Finaliza quando `done_rx='1'`.

---

## 4. Procedimento de Simulação

```tcl
vlib work
set OPTS "-2002 -explicit"
foreach f {UART Encryptor Decryptor FIR_Filter_21tap Sender Receiver tb_FinalProject} {
  vcom $OPTS $f.vhd
}
vsim work.tb_FinalProject
run -all
```

*Transcript esperado*

```
# ** Note: Simulation completed, all samples processed.
```

O ficheiro `ram_output.txt` contém 64 linhas com o resultado filtrado (ex.: 0, 0, -1, -3, …).

---

## 5. Resultados de Simulação

| Métrica                | Valor                          |
| ---------------------- | ------------------------------ |
| Amostras enviadas      | 64                             |
| Bytes UART             | 128                            |
| Latência Sender (byte) | 3 clks (cifra) + transferência |
| Latência FIR           | 20 clks (N-taps - 1)           |
| Erros de simulação     | 0                              |

Forma de onda confirma:

* Sincronismo perfeito entre `tx_start` e `uart_busy`.
* `state_reg` dos dois FSMs evolui como previsto.
* Sequência filtrada em `data_out` corresponde ao sinal ROM convolvido com coeficientes FIR.

---

## 6. Considerações de Síntese

* **ROM** é inferida como **RAM-style=ROM** (Altera/Intel).
* UART, LFSR e FIR utilizam apenas lógicas combinatórias + FF; sem recursos DSP a menos que o FIR seja mapeado para DSP48 (opcional).
* Clock 50 MHz << 100 MHz típico do Cyclone IV, logo sem problemas de *timing*.
* Para demonstração em laboratório, ligar:

  * `Sender.tx_out` ↔ `Receiver.rx_in` (jumper de fio curto)
  * `Receiver.data_out` aos pinos DAC ou pinos GPIO + DAC externo.

---

## 7. Conclusão

O sistema desenvolvido demonstra a integração de **criptografia, comunicação serial** e **processamento digital de sinal** num FPGA, respeitando os requisitos do trabalho final. A arquitetura modular facilita futuras alterações (novo filtro, outro algoritmo de cifra, etc.).
