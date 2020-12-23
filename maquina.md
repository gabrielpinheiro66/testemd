# Programa principal (máquina de estados)

---

### Antes da máquina:

~~~python3
#!/usr/bin/env python3

#Bibliotecas

import time
import funcoesRFID
import funcoesBD
import display
import funcoesCalcular
from funcoesCalcular import solenoide
from datetime import datetime
import RPi.GPIO as GPIO
import sys
~~~

Essa primeira linha é muito importante e diz para o serviço que vai rodar o programa que é para rodar com o 'python3'. Haverá mais sobre isso no tutorial sobre serviços.
Depois disso estão os imports de bibliotecas. Note que a 'datetime' e a 'sys' acabaram não sendo utilizadas. A parte do datetime ainda está comentada no código original e não irei mostrar nesse tutorial.

Ainda antes do loop principal temos:

~~~python3
GPIO.setwarnings(False)

print("Programa iniciado")
funcoesBD.salvar_log("Programa iniciado", 0, 0)

lcd = display.lcd()

choppeira = 1
preco = None
chope = None
estado = 'lig'
id = None
total = 0.0
saldoini = 0.0
_insira = False
solenoide(False)
display.bem_vindo(lcd)
time.sleep(2)

leitor = funcoesRFID.Leitor()
leitor.start()



while(True):
~~~

Como pode ser notado, aqui temos apenas algumas definições. Em ordem, o que cada linha faz é:

- Setar os avisos da GPIO como desligados;
- Anunciar que o programa começou (primeiro um print e depois salva no banco da raspberry)
- A variável 'choppeira' serve para especificar onde o código irá ler nome e preço do chopp no banco de dados. Isso foi feito pois sistema foi criado para que possam haver mais choppeiras. No entanto, no momento em que esse tutorial está sendo escrito, só há uma choppeira então esse número sempre deverá ser 1;
- A váriavel 'estado' é setada como 'lig' (máquina acabou de ligar);
- 'id', 'total', 'saldoini', '_insira' ganham valores iniciais;
- Caso a solenoide por algum motivo esteja aberta, disparamos o método para fechá-la;
- Bem-vindo é mostrado no display por 2 segundos;
- O objeto do leitorRFID é criado e a thread dele é iniciada. Observe que ele continuará durante todo o funcionamento dentro do loop, isso  foi definido assim pois o leitor rfid é usado em praticamente todo o funcionamento dos estados. Enquanto isso, observaremos mais adiante que a thread do sensor de vazão é parada quando não há necessidade dela;
- E por último, o while true irá começar nosso loop.

---

### Estado 'lig' (máquina acabou de ligar):

O estado inicial tem a seguinte proposta: Ler no banco de dados qual o nome do chopp e qual o seu preço. Assim, segue o código:

~~~python3
    #Estado em que acabou de ligar
    if estado == 'lig':

        (chope,preco) = funcoesBD.le_preco_chopp(choppeira)
        print(chope)
        print(preco)
        if (preco != None)and(chope != None):
            estado = 0
        else:
            display.falha_bd(lcd)
            print("Erro ao ler chope e preco")
            funcoesBD.salvar_log("Erro ao ler chope e preco", None, None)
            break
~~~

Observe que, caso o programa defina 'preco' ou 'chope' com o valor 'None', isso será reconhecido como falha com o banco de dados e o programa será **INTERROMPIDO**. Definindo corretamente os valores do preço e do nome do chopp, o programa segue para o estado 0

---

### Estado 0 (esperando cartão):

No estado 0, a máquina ficará basicamente parada. Esperando que algum cartão seja inserido para que o cliente possa retirar o chopp. Segue o código:

~~~python3
    elif estado == 0:
        if (id == None)and(_insira == False):
            display.insira(lcd, chope, preco)
            _insira = True
            leitor.zerar()
        id = leitor.get_uid()
        if (id == leitor.get_uid())and(id != None):
           leitor.zerar()
           #logging.warning('%s [INFO] Cartao inserido. ID: %d', agora(), id)
           print(" [INFO] Cartao inserido. ID: ", id)
           funcoesBD.salvar_log("Cartao inserido.", id, 0)
           estado = 1
~~~

Aqui, pela primeira vez vemos uma lógica mais complexa. Para que seja bem absorvida, e necessário que você entenda exatamente como funciona a lógica do código do leitorRFID.

