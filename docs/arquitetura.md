# Arquitetura e decisões

## Evolução da infraestrutura

A estrutura inicial possuía dois pfSense conectados ao mesmo switch central. Apesar da duplicação dos firewalls, a falha desse switch interromperia o acesso das redes internas. O laboratório passou a utilizar dois cores e dois uplinks por switch de acesso.

A função dos cores neste desenho é de camada 2. O roteamento das redes e os gateways dos clientes permanecem nos pfSense. Não foi configurado HSRP nos switches, nem um gateway adicional concorrente com os VIPs CARP.

## Papéis distintos

| Recurso | Função |
|---|---|
| VLAN / 802.1Q | Identificação e transporte das redes de camada 2 |
| CARP | Endereços virtuais compartilhados entre os firewalls |
| XMLRPC | Cópia de configurações compatíveis do primário para o secundário |
| pfsync | Sincronização de estados de conexões |
| Kea HA | Coordenação do DHCP e das concessões |
| Rapid PVST+ | Topologia de camada 2 sem loops, com instância por VLAN |

O pfsync estava habilitado no backup do secundário, usando `opt1` e o peer `172.16.1.3`. A continuidade de sessões durante falha completa de um firewall não foi testada. Comunicação Kea HA não comprova, sozinha, funcionamento do pfsync ou do CARP em todas as redes.

## Trunks e portas access

A máquina envia tráfego sem tag para uma porta access. O switch associa esse tráfego à VLAN da porta. Nos uplinks, a VLAN correspondente é transportada com tag 802.1Q.

Exemplos finais:

| Switch de acesso | Porta da máquina | VLAN | Uplinks |
|---|---|---:|---|
| SW-CORP | `Gi0/1` | 10 | `Gi0/0` para CORE1 e `Gi0/2` para CORE2 |
| SW-GUEST | `Gi0/1` | 20 | `Gi0/0` para CORE1 e `Gi0/2` para CORE2 |

Nos trunks de acesso, a VLAN nativa permanece 1, mas a lista permitida contém somente a VLAN de usuários daquele switch. A gerência sem tag é transportada entre firewalls e cores, e entre os cores; não foi estendida como serviço aos switches de acesso por esses trunks restritos.

## Gerência no pai das VLANs

No pfSense, `vtnet3` permanece atribuída à GERENCIA e também é o pai das subinterfaces `vtnet3.10`, `vtnet3.20` e `vtnet3.30`. Tráfego sem tag nesse enlace pertence à rede de gerência.

Para preservar esse desenho, os trunks firewall–core e core–core passaram a permitir `1,10,20,30`, com VLAN nativa 1. Os trunks iniciais permitiam apenas `10,20,30`.

Uma migração futura para VLAN de gerência explicitamente marcada pode ser planejada separadamente. Ela não foi realizada neste projeto.

## Spanning Tree

Foi utilizado `spanning-tree mode rapid-pvst`.

| Equipamento | Prioridade base planejada | Papel |
|---|---:|---|
| CORE1 | 24576 | Root principal |
| CORE2 | 28672 | Root secundário |
| SW-CORP / SW-GUEST | 32768 | Switches de acesso |

Os comandos `root primary` e `root secondary` configuram a preferência de eleição. O valor exibido no STP inclui o identificador estendido da VLAN: por exemplo, `24586 = 24576 + 10` e `24596 = 24576 + 20`. As prioridades observadas são as registradas neste laboratório; o comando `root primary` pode escolher uma prioridade diferente em outra topologia.

A prioridade define a referência da árvore. É o protocolo STP que evita loops ao selecionar caminhos e bloquear caminhos redundantes.

### Estado nominal comprovado

Nas VLANs 10 e 20, o root identificado foi `5000.0006.0000`, correspondente ao CORE1.

| Porta dos switches finais | Papel / estado | Custo até o root |
|---|---|---:|
| `Gi0/0` para CORE1 | `Root FWD` | 4 |
| `Gi0/2` para CORE2 | `Altn BLK` | Caminho alternativo |

Esse bloqueio é por VLAN. A presença de uma porta Alternate/Blocking nesse desenho é esperada.

### Durante a falha testada

Ao desativar `CORE1 Gi1/1`, o SW-GUEST selecionou `Gi0/2` como `Root FWD`, com custo 8. O CORE1 permaneceu root; o caminho alcançou esse root por meio do CORE2 e do trunk entre os cores.

Esse teste valida a substituição de um uplink. Não equivale à queda completa do CORE1 ou à troca do firewall que responde pelo VIP.

## Limites do desenho

- Um host ligado a um único switch de acesso continua dependente desse switch e de sua placa/cabo.
- Um único enlace entre cores foi usado. Não há EtherChannel, stack ou MLAG documentado.
- Existem dois caminhos externos desenhados, mas não há teste comprovando contingência de provedores.
- WANs com DHCP no laboratório não demonstram uma implantação CARP de WAN completa.
- A existência de VLANs não comprova regras de isolamento entre elas no pfSense.
- A disponibilidade do próprio host do EVE-NG não foi avaliada.
