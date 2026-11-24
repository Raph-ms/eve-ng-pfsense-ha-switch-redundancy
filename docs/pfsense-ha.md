# pfSense: HA, interfaces e DHCP

## Escopo do registro

O cluster já possuía configuração de HA antes da expansão de switching. Foram revisadas as associações de interfaces, as VLANs, os VIPs, o serviço DHCP e os logs de sincronização. Esta documentação registra o que foi observado e corrigido; não é uma exportação completa das configurações de ambos os firewalls.

## Mapeamento alinhado

Os nomes amigáveis podem ser iguais enquanto os identificadores internos apontam para redes diferentes. Para sincronização XMLRPC, os mesmos identificadores devem representar as mesmas funções nos dois nós.

| Identificador | Função | Porta / subinterface nos dois nós |
|---|---|---|
| `wan` | WAN / NET | `vtnet0` |
| `lan` | LAN original | `vtnet1` |
| `opt1` | SYNC | `vtnet2` |
| `opt2` | GERENCIA | `vtnet3` |
| `opt3` | CORP | VLAN 10 sobre `vtnet3` |
| `opt4` | GUEST | VLAN 20 sobre `vtnet3` |
| `opt5` | DEVICES | VLAN 30 sobre `vtnet3` |
| `opt6` | VIVO | `vtnet4` |

No EVE-NG, as ligações correspondentes são `e0`, `e1`, `e2`, `e3` e `e4`.

## Endereços registrados no secundário

Valores preservados do backup analisado em 18/11/2026:

| Função | Endereço / configuração |
|---|---|
| WAN | DHCP |
| LAN | `192.168.1.3/24` |
| SYNC | `172.16.1.2/24` |
| GERENCIA | `172.16.12.3/24` |
| CORP | `10.10.10.2/24` |
| GUEST | `10.20.10.3/24` |
| DEVICES | `10.30.10.1/24` |
| VIVO | DHCP |

Esses endereços individuais não são os gateways virtuais dos clientes. Não foi atribuído um VIP à DEVICES nas evidências analisadas; o endereço terminado em `.1` nessa interface era individual no backup.

O peer SYNC do primário foi registrado como `172.16.1.3`. O endereço individual CORP `10.10.10.3` apareceu nas ofertas DHCP do primário. Os demais endereços individuais do primário precisam ser conferidos em uma exportação atual antes de uma reprodução integral.

## Falha 1: pai das VLANs no secundário

As VLANs 10, 20 e 30 do secundário estavam sobre `vtnet4`, ligada à VIVO, enquanto o cabo para o switch chegava à `vtnet3`.

Correção pela GUI: **Interfaces → Assignments → VLANs**, alterando o Parent Interface para `vtnet3` em cada VLAN.

Essa alteração corrigiu o enlace físico utilizado pelas VLANs, mas não corrigiu a divergência dos identificadores internos.

## Falha 2: interfaces internas desalinhadas

| Identificador | Primário | Secundário antes | Secundário corrigido |
|---|---|---|---|
| `opt2` | GERENCIA | VIVO | GERENCIA |
| `opt3` | CORP | GERENCIA | CORP |
| `opt4` | GUEST | CORP | GUEST |
| `opt5` | DEVICES | GUEST | DEVICES |
| `opt6` | VIVO | DEVICES | VIVO |

A configuração DHCP copiada para `opt3` destinava-se à CORP, mas esse identificador representava a GERENCIA no secundário. Os VIPs também apareciam associados a funções diferentes nas telas dos dois nós.

### Evidência no protocolo

A captura detalhada mostrou:

1. DHCP Discover da VM.
2. DHCP Offer de `10.10.10.3`, oferecendo `10.10.10.100`.
3. DHCP Request do cliente selecionando esse servidor.
4. Repetições de Request, sem DHCP ACK na captura.

### Evidência no log Kea

Trecho relevante observado:

```text
HA_LEASE_UPDATE_CONFLICT
The address 10.10.10.100 does not belong to subnet 172.16.12.0/24,
subnet-id=2
```

O parceiro rejeitava a atualização da concessão porque associava aquele identificador a uma sub-rede diferente. Esse erro explica a negociação parada antes do ACK.

**`subnet-id=2` é um identificador do Kea. Não significa automaticamente VLAN 2 ou interface `opt2`.**

O status Kea `hot-standby` mostrava comunicação entre peers, mas não impedia rejeições de concessões por configurações incompatíveis.

## Correção do backup

Foi preparada uma cópia corrigida do backup do secundário:

- Os cinco blocos de interfaces `opt2` a `opt6` foram remapeados integralmente para corresponder ao primário.
- Os endereços individuais acompanharam suas funções.
- WAN, LAN e SYNC foram preservadas.
- As demais seções do XML foram preservadas.
- As referências DHCP e VIP já sincronizadas em `opt3` e `opt4` passaram a corresponder às interfaces corretas.
- A estrutura XML e a compatibilidade entre redes de interfaces, pools DHCP e VIPs foram verificadas antes da entrega.

A cópia foi restaurada **somente no secundário**. A etapa de restauração orientada foi **Diagnostics → Backup & Restore → Restore area: All**. Após a correção, o usuário confirmou que a VM recebeu IP.

O XML integral não é incluído neste repositório público: backups desse tipo podem conter credenciais, chaves e certificados. Não é seguro aplicar uma cópia deste laboratório em outro par sem revisar todos os endereços e referências.

## DHCP e VIPs

| Rede | Pool registrado | Gateway esperado | VHID registrado |
|---|---|---|---:|
| CORP | `10.10.10.100–10.10.10.150` | `10.10.10.1` | 2 |
| GUEST | `10.20.10.100–10.20.10.150` | `10.20.10.1` | 3 |
| LAN original | `192.168.1.11–192.168.1.245` | `192.168.1.1` | 1 |

Os pools GUEST e LAN são registros do backup analisado, não uma nova exportação ao final dos testes. O cliente GUEST recebeu `10.20.10.100` nas evidências posteriores.

Para cada VIP correspondente, ambos os firewalls precisam usar a mesma rede e VHID. O skew observado no backup do secundário era 100. A prioridade/estado de cada VIP deve ser conferida em **Status → CARP (failover)** ao validar o cluster.

Uma oferta anterior anunciava `10.10.10.3` como gateway. O resultado final da VM CORP mostrou o VIP `10.10.10.1`. O uso do VIP evita vincular o gateway distribuído ao endereço individual de um único firewall.

## Kea HA registrado

| Campo | Primário | Secundário |
|---|---|---|
| Papel | Primary | Standby |
| Nome | `pfsense-primary` | `pfsense-secondary` |
| Endereço de SYNC | `172.16.1.3` | `172.16.1.2` |
| Peer remoto | Secundário | Primário |

A atualização observada utilizou o endereço SYNC do secundário e a porta TCP 8765. Para reprodução, conferir **Services → DHCP Server → Settings**, os nomes recíprocos, os endereços reais de SYNC, a comunicação na porta configurada e eventuais opções TLS. Não usar o VIP CARP como endereço do peer Kea.

## Cuidados de interpretação

- A captura exibiu anúncios do protocolo 112 como VRRP. No contexto do laboratório pfSense, a decodificação deve ser ajustada para CARP antes de interpretar campos como endereços e prioridade.
- IPs aparentemente aleatórios mostrados pelo decodificador VRRP não foram tratados como configuração real.
- Uma concessão aparecer na base do servidor não prova, isoladamente, que o cliente recebeu um ACK.
- O DHCP deve estar habilitado na interface VLAN correspondente, não apenas na LAN original.
