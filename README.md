## Apache RocketMQ with Spring Boot

# 1. Introdução
Neste tutorial, criaremos um produtor e consumidor de mensagens usando Spring Boot e Apache RocketMQ, uma plataforma de mensagens distribuídas e dados de streaming de código aberto.

# 2. Dependências
Para projetos Maven, precisamos adicionar a dependência RocketMQ Spring Boot Starter:

```
<dependency>
    <groupId>org.apache.rocketmq</groupId>
    <artifactId>rocketmq-spring-boot-starter</artifactId>
    <version>2.0.4</version>
</dependency>
```

# 3. Produção de mensagens
Para nosso exemplo, criaremos um produtor de mensagem básico que enviará eventos sempre que o usuário adicionar ou remover um item do carrinho de compras.

Primeiro, vamos configurar a localização do nosso servidor e o nome do grupo em nosso application.properties:

```
rocketmq.name-server=127.0.0.1:9876
rocketmq.producer.group=cart-producer-group
```

Observe que se tivéssemos mais de um servidor de nomes, poderíamos listá-los como host: porta; host: porta.

Agora, para mantê-lo simples, vamos criar um aplicativo CommandLineRunner e gerar alguns eventos durante a inicialização do aplicativo:

```
@SpringBootApplication
public class CartEventProducer implements CommandLineRunner {

    @Autowired
    private RocketMQTemplate rocketMQTemplate;

    public static void main(String[] args) {
        SpringApplication.run(CartEventProducer.class, args);
    }

    public void run(String... args) throws Exception {
        rocketMQTemplate.convertAndSend("cart-item-add-topic", new CartItemEvent("bike", 1));
        rocketMQTemplate.convertAndSend("cart-item-add-topic", new CartItemEvent("computer", 2));
        rocketMQTemplate.convertAndSend("cart-item-removed-topic", new CartItemEvent("bike", 1));
    }
}
```

O CartItemEvent consiste em apenas duas propriedades - o id do item e uma quantidade:

```
class CartItemEvent {
    private String itemId;
    private int quantity;

    // constructor, getters and setters
}
```

No exemplo acima, usamos o método convertAndSend (), um método genérico definido pela classe abstrata AbstractMessageSendingTemplate, para enviar nossos eventos de carrinho. Ele usa dois parâmetros: um destino, que em nosso caso é um nome de tópico, e uma carga útil da mensagem.

# 4. Mensagem ao consumidor
Consumir mensagens RocketMQ é tão simples quanto criar um componente Spring anotado com @RocketMQMessageListener e implementar a interface RocketMQListener:

```
@SpringBootApplication
public class CartEventConsumer {

    public static void main(String[] args) {
        SpringApplication.run(CartEventConsumer.class, args);
    }

    @Service
    @RocketMQMessageListener(
      topic = "cart-item-add-topic",
      consumerGroup = "cart-consumer_cart-item-add-topic"
    )
    public class CardItemAddConsumer implements RocketMQListener<CartItemEvent> {
        public void onMessage(CartItemEvent addItemEvent) {
            log.info("Adding item: {}", addItemEvent);
            // additional logic
        }
    }

    @Service
    @RocketMQMessageListener(
      topic = "cart-item-removed-topic",
      consumerGroup = "cart-consumer_cart-item-removed-topic"
    )
    public class CardItemRemoveConsumer implements RocketMQListener<CartItemEvent> {
        public void onMessage(CartItemEvent removeItemEvent) {
            log.info("Removing item: {}", removeItemEvent);
            // additional logic
        }
    }
}
```

Precisamos criar um componente separado para cada tópico de mensagem que estamos ouvindo. Em cada um desses ouvintes, definimos o nome do tópico e o nome do grupo de consumidores por meio da anotação @RocketMQMessageListener.

# 5. Transmissão Síncrona e Assíncrona
Nos exemplos anteriores, usamos o método convertAndSend para enviar nossas mensagens. No entanto, temos algumas outras opções.

Poderíamos, por exemplo, chamar syncSend, que é diferente de convertAndSend porque retorna o objeto SendResult.

Pode ser usado, por exemplo, para verificar se nossa mensagem foi enviada com sucesso ou obter seu id:

