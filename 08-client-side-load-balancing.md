# Client Side Load Balancing com Ribbon

## Exercício: executando uma segunda instância do serviço de distância

1. Para que todas as requisições do serviço de distância sejam logadas (e com mais informações), vamos configurar um `CommonsRequestLoggingFilter`.

  Para isso, crie a classe `RequestLogConfig` no pacote `br.com.caelum.eats.distancia`:

  ####### eats-distancia-service/src/main/java/br/com/caelum/eats/distancia/RequestLogConfig.java

  ```java
  @Configuration
  public class RequestLogConfig {

    @Bean
    public CommonsRequestLoggingFilter requestLoggingFilter() {
      CommonsRequestLoggingFilter loggingFilter = new CommonsRequestLoggingFilter();
      loggingFilter.setIncludeClientInfo(true);
      loggingFilter.setIncludeQueryString(true);
      loggingFilter.setIncludePayload(true);
      return loggingFilter;
    }

  }
  ```

  Os imports são os seguintes:

  ```java
  import org.springframework.context.annotation.Bean;
  import org.springframework.context.annotation.Configuration;
  import org.springframework.web.filter.CommonsRequestLoggingFilter;
  ```

2. Mude o nível de log do `CommonsRequestLoggingFilter` para `DEBUG` no `application.properties`:

  ####### eats-distancia-service/src/main/resources/application.properties

  ```properties
  logging.level.org.springframework.web.filter.CommonsRequestLoggingFilter=DEBUG
  ```

3. Configure a segunda instância para que seja executada na porta `8282`.

  No Eclipse, acesse o menu _Run > Run Configurations..._.

  Clique com o botão direito na configuração `EatsDistanciaServiceApplication` e, então, na opção _Duplicate_.

  Deve ser criada a configuração `EatsDistanciaServiceApplication (1)`.

  Na aba _Arguments_, defina `8282` como a porta dessa segunda instância, em _VM Arguments_:

  ```txt
  -Dserver.port=8282
  ```

  Clique em _Run_. Nova instância do serviço de distância no ar!

4. Acesse uma URL do serviço de distância que está sendo executado na porta `8082` como, por exemplo, a URL `http://localhost:8082/restaurantes/mais-proximos/71503510`. Verifique os logs no Console do Eclipse, na configuração `EatsDistanciaServiceApplication`.

  Use a porta para `8282`, por meio de uma URL como `http://localhost:8282/restaurantes/mais-proximos/71503510`. Note que os logs do Console do Eclipse agora são da configuração `EatsDistanciaServiceApplication (1)`.
