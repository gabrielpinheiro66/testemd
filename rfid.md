# Funções relativas ao leitor RFID.

Grande parte do que foi feito aqui foi seguindo [este tutorial](https://www.filipeflop.com/blog/controle-de-acesso-rfid-com-raspberry-pi/)

### Ligação na placa:

| Pino Leitor RFID | Pino Raspberry Pi (BOARD) |
|------------------|---------------------------|
| SDA              | 24                        |
| SCK              | 23                        |
| MOSI             | 19                        |
| MISO             | 21                        |
| IRQ              | -                         |
| GND              | 6                         |
| RST              | 22                        |
| 3.3V             | 1                         |

### Informações importantes:

- Modelo do sensor RFID: MFRC522;
- Biblioteca utilizada: arquivo **MFRC522.py**;
- Leia o tutorial sobre threads em python antes desse tutorial.

- **Antes de tudo:**
  A porta SPI precisa estar habilitada. Primeiro verifique se ela está:
    - Digite no terminal:
        ~~~bash
        ls /dev/spi*
        ~~~
    - Caso a saída seja:
        ~~~bash
        ls: cannot access /dev/spi*: No such file or directory
        ~~~
    Significa que você precisa habilitar a porta SPI. 
    
    - Primeiro, use o comando:
        ~~~bash
        sudo raspi-config
        ~~~
     Vá até 'Interfacing options' e 'SPI' e ative.
     
     - Em seguida:
     ~~~bash
     sudo nano /boot/config.txt
     ~~~
     - E adicione a seguinte linha a esse arquivo:
     ~~~bash
     dtoverlay=spi-bcm2708
     ~~~

- **Biblioteca SPI:**
  Como pode ser notado, na pasta jyba, há uma pasta chamada SPI-Py. Essa é a biblioteca chamada 'spi' que a MFRC522 utiliza. Para fazer a instalação, siga os passos (execute os comandos de preferência na pasta do projeto):
    - Instale o python dev:
        ~~~bash
        sudo apt install python3-dev
        ~~~
    - Baixe a pasta, entre nela e depois faça a instalação:
    	~~~bash
        git clone https://github.com/lthiery/SPI-Py.git
        cd SPI-Py
        sudo python3 setup.py install
        ~~~
	
	
### Ler cartão:

Segue o código:

~~~python3
def ler_cartao():
    uid = None
    a = None
    b = []
    try:
        LeitorRFID = MFRC522.MFRC522()
        status, tag_type = LeitorRFID.MFRC522_Request(LeitorRFID.PICC_REQIDL)
        if status == LeitorRFID.MI_OK:
            status, uid = LeitorRFID.MFRC522_Anticoll()
    finally:
        LeitorRFID.closeSPI()
        if uid == None:
           return None
        else:
           return traduzir(uid)
~~~

Basicamente, aqui não há muito que explicar porque nós mesmos fizemos seguindo um tutorial e deu certo tranquilamente. O Leitor RFID é declarado pela biblioteca e as linhas seguintes são primeiro verificando se há uma tag perto do módulo e depois retornando a tag para a variável 'uid'.

Foi usado o try e o finally para que, caso haja qualquer problema quando o código esteja pegando a tag, mesmo assim seja possível fechar a porta SPI com o método closeSPI e retornar o uid, mesmo que seja como 'None'.

**Obs.: O método closeSPI não existia a biblioteca originalmente e nós o criamos após vários problemas (o código abria milhares de portas e nunca as fechava). Segue o que foi feito na biblioteca:**

~~~python3
  def closeSPI(self):
     spi.closeSPI()
~~~

Sim, a explicação para o que foi feito é: Apenas adicionamos à classe MFRC522 o método closeSPI() já existente na classe spi (que o própio MFRC522 já utilizava)


### Traduzir o uid:

O seguinte método foi criado porque o leitorRFID que fica no caixa da chopperia e o leitorRFID que usamos na rasp lêm os bits dos cartões de forma **diferente**
O método consiste em trocar alguns bits de lugar e transformá-los de hexadecimal (como a biblioteca os retorna) para inteiro. Esse método não será explicado passo a passo.

~~~python3
def traduzir(uid):
  if len(uid) == 5:
    a = []
    for i in range(len(uid)):
        a.append("%X" % uid[4-i])
    a.pop(0)
    for i in range(len(a)):
            a[i] = str(a[i]).zfill(2);
    a = ''.join("%s" % s for s in a)
    x = int(a, 16)
    return x
  else:
    return None
~~~



### Classe Leitor:

Segue o código da declaração da classe e do construtor:

~~~python3
class Leitor(threading.Thread):
	def __init__ (self, *args, **kwargs):
		super(Leitor, self).__init__(*args, **kwargs)
		self.uid = ''
		self.const = 0.02
		self.antigo = None
		self._tem_cartao = threading.Event()
		self.countcerto = 1
		self.countnone = 0
~~~

Basicamente, a classe herda a classe threading.Thread e o construtor de Thread também é chamado no construtor de Leitor. (Pra isso o uso da palavra especial **super**)
As variáveis declaradas são:
- uid: O uid armazenado do cartão lido;
- const: razão de 'None' aceitável quando há um cartão inserido no leitor (fixado em 2% empiricamente);
- antigo: (nao é mais utilizada mas esqueci de apagar);
- _tem_cartao: flag que irá indicar se há cartão;
- countcerto: numero de vezes que o leitor pegou o uid certo do cartão;
- countnone: numero de vezes que o leitor pegou None.

Método run (loop da thread) e método taxa:

~~~python3
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
            
def taxa(self):
	return self.countnone/(self.countcerto+self.countnone)
~~~

Descrição passo a passo: 

O método ler_cartão é disparado e retorna o uid. Caso o uid seja None ou vazio, a contagem countnone é incrementada. Caso contrário, a contagem countcerto.

Após isso, a taxa de None obtida é comparada à contante definida (2%):
- Caso a taxa de 'None' seja menor ou igual à mínima definida, assume-se que há cartão (_tem_cartão recebe set);
- Caso a taxa de 'None' seja maior que a mínima definida, no entanto haja menos de 50 leituras, a flag _tem_cartão não é alterada;
- Caso haja mais que 50 leituras e a taxa de 'None' ainda seja maior, assume-se que não há cartão (_tem_cartão recebe clear).

Outros métodos da classe:

~~~python3
def get_uid(self):
	return self.uid
def tem_cartao(self):
	return self._tem_cartao.isSet()
def zerar(self):
	self.countcerto = 1
	self.countnone = 0
~~~

Esses são os métodos que serão usados no código principal (máquinaJyba). Um get para o uid, um método para verificar o estado da flag _tem_cartao e um método para zerar os contadores.




