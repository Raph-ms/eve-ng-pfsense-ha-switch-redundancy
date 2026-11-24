# Implementação dos switches

## Pré-requisitos

- Quatro nós Cisco IOSv Layer 2, com as portas previstas no README.
- Console de cada equipamento identificado pelo hostname.
- VLANs dos pfSense sobre `vtnet3`, com o mapeamento HA alinhado.
- Cópias privadas das configurações antes de alterações.
- Links redundantes adicionados depois da configuração dos dois lados e do STP.

## Acesso ao modo correto

```text
enable
terminal length 0
show running-config
```

O prompt `>` indica modo de usuário. Use `enable` para chegar ao prompt `#`, necessário para comandos como `show running-config` e para entrar em configuração. `terminal length 0` remove a paginação; alternativamente, use Espaço para avançar em `--More--`.

## Arquivos base

| Equipamento | Arquivo |
|---|---|
| CORE1 | [CORE1.cfg](../configs/base/CORE1.cfg) |
| CORE2 | [CORE2.cfg](../configs/base/CORE2.cfg) |
| SW-CORP | [SW-CORP.cfg](../configs/base/SW-CORP.cfg) |
| SW-GUEST | [SW-GUEST.cfg](../configs/base/SW-GUEST.cfg) |

Os arquivos consolidam somente as interfaces e os recursos necessários ao caminho documentado. Não são dumps integrais de running-config. Uma porta antiga do CORE1, `Gi0/2`, ainda apareceu como trunk nas capturas; ela não faz parte do cabeamento final e foi omitida dessas configurações mínimas. Não há instrução aqui para apagá-la em um equipamento existente.

As portas não citadas não foram padronizadas como portas de usuários. Neste laboratório, as VMs devem usar `Gi0/1` dos switches de acesso.

## Ordem de implementação

1. Aplicar as configurações de CORE1 e CORE2 pelo console.
2. Conferir o trunk `Gi1/3` entre os cores e o reconhecimento do root.
3. Configurar SW-CORP e SW-GUEST.
4. Conferir os trunks correspondentes nos dois lados.
5. Conectar os uplinks de acordo com o mapa.
6. Renovar DHCP e conferir os endereços/gateways.
7. Validar o estado nominal do STP.
8. Salvar a configuração de cada equipamento.
9. Realizar os testes de falha separadamente, com retorno ao estado nominal.

A alteração de trunks em enlaces já conectados pode causar uma interrupção transitória enquanto os lados são alinhados.

## Salvar e conferir

```text
end
copy running-config startup-config
```

Quando surgir `Destination filename [startup-config]?`, pressionar Enter.

Conferir imediatamente os comandos aplicados. A confirmação de salvamento não substitui a inspeção das portas e das VLANs. O teste de persistência após reiniciar todos os nós não foi registrado.

## Estado nominal esperado

Nos cores:

```text
show interfaces trunk
show vlan brief
show cdp neighbors
show spanning-tree vlan 10
show spanning-tree vlan 20
```

O CDP do CORE1 confirmou:

| Interface local | Vizinho / porta remota |
|---|---|
| `Gi0/3` | SW-CORP `Gi0/0` |
| `Gi1/1` | SW-GUEST `Gi0/0` |
| `Gi1/3` | CORE2 `Gi1/3` |

Nos switches de acesso:

```text
show interfaces status
show interfaces trunk
show vlan brief
show mac address-table dynamic vlan 10
```

No GUEST, usar `vlan 20` no comando da tabela MAC e no STP.

| Porta de acesso | CORP | GUEST |
|---|---|---|
| `Gi0/1` | Connected, access VLAN 10 | Connected, access VLAN 20 |
| `Gi0/0` | Trunk 10 / Root FWD | Trunk 20 / Root FWD |
| `Gi0/2` | Trunk 10 / Altn BLK | Trunk 20 / Altn BLK |

Uma porta trunk não costuma aparecer como porta access na linha da VLAN em `show vlan brief`. Conferir `show interfaces trunk` para os uplinks.

## Otimização proposta: reduzir a interrupção

**Status: comandos fornecidos, aplicação e novo teste ainda não confirmados.**

No teste, os enlaces apareceram como `Shr` e a porta da VM entrou em `LRN`. Foram propostos:

1. `spanning-tree link-type point-to-point` nos dois lados dos enlaces diretos entre switches.
2. `spanning-tree portfast` na porta da máquina.
3. `spanning-tree bpduguard enable` na porta da máquina.

| Equipamento | Enlaces entre switches a ajustar |
|---|---|
| CORE1 | `Gi0/3`, `Gi1/1`, `Gi1/3` |
| CORE2 | `Gi0/3`, `Gi1/0`, `Gi1/3` |
| SW-CORP | `Gi0/0`, `Gi0/2` |
| SW-GUEST | `Gi0/0`, `Gi0/2` |

Os arquivos de [otimização proposta](../configs/otimizacao-proposta/) são deltas aplicados sobre a base, não configurações completas.

No laboratório, esses enlaces são conexões diretas entre dois switches. Em outro ambiente, verificar se o meio é realmente ponto a ponto antes de forçar o tipo de enlace. PortFast/BPDU Guard de borda não devem ser aplicados aos uplinks entre switches.

Após os ajustes, esperar `P2p` nos uplinks e indicação de `Edge` na porta de cliente. Repetir o teste e medir a indisponibilidade; não foi estabelecida uma meta comprovada de convergência em segundos.

O delta do CORE1 inclui `no shutdown` na `Gi1/1` para restaurar o uplink usado no teste antes de salvar.
