# Funções relativas ao display.

---

Basicamente tudo que fizemos aqui veio da biblioteca CharLCD. Sugiro ler sobre ela.

- https://rplcd.readthedocs.io/

---

### Especificações do display utilizado:

- Nao sei

---

### Código:

Obs.: Quando estávamos com muitos problemas em otimizar o código tentamos várias estratégias, inclusive criar uma thread para o display (o que seria ótimo, mas deixou o código tão lento que não utilizamos)

Por causa de tudo isso, o código **display.py** está com mais de 200 linhas. No entanto, **apenas as 70 primeiras linhas são utlizadas no projeto** e por isso apenas elas serão explicadas.

~~~python3
import time
from RPLCD.i2c import CharLCD
import RPi.GPIO as GPIO

def lcd():
    lcd = CharLCD('PCF8574', 0x27)
    return lcd

def limpar(lcd):
    lcd.clear()

def bem_vindo(lcd):
        lcd.clear()
        lcd.cursor_pos = (0,6)
        lcd.write_string('BEM-VINDO')
        lcd.cursor_pos = (2,0)
        lcd.write_string("Feito por Asimov Jr.")
def falha_bd(lcd):
        lcd.clear()
        lcd.cursor_pos = (0,8)
        lcd.write_string("ERRO")
        lcd.cursor_pos = (2,0)
        lcd.write_string("Falha ao obter dados do chope")
def insira(lcd, chope, preco):
        lcd.clear()
        lcd.cursor_pos = (0,0)
        lcd.write_string("Chope: %s" % chope)
        lcd.cursor_pos = (1,0)
        lcd.write_string("Preco: R$ %s/L" % round(preco, 2))
        lcd.cursor_pos = (3,0)
        lcd.write_string("Insira o cartao")
def insuficiente(lcd, chope, preco):
        lcd.clear()
        lcd.cursor_pos = (0,0)
        lcd.write_string("Chope: %s" % chope)
        lcd.cursor_pos = (1,0)
        lcd.write_string("Preco: R$ %s/L" % round(preco, 2))
        lcd.cursor_pos = (2,0)
        lcd.write_string("Saldo insuficiente")
        lcd.cursor_pos = (3,0)
        lcd.write_string("ou nao identificado")
def saldo(lcd, saldoini, total):
        lcd.cursor_pos = (0,0)
        lcd.write_string("Saldo: R$ %.2f" % saldoini)
        lcd.cursor_pos = (1,0)
        lcd.write_string("Valor: R$ %.2f" % total)
        lcd.cursor_pos = (3,0)
        lcd.write_string("Retire seu chope")
def sem_saldo(lcd):
        lcd.clear()
        lcd.cursor_pos = (0,2)
        lcd.write_string("Saldo esgotado!")
        lcd.cursor_pos = (2,2)
        lcd.write_string("Retire o cartao")
def cartao_retirado(lcd, saldo):
        lcd.clear()
        lcd.cursor_pos = (0,2)
        lcd.write_string("Cartao retirado!")
        lcd.cursor_pos = (2,0)
        lcd.write_string("Saldo: R$ %.2f" % saldo)
def falha_ao_salvar(lcd):
        lcd.clear()
        lcd.cursor_pos = (0,0)
        lcd.write_string("ERRO AO SALVAR SALDO")
        lcd.cursor_pos = (2,0)
        lcd.write_string("Tentando novamente..")
def saldo_aovivo(lcd, total):
        lcd.cursor_pos(1,10)
        lcd.write_string("%.2f" % total)
~~~

---

### Algumas estratégias interessantes:

- No estado 0, essa trava '_insira' além de ajudar em outras coisas que já foram explicadas, faz com que o display não seja executado toda hora (caso fosse, a mensagem ficaria piscando)
~~~python3
    elif estado == 0:
        if (id == None)and(_insira == False):
            display.insira(lcd, chope, preco)
            _insira = True
            leitor.zerar()
~~~
  
- Um problema que demoramos pra identificar foi:
~~~python3
display.bem_vindo(display.lcd())     ### MT LENTO E TRAZIA PROBLEMAS
~~~
~~~python3
lcd = display.lcd()                 ### AQUI TRAZEMOS O LCD PARA O CODIGO NO INICIO
display.bem_vindo(lcd)              ### E TDS FUNCOES RECEBEM O LCD Q JA ESTAVA NO COD PRINCIPAL
~~~

- Dentro do estado 2 fica piscando o display?
~~~python3
while(total<=saldoini)and(leitor.tem_cartao()):
       display.saldo(lcd, saldoini, total)
       total = sensor.get_total()
~~~

 - Se você reparar, a função display.saldo não executa nenhuma vez lcd.clear;
 - Outra coisa é que ela é executada muito rápido (não há delay).
     
Então ou não pisca ou pisca tão rápido que nem dá pra perceber kkkkk, nem sei, só sei que dá certo assim.