```
public void run(String... args) throws Exception { 
    SendResult addBikeResult = rocketMQTemplate.syncSend("cart-item-add-topic", 
      new CartItemEvent("bike", 1)); 
    SendResult addComputerResult = rocketMQTemplate.syncSend("cart-item-add-topic", 
      new CartItemEvent("computer", 2)); 
    SendResult removeBikeResult = rocketMQTemplate.syncSend("cart-item-removed-topic", 
      new CartItemEvent("bike", 1)); 
}
```

Como convertAndSend, esse método é retornado apenas quando o procedimento de envio é concluído.

Devemos usar a transmissão síncrona em casos que exigem alta confiabilidade, como mensagens de notificação importantes ou notificação por SMS.

Por outro lado, podemos querer enviar a mensagem de forma assíncrona e ser notificados quando o envio for concluído.

Podemos fazer isso com asyncSend, que usa SendCallback como parâmetro e retorna imediatamente:

```
rocketMQTemplate.asyncSend("cart-item-add-topic", new CartItemEvent("bike", 1), new SendCallback() {
    @Override
    public void onSuccess(SendResult sendResult) {
        log.error("Successfully sent cart item");
    }

    @Override
    public void onException(Throwable throwable) {
        log.error("Exception during cart item sending", throwable);
    }
});
```

Usamos transmissão assíncrona em casos que exigem alto rendimento.

Por último, para cenários em que temos requisitos de rendimento muito altos, podemos usar sendOneWay em vez de asyncSend. sendOneWay é diferente de asyncSend porque não garante que a mensagem seja enviada.

A transmissão unidirecional também pode ser usada para casos de confiabilidade comuns, como coleta de logs.

# 6. Envio de mensagens na transação
RocketMQ nos fornece a capacidade de enviar mensagens dentro de uma transação. Podemos fazer isso usando o método sendInTransaction():

```
MessageBuilder.withPayload(new CartItemEvent("bike", 1)).build();
rocketMQTemplate.sendMessageInTransaction("test-transaction", "topic-name", msg, null);
```

Além disso, devemos implementar uma interface RocketMQLocalTransactionListener:

```
@RocketMQTransactionListener(txProducerGroup="test-transaction")
class TransactionListenerImpl implements RocketMQLocalTransactionListener {
      @Override
      public RocketMQLocalTransactionState executeLocalTransaction(Message msg, Object arg) {
          // ... local transaction process, return ROLLBACK, COMMIT or UNKNOWN
          return RocketMQLocalTransactionState.UNKNOWN;
      }

      @Override
      public RocketMQLocalTransactionState checkLocalTransaction(Message msg) {
          // ... check transaction status and return ROLLBACK, COMMIT or UNKNOWN
          return RocketMQLocalTransactionState.COMMIT;
      }
}
```

Em sendMessageInTransaction (), o primeiro parâmetro é o nome da transação. Deve ser igual ao campo de membro txProducerGroup de @ RocketMQTransactionListener.

# 7. Configuração do Produtor de Mensagem
Também podemos configurar aspectos do próprio produtor da mensagem:

- rocketmq.producer.send-message-timeout: O tempo limite de envio da mensagem em milissegundos - o valor padrão é 3000;
- rocketmq.producer.compress-message-body-threshold: Limite acima do qual RocketMQ compactará as mensagens - o valor padrão é 1024;
- rocketmq.producer.max-message-size: O tamanho máximo da mensagem em bytes - o valor padrão é 4096;
- rocketmq.producer.retry-times-when-send-async-failed: O número máximo de novas tentativas para executar internamente em modo assíncrono antes de enviar falha - o valor padrão é 2;
- rocketmq.producer.retry-next-server: indica se deve tentar novamente outro broker ao enviar falha internamente - o valor padrão é false;
- rocketmq.producer.retry-times-when-send-failed: O número máximo de novas tentativas para executar internamente em modo assíncrono antes de enviar falha - o valor padrão é 2.

# 8. Conclusão
Neste artigo, aprendemos como enviar e consumir mensagens usando Apache RocketMQ e Spring Boot. Como sempre, todo o código-fonte está disponível no GitHub.


