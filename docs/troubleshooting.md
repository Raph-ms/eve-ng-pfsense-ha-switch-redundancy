# Troubleshooting: evidências, causas e correções

## 1. DHCP bloqueado pela incompatibilidade entre os peers

**Sintoma:** o cliente enviava Discover, recebia Offer e repetia Request sem ACK.

**Evidência:** `HA_LEASE_UPDATE_CONFLICT` informando que `10.10.10.100` não pertencia a `172.16.12.0/24` no identificador recebido.

**Causa comprovada:** o mesmo identificador de interface representava redes diferentes nos dois pfSense; o secundário rejeitava a atualização da concessão.

**Correção:** alinhar o pai das VLANs e remapear os blocos de interface do secundário para corresponder ao primário, preservando os IPs individuais e as referências compatíveis já sincronizadas.

**Resultado:** entrega DHCP confirmada após a restauração da cópia corrigida.

Consultar [pfSense HA](pfsense-ha.md) para a matriz antes/depois.

## 2. VLANs sobre a interface física errada no secundário

**Sintoma/configuração:** VLANs 10, 20 e 30 sobre `vtnet4`; enlace do switch em `vtnet3`.

**Correção:** alterar o Parent Interface das VLANs para `vtnet3`.

**Lição:** uma tag correta sobre o enlace errado não chega ao switch pretendido. Essa correção foi necessária, mas não resolveu sozinha a divergência dos identificadores internos do HA.

## 3. Cabos conectados a portas diferentes das configuradas

Foram identificadas três divergências no desenho intermediário:

| Ligação | Desenho intermediário | Cabeamento final |
|---|---|---|
| VM CORP | SW-CORP `Gi0/0` | SW-CORP `Gi0/1` |
| CORE1–CORP | CORE1 `Gi0/0` ↔ CORP `Gi0/3` | CORE1 `Gi0/3` ↔ CORP `Gi0/0` |
| CORE2–GUEST | CORE2 `Gi0/2` ↔ GUEST `Gi1/0` | CORE2 `Gi1/0` ↔ GUEST `Gi0/2` |

**Correção:** mover os cabos para as portas correspondentes ao plano. O CDP confirmou posteriormente o cabeamento real no CORE1.

**Lição:** o desenho deve corresponder às interfaces que receberam os comandos.

## 4. Cliente CORP na VLAN 1

**Sintoma:** IP APIPA `169.254.x.x`, sem gateway e sem DHCP.

**Evidência:** `show interfaces status` mostrou `Gi0/1 connected 1`. A VLAN 10 existia, os uplinks permitiam apenas 10 e a MAC da máquina não aparecia na tabela dinâmica dessa VLAN.

**Causa comprovada:** a porta da VM pertencia à VLAN 1. O tráfego de DHCP não ingressava na VLAN 10 que os uplinks transportavam.

**Correção efetiva:**

```text
enable
configure terminal
interface GigabitEthernet0/1
 switchport mode access
 switchport access vlan 10
 no shutdown
end
copy running-config startup-config
```

**Resultado:** `Gi0/1 connected 10`; cliente `10.10.10.100/24`, gateway `10.10.10.1`.

Uma declaração intermediária de que o cliente havia recebido IP foi corrigida pelo próprio teste posterior. O estado de sucesso aqui considerado é o print final com IP e gateway, não aquela declaração intermediária.

## 5. GUEST sem transporte da VLAN 20 nos uplinks

**Sintoma inicial:** `Spanning tree instance(s) for vlan 20 does not exist`.

Após criar/ativar a VLAN 20 e associar a máquina à `Gi0/1`, o DHCP continuou sem resposta.

**Evidência seguinte:**

- `Gi0/1 connected 20`.
- `Gi0/0` e `Gi0/2` ainda na VLAN 1.
- `show interfaces trunk` sem portas.
- SW-GUEST se elegendo root da VLAN 20 e mostrando apenas a porta da máquina nessa instância.

**Causa comprovada nessa etapa:** os uplinks do SW-GUEST não transportavam a VLAN 20.

**Correção:** configurar trunks permitindo VLAN 20 em `SW-GUEST Gi0/0` e `Gi0/2`, e nos pares `CORE1 Gi1/1` e `CORE2 Gi1/0`.

**Resultado:** cliente `10.20.10.100/24`, gateway `10.20.10.1`.

## 6. DHCP funcionando, mas caminho direto ao CORE1 incompleto

**Sintoma:** CORP e GUEST alcançavam o root pela `Gi0/2`, com custo 8; a `Gi0/0` aparecia como Designated/Forwarding, em vez do caminho direto esperado.

**Investigação:**

```text
show cdp neighbors
show interfaces trunk
show running-config interface GigabitEthernet0/3
show running-config interface GigabitEthernet1/1
```

**Evidência:** CDP com vizinhos e portas corretos; `CORE1 Gi0/3` e `Gi1/1` ausentes da relação de trunks.

**Correção:** configurar os trunks no lado do CORE1: VLAN 10 na `Gi0/3` e VLAN 20 na `Gi1/1`.

**Resultado:** os dois acessos passaram a usar `Gi0/0 Root FWD`, custo 4, e `Gi0/2 Altn BLK`.

**Lição:** obter DHCP por um dos caminhos não comprova que o segundo uplink está funcional.

## 7. Convergência com interrupção perceptível

**Teste:** shutdown da `CORE1 Gi1/1` durante ping contínuo da GUEST para seu VIP.

**Evidência:** `SW-GUEST Gi0/2 Root FWD`, custo 8, perdas de ping seguidas de retorno das respostas. Durante a transição, a `Gi0/1` apareceu em Learning e as portas com tipo Shared.

**Resultado comprovado:** recuperação do caminho.

**Hipótese técnica para a demora:** enlaces tratados como compartilhados limitam as transições rápidas do RSTP; a porta da VM em aprendizado também pode prolongar a interrupção. Não há cronometragem nem dados suficientes para atribuir cada segundo da perda a um único fator.

**Ajuste proposto:** links ponto a ponto e PortFast/BPDU Guard somente na borda. A validação após esses comandos permanece pendente.

## 8. Erros de operação do console

- `show cdp neigbors`: erro de digitação; o comando correto é `show cdp neighbors`.
- `show running-config` em `CORE1>`: usar `enable` antes.
- `--More--`: paginação, não fim da configuração. Usar Espaço ou `terminal length 0`.
- Conferir a saída atual após cada alteração; uma instrução fornecida não é evidência de que foi aplicada.

## Sequência prática de diagnóstico

1. Verificar porta física da máquina e sua VLAN access.
2. Conferir se a VLAN existe e está ativa.
3. Conferir trunks, VLANs permitidas e VLANs ativas nos dois lados.
4. Conferir CDP e tabela MAC.
5. Verificar STP e caminhos até o root.
6. Capturar DHCP na interface VLAN do pfSense.
7. Identificar Discover, Offer, Request e ACK.
8. Consultar logs do Kea e compatibilidade dos peers.
9. Validar IP, máscara e gateway no cliente.
10. Executar um teste de falha controlado e restaurar o estado nominal.
