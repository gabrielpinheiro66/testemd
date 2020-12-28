# Funções relativas ao Sensor de Vazão e à Solenoide. 

---

Aqui não houve um tutorial específico seguido. Tivemos que ler muito sobre o sensor de vazão e sobre as funções GPIO da raspberry.
Para acompanhar esse tutorial é necessário que você tenha lido o sobre Thread e desejável que saiba como funcionam o sensor de vazão e a válvular solenóide.

Alguns sites que talves possam ser úteis:

- https://www.usinainfo.com.br/blog/projeto-raspberry-pi-3-com-sensor-de-fluxo-de-agua/

---

### Especificações:

- Válvula solenóide:
    - aaaaaaaaaaaaaaaaaaaa

- Sensor de vazão: 
    - aaaaaaaaaaaaaaaaaaaa

---

### Ligação da válvula: 

PEDRO Q FEZ. USAMOS RELÉ


---

### Implementação da válvula: 

Segue o código: 

~~~python3
GPIO.setup(11, GPIO.OUT)
def solenoide(indi):
   if(indi == True):
      GPIO.output(11, GPIO.HIGH)
   elif(indi == False):
      GPIO.output(11, GPIO.LOW)
   else:
      print("Erro solenoide")
~~~

Como pode ser visto, essa é uma função bem simples. Antes de tudo, configuramos o pino 11 da rasp como OUTPUT.
Após isso, é declarada a função que recebe a variável 'indi'. Se indi for True, a saída é configurada como nível lógico alto. Caso seja False, nível lógico baixo.

Dessa forma, no código principal, caso seja para abrir a solenóide, basta apenas passar o indi como True:
~~~python3
funcoesCalcular.solenoide(True)
~~~

---

### Ligação do sensor de vazão:

O sensor de vazão tem 3 fios: VCC, Sinal e GND.

- O VCC deverá ser ligado em 5V. Pode ser no da raspberry como mostrado na imagem ou não;
- O GND deverá ser ligado no GND padrão do projeto. Junto com o GND da rasp;
- O Sinal é o mais complicado:

Como o sensor funciona em 5V e os pinos GPIO da rasp em 3.3V, deverá ser feito um divisor de tensão, como mostra na imagem. Os resistores podem ser escolhidos por quem estiver fazendo desde que cheguem 3.3V na rasp.

**(Não lembro quais nós utilizamos. Se lembrar coloco aqui)**

![Ligacao](https://www.usinainfo.com.br/blog/wp-content/uploads/2017/12/Fluxo.jpg)


---


### Implementação do sensor:

Segue o código das configurações:

~~~python3
GPIO.setup(12,GPIO.IN)
GPIO.add_event_detect(12, GPIO.BOTH)
~~~

Primeiro, definimos o pino 12 como entrada. Depois, adicionamos um evento relativo a esse pino: Nesse caso, sempre que a entrada mudar de estado (False->True ou True->False) iremos contar como evento.
Como sabemos, o sensor funciona mandando pulsos de subida (False->True) de acordo com a vazão. **Ou seja: o número de eventos será igual ao dobro do número de pulsos que nos interessam (pulsos de subida).** 

Classe Sensor:

~~~python3
class Vazao(threading.Thread):
	def __init__ (self, preco, *args, **kwargs):
		super(Vazao, self).__init__(*args, **kwargs)
		self.litros = 0.0
		self.dobro = 0
		self.preco = preco
		self._stop = threading.Event()
	def run(self):
		while True:
			if self.stopped():
				return
			if(GPIO.event_detected(12)):
				self.dobro+=1
			time.sleep(0.0000000001)
	def get_litros(self):
		self.litros = (((self.dobro)/2)/4981.0)
		return self.litros
	def get_total(self):
		self.litros = (((self.dobro)/2)/4981.0)
		return arredondar(self.litros*self.preco)
	def stop(self):
		self._stop.set()
	def stopped(self):
		return self._stop.isSet()
	def __del__(self):
		del self
	def pulsos(self):
		return (self.dobro/2)
~~~

Como pode-se notar, a declaração é como a do leitor RFID. No entanto, aqui o construtor precisa também receber o **preço** do chopp. Dessa forma, a declaração seria:
~~~python3
sensor = funcoesCalcular.Vazao(preco)
~~~

Sendo que, obviamente preco será a variável responsável no código principal por armazenar o preço do chopp.

Sobre as variáveis, teremos:
- litros (recebe 0.0, mas depois irá receber qtde de litros que passaram no sensor);
- dobro (numero inteiro, receberá quantas vezes o 'evento' mencionado anteriormente acontecer);
- preco (recebe o preço do chopp no momento da declaração);
- _stop (flag que será utilizada para parar o loop principal).

No loop principal, apenas contabilizamos o número de eventos disparados. Primeiro verificando se o stop está acionado e depois dobro é incrementado todo evento que aparece. O time.sleep como já explicado algumas vezes serve para deixar o loop mais lento e assim otimizar o processamento. **(deixar o tempo muito alto pode fazer com que o código não pegue eventos que aconteçam e consequentemente erre na medição)**

Após isso temos os métodos, teremos:
- get_litros (retorna quantos litros passaram pelo sensor. Não utilizado no momento);
- get_total (retorna o preço até o momento. Esse sim é utilizado ao invés do anterior. Leva em conta a constante do sensor que deverá ser calibrada);
- stop (será usado para parar o loop);
- stopped (serve para verificar se a thread está parada. Não está sendo utilizado);
- del (serve para **deletar** o objeto);
- pulsos (apenas a divisão de dobro por 2).

Para facilitar aqui estão alguns exemplos de como utilizar a classe no código principal:

~~~python3
sensor = funcoesCalcular.Vazao(preco)         #declarado o objeto com nome de sensor
sensor.start()                                #iniciada a thread. A partir de agora o líquido que passar será contabilizado
total = sensor.get_total()                    #aqui pega-se o preço total do que passou pelo sensor (Ele mesmo já recebeu o preço/litro e mediu a qtde de litros)
sensor.stop()                                 #para a thread
del sensor                                    #deleta o objeto
~~~

**Porque deletar o objeto?**

As outras opções seriam:

- Parar a thread, zerar as variáveis e depois, quando for utilizar novamente, tirar a pausa;
- Não parar a thread, apenas zerar todas as variáveis quando for utilizar novamente.

Achamos mais viável parar e deletar o objeto. E quando o programa tiver que medir vazão, cria-se um novo objeto e inicia-se uma nova thread.
