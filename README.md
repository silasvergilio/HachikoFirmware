# HachikoFirmware
Repositório para ensinar novos e antigos competidores de Sumô 3kg (Auto) a usar o código usado no robô Hachiko da equipe FEG-Robótica e adaptá-lo para seu próprio robô.

## Sumário

1. [Objetivo](#objetivo)
2. [Hardware](#hardware)
    1. [Placa Hachiko](#placaHachiko)
    2. [Mecânica Hachiko](#mecHachiko)

#### Objetivo <a name="objetivo"></a>
O principal objetivo deste repostório é capactar colegas programadores a usar este código como inspiração para seus códigos, tanto em placas semelhantes a que eu usei, quanto em placas diferentes como Arduino ou Arm, portanto tento nesta descrição descrever tanto do ponto de vista do hardware que eu estou manipulando para alcançar meus objetivos, bem como também descrever a lógica por trás do código por meio de imagens e explicações textuais. 

<a name="hardware"></a>
#### Hardware 

Esta seção se dedica a explicar o hardware que foi usado neste projeto, uma vez que ele irá definir diversas decisões que foram tomadas ao longo deste projeto. Uma descrição mais precisa pode ser obtida acessando a documentação específica do hardware. Uso este momento também para lembrar que é possível acessar neste [link](https://github.com/silasvergilio/MetalGarurumon-Firmware) o repositório do robô que antecedeu este, muitos aspectos daquele projeto foram usados aqui e corrigidos aqui também, porém a sua formatação é em um arquivo pdf, espero com este repositório criar algo que ajude a criar um guia mais reproduzível. 

<a name="placaHachiko"></a>
##### Placa Hachiko 

<img src="./images/img01.jpg" heigth = "600" width = "600">

Após uma série de melhorias desde a **Placa Lobo**, a placa Hachiko traz mais portas para sensores de distância. Pinos de interrupção externa em dois sensores de linha, mas mantém suas outras qualidades e características já bem conhecidas. Abaixo segue as principais que irão influenciar no nosso código. 

| Característica  | Valor |
| ------------- | ------------- |
| Sensores de distância  | 6  |
| Sensores de Linha  | 4  |
| Sensores de Linha com Interrupção Externa  | 4  |
| Acionamento  | Bluetooth  |
| Tensão de Entrada  | 24V - 36V |

<a name="mecHachiko"></a>
##### Mecânica Hachiko 

<img src="./images/img02.jpg" heigth = "600" width = "600">

Por mais que em geral se veja o programador como alguém num mundo abstrato, para o sumô de robôs é preciso uma boa interdisciplinaridade, o nosso código deve conversar com as caraterísticas físicas do nosso robô, tanto do ponto de vista mecânica quanto do ponto de vista da eletrônica, por mais que o conhecimento necessários nestas áreas não seja profundo, quanto mais profundo melhor, segue abaixo algumas das principais características, seguidas de uma conclusão.

| Característica  | Valor |
| ------------- | ------------- |
| Motores  | Maxon RE40  |
| Velocidade Máxima Teórica | 6m/s  |
| Relação de transmissão de engrenagens  | 4.9 : 1  |
| Dimensões | 196x196x90mm |
| Ângulo da rampa | 18º |

Conclusão. **Este robô é __muito__ rápido**. Todas as ideias aplicadas aqui levam este fato em consideração. 
> Caso isso não fosse verdade, mudaria muito no programa ? 

Não **muito**. Todavia algumas decises na parte de estratégias mudariam. Mas não vamos colocar a carroça na frente dos bois. Vamos seguir com nossas primeiras linhas de código.

#### Software

A linguagem de programação usada neste projeto foi a linguagem C, todavia ao se programar PIC é natural que exista diferentes compiladores, um muito comum seria usar o MPLab, todavia para este projeto foi usado o compilador CCS. Ele tem uma linguagem simples de usar que não exige grande conhecimento sobre manipulação de bits e uso de registradores específicos.

##### Configuração

Toda a seção de configuração do hardware foi feita em uma única função da biblioteca *hachiko_reference.c*, a primeira parte que entraremos em detalhe é o *set_tris*, essa é uma função que define o sentido de cada porta, ou seja, se determinado pino de cada porta é **entrada** ou **saída**, cabe ao programador em conjunto com o seu grupo da eletrônica analisar qual a função de cada porta da sua placa. 

Exemplos de dispositivos de **entrada**:

1. Sensor de Distância
2. Sensor de Linha
3. Sensores Internos (sensor de sobrecorrente)
4. Sensores em geral

Exemplos de dispositivos de **saída**:

1.
2. 




```C
void config()
{
   /*Definindo a direcao de cada uma das portas, A, B, C, D, E.
1 indica input e 0 output e a ordem funciona da seguinte maneira
e.g.

set_tris_a(a7,a6,a5,a4,a3,a2,a1,a0);

*/
   set_tris_a(0b11111111);
   set_tris_b(0b00000000);
   set_tris_c(0b10111111);
   set_tris_d(0b00000000);
   set_tris_e(0b1111);

//Configuracao dos modulos PWM do PIC18F2431
   setup_power_pwm_pins(PWM_BOTH_ON,PWM_BOTH_ON,PWM_BOTH_ON,PWM_BOTH_ON); // Configura os 4 módulos PWM.
   setup_power_pwm(PWM_FREE_RUN, 1, 0, POWER_PWM_PERIOD, 0, 1,30);

   /*
   FREE RUN eh o modo mais aconselhado para mover motores DC, outros modos podem controlar de maneiras especificas, dividindo a frequencia
   o modo UP/DOWN aparentemente eh mais eficiente, eh preciso mais testes para garantir isso.

   1 indica o postscale usado na saída do PWM, o postscale divide a frequencia que recebe por um número, no caso não estamos dividindo por nada

   0 indica o valor inicial da contagem do Timer responsavel pela geracao do pwm

   POWER_PWM_PERIOD eh o periodo do PWM, isso pode controlar a frequencia do PWM de acordo com o periodo, ele eh dado em ciclos do clock

   0  o compare, ele compara esse valor com o valor do timer para verificar se algum evento especial deveria acontecer

   1 o postscale compare,ele afeta o compare utilizado no parâmetro anterior.

   30 o dead time, ele altera a diferença de tempo entre o ON de um PWM e o OFF de seu complementar

   */
   setup_adc_ports(ALL_ANALOG); //Define que todas as portas que possuem conversão A/D serão usadas como conversão A/D
   setup_adc(ADC_CLOCK_INTERNAL); //Utiliza o mesmo clock do PIC para o conversor A/D
   enable_motors(); //Controla a habilitação das duas ponte H, para desabilitar uma delas, basta mudar o valor das constantes ENA e ENB

   disable_interrupts(global); //Habilita interrupcao globais
   enable_interrupts(INT_TIMER1); //Habilita interrupcao do Timer1
   setup_timer_1(T1_INTERNAL|T1_DIV_BY_8); //Configura Timer1 e uso o pre scaler para dividir por 8;

   disable_interrupts(INT_TIMER0);
   setup_timer_0(RTCC_INTERNAL|RTCC_DIV_2|RTCC_8_BIT); //Configura Timer1 e uso o pre scaler para dividir por 8;

   enable_interrupts(INT_TIMER5); //Habilita interrup��o do Timer1
   setup_timer_5(T5_INTERNAL|T5_DIV_BY_1);//Configura Timer5 e divide o seu clock por 1.

   
   
   enable_interrupts(INT_RDA); //Habilita interrupção da porta serial
}

```
