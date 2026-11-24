# Plano de testes e resultados

## Matriz de evidências

| ID | Teste | Estado | Evidência / observação |
|---|---|---|---|
| T01 | DHCP CORP | Aprovado | `10.10.10.100/24`, gateway `10.10.10.1` |
| T02 | DHCP GUEST | Aprovado | `10.20.10.100/24`, gateway `10.20.10.1` |
| T03 | STP nominal CORP | Aprovado | Root CORE1, custo 4, Gi0/0 Root FWD e Gi0/2 Altn BLK |
| T04 | STP nominal GUEST | Aprovado | Root CORE1, custo 4, Gi0/0 Root FWD e Gi0/2 Altn BLK |
| T05 | Perda do uplink GUEST–CORE1 | Recuperação comprovada | Gi0/2 Root FWD, custo 8, ping voltou após perdas |
| T06 | Retorno ao estado nominal após o último teste | Pendente de evidência | Comando de restauração fornecido |
| T07 | Otimização P2p / PortFast e repetição de T05 | Proposto | Sem resultado posterior recebido |
| T08 | Perda do uplink CORP–CORE1 | Pendente | Repetir com cliente na CORP |
| T09 | Queda completa do CORE1 | Pendente | Validar STP, CARP e conectividade |
| T10 | Queda completa de um pfSense | Pendente | Validar VIPs, DHCP e sessões |
| T11 | Queda de NET ou VIVO | Pendente | Depende da configuração Multi-WAN |
| T12 | Persistência após reiniciar os nós | Pendente | Conferir running/startup e VLANs |
| T13 | Isolamento da GUEST e regras entre VLANs | Pendente | Não foi auditado |
| T14 | Cliente e DHCP da DEVICES | Pendente | VLAN 30 criada; sem cliente validado |

Uma concessão de DHCP confirma a negociação do endereço, não acesso à Internet nem todas as políticas de firewall.

## T01/T02 — Cliente

Conectar a VM à `Gi0/1` do switch em teste, sem tag VLAN configurada na placa e com IPv4 automático.

```bat
ipconfig /renew
ipconfig /all
```

| Rede | Endereço esperado segundo o pool registrado | Gateway |
|---|---|---|
| CORP | `10.10.10.100–10.10.10.150/24` | `10.10.10.1` |
| GUEST | `10.20.10.100–10.20.10.150/24` | `10.20.10.1` |

Um endereço `169.254.x.x` indica que o Windows não obteve a concessão nesse teste. Se houver uma renovação bloqueada na tela, interromper com Ctrl+C antes de repetir.

## T03/T04 — Estado nominal

SW-CORP:

```text
enable
show spanning-tree vlan 10
show interfaces trunk
show interfaces status
```

SW-GUEST:

```text
enable
show spanning-tree vlan 20
show interfaces trunk
show interfaces status
```

Verificar root `5000.0006.0000`, custo 4, uplink para CORE1 Root/Forwarding e uplink para CORE2 Alternate/Blocking. O identificador MAC do root vale para este laboratório, não para imagens recriadas.

## T05 — Falha controlada do uplink da GUEST

### Condições

- Cliente com IP correto e ping inicial respondendo.
- Dois cores e respectivos trunks operacionais.
- Acesso aos consoles para restaurar a porta.
- Somente uma falha induzida por vez.

### Procedimento realizado

Na VM GUEST:

```bat
ping 10.20.10.1 -t
```

No CORE1:

```text
enable
configure terminal
interface GigabitEthernet1/1
 shutdown
end
```

No SW-GUEST:

```text
show spanning-tree vlan 20
```

### Resultado observado

- CORE1 permaneceu root da VLAN 20.
- A root port do SW-GUEST passou para `Gi0/2`.
- O custo até o root passou de 4 para 8.
- Ocorreram timeouts e uma mensagem de destino inacessível na captura de ping.
- As respostas de `10.20.10.1` retornaram.

A captura mostra recuperação do uplink pelo caminho alternativo. Não comprova mudança de MASTER do CARP. Nenhum tempo exato de indisponibilidade foi medido.

### Restauração obrigatória antes de outro teste

No CORE1:

```text
configure terminal
interface GigabitEthernet1/1
 no shutdown
end
```

Conferir novamente `show spanning-tree vlan 20` no SW-GUEST. Esperar Gi0/0 Root FWD e Gi0/2 Altn BLK após convergir.

Não salvar o estado temporário com a porta em shutdown. Salvar a configuração de operação depois de restaurar e conferir.

## T07 — Otimização e medição

Os deltas em `configs/otimizacao-proposta/` ainda não têm validação posterior registrada.

1. Confirmar o retorno ao estado nominal.
2. Aplicar os deltas apropriados em cada equipamento.
3. Conferir `P2p` nos enlaces entre switches e `Edge` nas portas dos clientes.
4. Repetir T05 com as mesmas condições.
5. Registrar horário de início e fim da indisponibilidade ou utilizar captura com timestamps.
6. Comparar a perda com o teste anterior; registrar também carga/recursos do EVE-NG.
7. Restaurar o uplink e salvar somente o estado nominal.

Não usar apenas a quantidade visual de linhas de ping para afirmar uma convergência em segundos: o tempo de espera de cada ping pode variar.

## T08 — Falha do uplink CORP

Procedimento equivalente a T05, com a VM em SW-CORP Gi0/1, ping para `10.10.10.1` e falha em `CORE1 Gi0/3`. Restaurar essa mesma porta com `no shutdown`.

## T09/T10 — Queda completa de core ou firewall

Roteiro planejado, sem resultado registrado:

1. Confirmar DHCP, gateway e ping nas redes em teste.
2. Registrar estados dos VIPs em ambos os pfSense.
3. Registrar root ports dos switches de acesso.
4. Parar somente o equipamento selecionado no EVE-NG.
5. Acompanhar recuperação de ping, STP e estados CARP.
6. Se for avaliar DHCP em contingência, usar cliente de teste e registrar a renovação, não apenas uma concessão antiga.
7. Se for avaliar pfsync, manter uma sessão de aplicação e acompanhar sua continuidade.
8. Religar o equipamento, conferir retorno estável e registrar o resultado.

Aprovação exige observação do tráfego e dos estados; a presença dos cabos no desenho não basta.

## Registro de novos testes

| Data / hora e fuso | Teste | Falha aplicada | Tempo medido | Resultado | Evidência |
|---|---|---|---|---|---|
| A preencher | T07–T14 | A preencher | A preencher | A preencher | A preencher |

As telas dos equipamentos exibiram UTC e avançaram para 24/11 em parte dos testes. Isso pode corresponder à noite de 23/11 em America/Sao_Paulo. Registrar o fuso nos próximos testes para permitir comparação.