Caso o código chegue pela primeira vez no estado 0, a trava '_insira' **SEMPRE** será 'False' (ela acabou de ser definida assim no estado 'lig'). 
Dessa forma, como você pode ver se acompanhar linha a linha. Na primeira vez sempre id será 'None' e _insira será 'False'. Assim, o **if (id == None)and(_insira == False):** irá retornar verdadeiro.

Agora que o programa entrou nesse 'if', vemos que o código já executa o método do display, muda a trava '_insira' para 'True' e usa o método 'zerar' do leitor.

Como a trava '_insira' foi mudada para 'True' esse if não poderá acontecer mais. Isso foi feito para que o display não fique piscando (caso executasse o método mais de uma vez) e para que as varíaveis de contagem do leitorRFID sejam zeradas apenas uma vez.

Depois disso está a linha que recebe o id (id = leitor.get_uid()). O código ficará executando essa linha quase sempre, basta pensar: O estado 0  é o estado em que a máquina passa mais tempo. Dentro do estado 0, entra-se apenas uma vez no primeiro if e depois a máquina fica o tempo inteiro pegando o uid que está no leitor e vendo se ele não é igual a 'None' (não tem cartão).

Portanto enquanto o id for None (não tem cartão) o loop fica aí e é isso. Apenas esperando.

Quando um cartão for inserido, a o if: **if (id == leitor.get_uid())and(id != None):** deverá retornar verdadeiro. Observe que além de conferir se o id não é 'None', ele também pega o id que ele acabou de pegar na linha anterior (variável 'id') e compara com uma **nova** leitura. Isso apenas para ter certeza que a leitura  foi feita corretamente e não houve troca de cartões nem nada do tipo.

Obs.: **Sim, essa linha citada PRATICAMENTE se equivale ao método tem_cartão, do objeto leitorRFID. O método tem_cartão é mais preciso e por isso foi criado**

~~~python3
if (id == leitor.get_uid())and(id != None):
~~~

~~~python3
if (leitor.tem_cartao()):
~~~

**A linha que está atualmente não foi substituida pois quando o método foi criado, o código já funcionava bem com a primeira opção e para que a segunda fosse implementada, ainda deveriam ser feitos ajustes, o que seria inviável.**

Dessa forma, seguimos com linhas mais fáceis de entender. Primeiro, as variáveis de contagem do RFID são zeradas mais uma vez. Aqui pois agora iremos trabalhar com o cartão dentro da máquina, diferente de como estava antes. Caso isso não fosse feito, haveria como se fosse um 'lixo de memória'.
Depois disso, há os logs e o aviso no display do cartão inserido. Assim seguimos para o estado 1.

---

### Estado 1 (verificando cartão):

O estado 1 tem uma tarefa muito simples, identificar o cartão no banco de dados e obter seu saldo. Segue o código:

~~~python3
    elif estado == 1:
        _insira = False
        while (leitor.tem_cartao())and(estado == 1):

            saldoini = funcoesBD.le_saldo(id)

            if (saldoini > 0):
                sensor = funcoesCalcular.Vazao(preco)
                sensor.start()
                estado = 2
            elif (saldoini == 0)or(saldoini == None)or(saldoini < 0):
                #logging.warning('%s [INFO] Saldo insuficiente ou cartao nao identificado. ID: %d', agora(), id)
                print("Saldo insuficiente ou cartao nao identificado. ID: ", id)
                funcoesBD.salvar_log("Saldo insuficiente ou cartao nao identificado.", id, 0)
                display.insuficiente(lcd, chope, preco)
                leitor.zerar()
                estado = 0
            else:
                estado = 0
~~~

A primeira linha dentro do estado já volta a trava '_insira' para 'False'. Assim, quando a máquina voltar para o estado 0, essa será uma 'primeira vez', novamente.
Após isso, há um while, que é executado enquanto há cartão e o estado ainda é o 1.

A variável 'saldoini' recebe o saldo do cartão. Que para nós é o saldo inicial.

Caso o saldo seja maior que 0, o sensor é definido, a thread iniciada e passaremos para o estado 2.

Caso não haja saldo suficiente ou o cartão não seja identificado, isso é avisado na tela. As variaveis de contagem do leitorRFID são zeradas mais uma vez e a máquina retorna ao estado 0 (esperando cartão).

---

### Estado 2 (chopp sendo retirado):

