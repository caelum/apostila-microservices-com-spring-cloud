# Encolhendo o monólito

## Desafio: extrair serviços de pedidos e de administração de restaurantes

### Objetivo

Extraia os módulos `eats-pedido` e `eats-restaurante` do monólito para serviços próprios.

Escolha um mecanismo de persistência adequado, fazendo a migração, se necessária.

Minimize as dependências entre os serviços.

Considere o uso de eventos e mensageria.

Não deixe de pensar em client side load balancing e self registration no Service Registry.

Em caso de necessidade, use circuit breakers e retries.

Faça com que os novos serviços usem o Config Server e enviem informações de monitoramento.
