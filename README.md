# T2-Eletrônica - Circuito com Arduino
Trabalho 2 de Eletrônica do Simões

## Tabela de preços

| Componente | Quantidade | Preço unitário | Subtotal |
| :--- | :--- | :--- | :--- |
| Protoboard | 1 | R$ 6,90 | R$ 6,90 |
| Resistor 100 Ω 1/4 W | 1 | R$ 1,20 | R$ 3,60 |
| Resistor 22 Ω 2 W | 3 | Emprestado | R$ 0,00 |
| Resistor 1 kΩ 1/4 W | 1 | Emprestado | R$ 0,00 |
| Resistor 330 kΩ 1/4 W | 1 | Emprestado | R$ 0,00 |
| LED Infravermelho | 1 | R$ 1,20 | R$ 1,20 |
| Receptor Infravermelho | 1 | R$ 1,20 | R$ 1,20 |
| Transistor NPN TIP41C | 1 | Emprestado | R$ 0,00 |
| Kit Jumper Macho/Macho | 1 | R$ 12,80 | R$ 12,80 |
| **Total** | | | **R$ 45,10** |

## Explicação dos Componentes

* **Resistores:** Componentes eletrônicos que dificultam a passagem de corrente elétrica. Em projetos, eles são empregados para atenuar a intensidade da corrente e reduzir o potencial elétrico que flui pelos demais componentes.
* **Transistor:** Este componente eletrônico tem a função de amplificar ou atenuar a intensidade da corrente elétrica em circuitos. No contexto deste projeto, ele é empregado para regular a corrente que flui através do diodo Zener.
* **Receptor IR:** Esse componente tem a função de captar o sinal infravermelho emitido e passar ele para o arduino por meio de um pulso PWM.
* **LED IR:** Esse componente tem a função de emitir sinal infravermelho de acordo com o pulso PWM emitido pelo arduino para controlar a frequencia do sinal.

## Cálculos

![image](https://github.com/user-attachments/assets/d9308c17-a59f-40db-bda9-da1e9720241a)

    
## Foto e Vídeo do Circuito

![image](https://github.com/user-attachments/assets/ae90e956-0f82-4351-b938-cfcb9499f68c)

https://github.com/user-attachments/assets/6d1ea86a-db5b-4c54-b14b-bcedd32fb638

## Código do Arduino

> O sketch completo está dobrado em um `<details>` para manter o README enxuto.  

<details>
<summary>▼ Ver clonador_ir.ino completo</summary>

```arduino
/*  ──────────────────────────────────────────────────────────────
    CLONADOR IR   · Somente NEC (controle do kit)  +  NEC2 (projetor)
    --------------------------------------------------------------
    ▸ Pressione o botão "KEY_LEARN" (código 70) do controle NEC
      • 1ª vez  → entra em modo aprendizagem; aguarda quadro NEC2
      • 2ª vez  → transmite o quadro NEC2 gravado (liga/desliga projetor)
    ▸ Pressione "KEY_RESET" (código 68) para apagar o slot
    -------------------------------------------------------------- */

#include <IRremote.h>

#define IR_RX_PIN    3
#define IR_TX_PIN    9

constexpr uint8_t KEY_LEARN = 70;   // highlight-line
constexpr uint8_t KEY_RESET = 68;   // highlight-line
constexpr int_fast8_t REPEATS = 2;  // 0-3, ajuste conforme necessário  // highlight-line

// ---------------------- slot único p/ o projetor ----------------------
bool     nec2Learned = false;
uint16_t projAddr    = 0;
uint8_t  projCmd     = 0;

// ---------------------- funções utilitárias --------------------------
void logDecode() {
  auto &d = IrReceiver.decodedIRData;
  Serial.print(F("[RX] proto="));
  Serial.print(d.protocol == NEC        ? "NEC" :
               d.protocol == NEC2       ? "NEC2" : "OUTRO");
  Serial.print(F(" addr=0x")); Serial.print(d.address, HEX);
  Serial.print(F(" cmd=0x"));  Serial.print(d.command, HEX);
  Serial.print(F(" flags=0b")); Serial.println(d.flags, BIN);
}

void learnNEC2() {
  Serial.println(F("Aguardando sinal NEC2 do projetor..."));
  IrReceiver.resume();
  while (true) {
    if (!IrReceiver.decode()) continue;
    if (IrReceiver.decodedIRData.protocol == NEC2 &&
        IrReceiver.decodedIRData.command != 0x00) {
      // quadro válido
      projAddr    = IrReceiver.decodedIRData.address;
      projCmd     = IrReceiver.decodedIRData.command;
      nec2Learned = true;
      Serial.print(F("Gravado NEC2 addr=0x")); Serial.print(projAddr, HEX);
      Serial.print(F(" cmd=0x"));              Serial.println(projCmd, HEX);
      IrReceiver.resume();
      return;
    }
    IrReceiver.resume();   // ignora qualquer coisa que não seja NEC2
  }
}

void sendNEC2() {
  if (!nec2Learned) {
    Serial.println(F("Slot vazio — aprenda primeiro."));
    return;
  }
  Serial.print(F("Enviando NEC2 0x")); Serial.print(projAddr, HEX);
  Serial.print(F(" 0x"));              Serial.print(projCmd, HEX);
  Serial.print(F(" rep="));            Serial.println(REPEATS);
  IrSender.sendNEC2(projAddr, projCmd, REPEATS);
}

// ---------------------------------------------------------------------
void setup() {
  Serial.begin(115200);
  IrReceiver.begin(IR_RX_PIN);
  IrSender.begin(IR_TX_PIN);
  Serial.println(F("=== Clonador IR (NEC + NEC2) pronto ==="));
}

void loop() {
  if (!IrReceiver.decode()) return;

  // Ignora tudo que não for NEC (controle do kit) ou NEC2 (projetor)
  if (IrReceiver.decodedIRData.protocol == NEC) {
    uint8_t key = IrReceiver.decodedIRData.command;
    logDecode();                // mostra a tecla pressionada no kit
    switch (key) {
      case KEY_LEARN:
        if (!nec2Learned) learnNEC2();
        else              sendNEC2();
        break;

      case KEY_RESET:
        nec2Learned = false;
        Serial.println(F("Slot apagado."));
        break;
    }
  }
  else if (IrReceiver.decodedIRData.protocol == NEC2) {
    // útil para monitorar o sinal original do projetor, caso queira
    logDecode();
  }

  IrReceiver.resume();
}
```

</details>

## Integrantes
* Clara Santos de Melo
* Davi Nascimento Araujo
* Leonardo Vianna Molina Silva