~~~python3
    elif estado == 2:
     total = 0.00
     display.limpar(lcd)
     solenoide(True)
     while(total<=saldoini)and(leitor.tem_cartao()):
       display.saldo(lcd, saldoini, total)
       total = sensor.get_total()
       #time.sleep(0.0001)
     else:
       solenoide(False)
       if (total >= saldoini):
           print('1')
           total = saldoini
           display.sem_saldo(lcd)
           sensor.stop()
           del sensor
           estado = 3
       else:
           print('2')
           display.cartao_retirado(lcd, saldoini-total)
           print(sensor.pulsos())
           sensor.stop()
           del sensor
           leitor.zerar()
           time.sleep(1)
           estado = 3
~~~

Para que não haja confusão, partes comentadas foram retiradas do tutorial e devem ser desconsideradas.

O estado já começa definindo em 0 a variável 'total', limpando o lcd e abrindo a solenoide.

Após isso temos o loop que funciona enquanto o saldo inicial (saldoini) é maior ou igual ao valor total do chopp retirado (total) e o loop deve funcionar com o cartão RFID inserido no leitor.

Dentro desse loop, temos o display que mostra o saldo inicial e o total (ao vivo). E, além disso, o total é atualizado a cada rotina.
Observe o time.sleep que está comentando ali dentro do loop. Ele pode ser utilizado para limitar a velocidade de funcionamento do loop e, para a versão final, acabou não sendo utilizado.

Quando a máquina sai do loop, pode ser por dois motivos, 1: o valor do chopp retirado se torna maior ou igual ao saldo inicial ou 2: o cliente já retirou o chopp que queria e por isso o cartão foi retirado. O funcionamento em cada uma das opções será:

**1. total >= saldoini:**
- o total é ajustado para receber saldoini (assim nunca acontece saldo negativo de -R$0,02 por exemplo);
- display avisa que o cartão ficou sem saldo;
- a thread é parada e o objeto deletado;
- passamos ao estado 3.

**2. cartão retirado**
- o display avisa que o cartão foi retirado e o preço que foi descontado;
- a thread é parada e o objeto deletado;
- as variáveis de contagem do RFID são zeradas mais uma vez (cartão retirado);
- passamos ao estado 3.

OBS.: **print(sensor.pulsos())** essa linha é utilizada para calibrar o sensor, veja mais no guia de como fazer isso.

---

### Estado 3 (salvando transação (ou não)):

O estado 3 serve basicamente para salvarmos o que foi feito no estado 2. Segue o código:

~~~python3
    elif estado == 3:
       if leitor.tem_cartao():
         print("retire")
       else:
         leitor.zerar()
         if saldoini- total != saldoini:
           salvou = True
           salvou = funcoesBD.altera_saldo(id, saldoini-total)
           print(salvou)
           if(salvou):
            prt = "[INFO] Transacao feita com sucesso. ID: " + str(id) + " | Valor:" + str(total)
            print(prt)
            funcoesBD.salvar_log("Transacao feita com sucesso", id, total)
            id = None
            estado = 0
           else:
            prt = ' [ERRO] Nao foi possivel salvar a transacao. ID: ' + str(id) + ' | Saldo antes: ' + str(saldoini) + ' | Total gasto: ' + str(total)
            ot = ' [ERRO] Nao foi possivel salvar a transacao.' + ' | Saldo antes: ' + str(saldoini) + ' | Total gasto: ' + str(total)
            print(prt)
            funcoesBD.salvar_log(ot, id, 0)
            display.falha_ao_salvar(lcd)
         else:
          # logging.warning('%s [INFO] Chope nao retirado. ID: %d', agora(), id)
           prt = '[INFO] Chope nao retirado. ID: ' + str(id)
           print(prt)
           funcoesBD.salvar_log("Chope nao retirado", id, 0)
           id = None
           estado = 0
~~~

Assim, primeiro o código verifica se ainda há cartão na máquina e não prossegue até que ele não seja retirado (aqui isso pode acontecer caso acabou o saldo de um cartão no estado 2 e ele chegou até aqui ainda na máquina).

Após isso, não tendo cartão, como sempre, usamos o leitor.zerar(). E aí então, serão feitas as verificações para salvar corretamente.

As verificações estão funcionando nesse esquema:

**Chopp foi retirado?** (total != 0)

- Caso sim: 
    - O programa tenta salvar a transação; 
        - Caso seja salva ele avisa e voltamos para o estado 0; 
        - Caso não seja, ele fica travado no aviso de que não salvou e deixa guardado no log.
- Caso não:
    - O programa avisa que o chopp não foi retirado e voltamos para o estado 0.
