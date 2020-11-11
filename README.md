# Thread em python

### O que é e para que serve?

Para um projeto em específico, estávamos usando uma RaspberryPi como um controlador em python. No entanto, diferente de um controlador, a raspberry em python não poderia executar multitarefas no mesmo programa (assim como faria um microcontrolador). Para exemplificar: Era necessário que enquanto a lógica do programa funcionasse, outras duas lógicas funcionassem ao mesmo tempo, uma verificando se havia um cartão RFID inserido no sensor e outra lógica monitorando um sensor de vazão. No início do projeto, não havíamos pensado nessa parte, mas quando eram feitos testes, percebemos que o programa executando linha por linha não conseguia monitorar os sensores o tempo todo, assim, não era possível funcionar corretamente. Fomos procurar algum tipo de programação assícrona e por sugestão de um professor acabamos indo atrás da biblioteca threading.

 ### Porque nem sempre utilizar?

 Com threads rodando paralelamente ao código, o processamento irá ser consideravelmente mais lento. E não é por demérito da placa: Como o interpretador do Python não suporta a verdadeira execução multi-core via multithreading, a biblioteca threading irá apenas "tapar buracos" aqui. Quanto mais threads você utilizar (mais execuções em paralelo), mais lenta será a execução.

 ### Apresentação do código base em si:

~~~python3
import threading
class Leitor(threading.Thread):
	def __init__ (self, *args, **kwargs):
		super(Leitor, self).__init__(*args, **kwargs)
		# Definir variáveis iniciais aqui. Exemplo:
		self.const = 1.0
	def run(self):
		# AQUI ESTARÁ O LOOP QUE SE REPETIRÁ EM PARALELO COM O PROGRAMA
~~~

Como pode ser visto, apenas há a importação da biblioteca threading e a declaração da classe Leitor que nós usaremos. Sendo que ela herda a classe Thread. No código principal, assim seria declarado o nosso Leitor:

    objeto = Leitor() ## aqui foi declarado o objeto do tipo Leitor
    Leitor.start() ## começa a ser executado o loop escrito em run
    

 ### Apresentação de um código do projeto:
 
 Agora será apresentado e explicado um código utilizado no projeto para o sensor de vazão:
 
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
		self.litros = (((self.dobro)/2)/4615.0)
		return self.litros
	def get_total(self):
		self.litros = (((self.dobro)/2)/4615.0)
		return arredondar(self.litros*self.preco)
	def stop(self):
		self._stop.set()
	def stopped(self):
		return self._stop.isSet()
	def __del__(self):
		del self
	def pulsos(self):
		return (self.dobro/2)
		
		
Como pode ser notado, as funções **get_litros**, **get_total** serão usados para retornar no código principal a quantidade de litros lida e o preço total, respectivamente.

Algo muito importante é que na segunda linha, nota-se que o objeto do tipo Vazao irá receber o valor do preco já no momento da declaração.

Além disso, existem dois métodos (**stopped** e **stop**) e uma variável (**_stop**) que serão usados para parar a execução da thread (observe como está feito no run também). Não irei entrar em detalhes do funcionamento pois além dele ser auto-explicativo, há várias formas de criar um método de parar uma thread, todas são de certa forma um improviso, e essa foi uma que funcionou bem no projeto.

Por último há um metodo **__del__** apenas para caso seja necessário apagar o objeto.

Assim, ficaria o funcionamento:

    sensor = Vazao(preco) ## declaração levando a variavel preco
    sensor.start() ## comeca o loop
    
    total = sensor.get_total() ## caso queira o pegar o total, por exemplo
    
    sensor.stop() ## para parar o loop
    del sensor ## deletar o objeto sensor
    
    
### Porque o time.sleep?

Como pode ser notado, no final do loop do método run() foi usada uma linha 

    time.sleep(0.0000000001)
    
Essa linha serve mesmo como delay no loop com a intenção de **otimizar o processamento**. O número 0.0000000001 foi definido por testes. Quanto maior o valor, pior a medição e, quanto menor, mais as threads irão comprometer o resto do código

### Último exemplo:

    class Leitor(threading.Thread):
	def __init__ (self, *args, **kwargs):
		super(Leitor, self).__init__(*args, **kwargs)
		self.uid = ''
		self.const = 0.02
		self.antigo = None
		self._tem_cartao = threading.Event()
		self.countcerto = 1
		self.countnone = 0
	def run(self):
		while True:
			self.uid = ler_cartao()
			if(self.uid == None)or(self.uid == ''):
				self.countnone += 1
			else:
				self.countcerto += 1
			if(self.taxa() <= self.const):
				self._tem_cartao.set()
			elif(self.countcerto + self.countnone < 50):
				pass
			else:
				self._tem_cartao.clear()
			time.sleep(0.01)
	def get_uid(self):
		return self.uid
	def tem_cartao(self):
		return self._tem_cartao.isSet()
	def taxa(self):
		return self.countnone/(self.countcerto+self.countnone)
	def zerar(self):
		self.countcerto = 1
		self.countnone = 0
	
No código o leitor RFID, foi criada uma tag **_tem_cartao** que, será retornada no método **tem_cartao**. Caso sejam True, significa que tem cartão inserido. Caso sejam False, indica o contrário.

Como o sensor não é perfeito, para determinar se há cartão ou não, a lógica é repetida várias vezes e o programa compara uma taxa em % de False com a constante definida em 2%. Caso haja mais de 2% de False, não há cartão. Caso haja menos que isso, há cartão inserido. Caso hajam menos de 50 leituras, o código não pode assumir que não há cartão ainda.

O método **zerar()** é utilizado para zerar todas as contagens quando um novo cartão é inserido. 
