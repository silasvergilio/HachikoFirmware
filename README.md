# HachikoFirmware
Repositório para ensinar novos e antigos competidores de Sumô 3kg (Auto) a usar o código usado no robô Hachiko da equipe FEG-Robótica e adaptá-lo para seu próprio robô.

## Sumário

1. [Objetivo](#objetivo)
2. [Hardware](#hardware)
    1. [Placa Hachiko](#placaHachiko)
    2. [Mecânica Hachiko](#mecHachiko)
3. [Software](#software)
    1. [Configuração](#configuracao)
        1. [Sentido dos Pinos](#sentidoPinos)
        2. [PWM](#pwm)
        3. [ADC](#adc)
        4. [Diretivas de Compilador](#diretivas)

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

Não **muito**. Todavia algumas decisões na parte de estratégias mudariam. Mas não vamos colocar a carroça na frente dos bois. Vamos seguir com nossas primeiras linhas de código.

<a name="software"></a>
#### Software

A linguagem de programação usada neste projeto foi a linguagem C, todavia ao se programar PIC é natural que exista diferentes compiladores, um muito comum seria usar o MPLab, todavia para este projeto foi usado o compilador CCS. Ele tem uma linguagem simples de usar que não exige grande conhecimento sobre manipulação de bits e uso de registradores específicos.

<a name="configuracao"></a>
##### Configuração

<a name="sentidoPinos"></a>
##### Sentido dos pinos
Toda a seção de configuração do hardware foi feita em uma única função da biblioteca *hachiko_reference.c*, a primeira parte que entraremos em detalhe é o *set_tris*, essa é uma função que define o sentido de cada porta, ou seja, se determinado pino de cada porta é **entrada** ou **saída**, cabe ao programador em conjunto com o seu grupo da eletrônica analisar qual a função de cada porta da sua placa. 

Exemplos de dispositivos de **entrada**:

1. Sensor de Distância
2. Sensor de Linha
3. Sensores Internos (sensor de sobrecorrente)
4. Sensores em geral

Exemplos de dispositivos de **saída**:

1. Sinal PWM
2. LED
3. Controle dos Motores

Para o caso do o *set_tris_x* ele recebe como argumento um número, em que cada bit dele representa um pino de determinada porta X. A maneira indicada aqui para fazer tal é escrever este número já na forma binária para facilitar sua vida usando a notação *0b* antes do número, como indicado abaixo. Onde 1 indica *input* enquanto que 0 indica *output*.
 

```C
   set_tris_a(0b11111111);
   set_tris_b(0b00000000);
   set_tris_c(0b10111111);
   set_tris_d(0b00000000);
   set_tris_e(0b1111);
```

A decisão dos números foi feita baseada no esquemático da placa acima. Há uma grande chance do seu projeto usar um Arduino, neste caso bastaria usar o comando *pinMode* para definir o sentido, como por exemplo:

```C
pinMode(pinoDesejado,INPUT); //Para um dispositivo de entrada
pinMode(pinoDesejado2,OUTPUT); //Para um dispositivo de saída
```

<a name="pwm"></a>
##### PWM

Existem 3 principais maneiras de gerar um sinal PWM usando o PIC18F4431. Usando o canal CCP (capture, compare, PWM), método que nunca foi usado por mim. Criando a onda "manualmente" a partir das interrupções do timer, fazendo os devidos cálculos, ou por fim, o método que usamos chamado "Power PWM", as principais vantagens são as seguintes:

1. Fácil utilização
2. Suporta frequências mais altas (faixa acima de 50kHz)
3. Permite diversos modos de utilização

Abaixo seguem as linhas necessárias para configurar o PWM, logo em seguida vamos explicar em mais detalhes como compreender e configurar para a sua necessidade. 

```C
//Configuracao dos modulos PWM do PIC18F4431

   setup_power_pwm_pins(PWM_BOTH_ON,PWM_BOTH_ON,PWM_BOTH_ON,PWM_BOTH_ON); // Configura os 4 módulos PWM.
   setup_power_pwm(PWM_FREE_RUN, 1, 0, POWER_PWM_PERIOD, 0, 1,30);
```
No Pic18F4431 existem 4 canais PWM, eles trabalham em dupla, na primeira linha decidimos qual será a relação entre cada um destes canais (duplas). Neste caso definimos todos como "PWM_BOTH_ON", indicando que eles funcionaram de maneira idêntica entre si e de modo não complementar.


A segunda linha indica como irá funcionar a onde do PWM em si, vamos compreender cada um dos argumentos.

1. O primeiro valor 'PWM_FREE_RUN', indica o modo de contagem em que o PWM irá se basear, o modo FREE RUN é o mais aconselhado para mover motores DC
2. O valor **1** indica o postscale da frequência usada para o PWM, com o valor unitário a frequência não será dividida, e para todos os cálculos vamos usar o valor de 5Mhz, que seria 20Mhz da placa dividido por 4 (divisão padrão realizada com toda frequência que entra do cristal no PIC).
3. **0** na terceira posição indica qual o valor inicial da contagem do contador para o PWM, este e outros valores ajudam a manipular melhor a frequência desejada no PWM
4. **POWER PWM PERIOD** é uma constante definida no programa que é o valor que representa o período da onda PWM, dentro deste projeto e outros é a maneira convencional de ajustar a frequência desejada, todavia não o único. Após estudos realizados pela equipe, que podem ser vistos **aqui[TODO LINK PARA CIC 2018]**, a frequência de 50kHz foi adotada pela melhor eficiência do motor Maxon.
5. Este valor **0** indica que não haverá tratamento de nenhum evento especial ligado ao PWM.
6. O 1 indica o postscale, mais uma opção para controle da frequência que desejamos utilizar.
7. 30 é o valor para o *deadtime* do PWM, é um valor pequeno apenas para segurança da eletrônica de potência no chaveamento.

Notamos a partir desta configuração que o principal valor a ser controlado neste caso é o valor do período do PWM, este valor é dado em ciclos de instrução do PIC. Exemplo:

Para um cristal de 20Mhz, no PIC entra na realidade 4MHz, 1/4000000 = 0.25us  , este será o tempo de cada instrução. Usando este valor para entendermos o período, caso tenhamos um valor de **POWER PWM PERIOD** de 500, isso representa um período de 0.125ms. A fim de calcularmos a frequência, basta usarmos a equação f = 1/T, onde *f* é a frequência, e *T* é o período que apresentamos anteriormente. Portanto para o exemplo aqui dado a frequência seria de 8kHz. 

Para o caso do **Arduino** não há uma maneira de trabalhar com maiores frequência,todavia é preciso utilizar as portas adequadas da placa em questo. O Arduino trabalha em geral com frequência de 1kHz.

<a name="adc"></a>
##### Conversor Analógico-Digital (ADC)

O conversor analógico-digital tem por função transformar um valor de tensão analógica (entre 0V e 5V) em um valor digital que vai de 0 a 1023, é a única maneira que temos de ler o valor de um grande grupo de sensores, incluindo alguns modelos de sensores que nos ajudarão a enxergar o adversário e a linha branca do *dohyo*. A sua configuração é simples, sua utlização é menos simples, mas trataremos disto mais a frente. O código abaixo configura todas as portas que podem, como conversores.

> Observação: Existem 8 portas que podem funcionar como conversores A/D, todavia o PIC na verdade tem apenas **um** conversor A/D, para usar mais do que uma porta para conversão o PIC possui um multiplexador, mais sobre isso será explicado na função da leitura do conversor A/D mais a frente.

```C
   setup_adc_ports(ALL_ANALOG); 
   setup_adc(ADC_CLOCK_INTERNAL); 
```

A primeira linha do excerto de código acima define que todas as portas possíveis serão analógicas, para  configurar apenas algumas portas como ADC é preciso consultar a documentação específica do CCS. A segunda linha configura o ADC para usar o clock que entra no PIC (no caso da nossa placa será 4Mhz) para realizar a conversão. O processo de conversão é sensível ao tempo, por isso esta informação é importante, alguns circuitos usam clocks externos para melhorar a velocidade de conversão. No nosso caso **ler um pino digital leva muito menos tempo do que realizar uma conversão analógico-digital**.


<a name="dretivas"></a>
##### Diretivas de Programação

Algumas das configurações que realizamos não acontecem dentro da funço *config()*, algumas delas ocorrem no arquivo principal, no formato que chamamos de *diretivas de programação*, so comandos que servem apenas para o compilador e não para a nossa sequência de códigos lógicos, é como uma instrução que não será executada pelo PIC em si, e sim pelo compilador a fim de garantir algumas configurações no funcionamento mais básico do Hardware. A primeira destas configurações se relaciona com o último tópico que falamos, de ADC. Toda diretiva é caracterizada por começar por *#*. 

```C
#device adc=10 
```
A diretiva acima garante que o nosso conversor terá a resolução de 10bits, ou seja, um valor de tensão de 0V a 5V irá gerar um número de 0 - 2^10 - 1


















