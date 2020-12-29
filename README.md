# Essa documentação está quase pronta.

---

# Sobre o projeto do jyba: 

O projeto consiste em automatizar o funcionamento de uma choppeira, isso se daria da seguinte forma: O cliente teria um cartão (fornecido pela loja) e esse cartão irá conter os créditos do cliente. O caixa da chopperia poderá vender créditos para o cliente. Com os créditos no cartão, o cliente apenas insere o cartão na chopperia, que abre uma válvula e autoriza que o cliente tire o chopp. Além disso deve ter um display na choppeira demonstrando o que está acontecendo.

Dessa forma, foi necessário:

- Deve ser criado um banco de dados na rede local para o armazenanamento de todas as informações e que pode ser acessado tanto pelo caixa quanto pela choppeira;
- Uma interface que seja possível consultar o histórico de transações e adicionar e remover crédidos dos clientes;
- Na choppeira, deverá haver um microcontrolador que acesse a internet (banco de dados) e possa controlar todos os atuadores; 
- Atuadores: 
    - Válvula solenóide (válvula de fácil acionamento que irá permitir ou bloquear a passagem  de chopp);
    - Sensor de Vazão (sensor que irá medir quanto de chopp está sendo retirado, ao estilo bomba de gasolina);
    - 2 Leitores RFID (sensor que irá ler a tag dos cartões RFID, um deverá estar na choppeira e outro no caixa);
    - Display para a choppeira;
    
    
---

### Como foi pensado o funcionamento:

O funcionamento da choppeira foi proposto em máquina de estados para que facilitasse a implementação. Ele será demonstrado nessa mesma lógica. Segue o esquema atual para um entendimento 

![Esquema](https://raw.githubusercontent.com/gabrielpinheiro66/testemd/main/Untitled%20Diagram(1).jpg)

---

### Explicação dos códigos:

- maquinaJyba.py -> maquina.md
- funcoesRFID.py -> rfid.md
- funcoesCalcular.py -> calcular.md
- funcoesBD.py -> (ainda n tem)
- display.py -> display.md
