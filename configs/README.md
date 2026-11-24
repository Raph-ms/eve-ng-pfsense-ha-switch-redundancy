# Configurações

## Base consolidada

Os arquivos em `base/` são blocos mínimos para aplicação pelo console em um laboratório com o mesmo mapa de portas. Consolidam o funcionamento comprovado antes da otimização. Não são backups completos e não removem configurações antigas que não estejam citadas.

- Criam as VLANs necessárias.
- Configuram os trunks nos dois lados.
- Associam a porta do cliente à VLAN correta.
- Definem os papéis de root.
- Mantêm as portas de operação habilitadas.

A porta da máquina foi mantida sem PortFast na base para distinguir o estado validado dos ajustes propostos depois do teste com perda de ping.

## Otimização proposta

Os arquivos em `otimizacao-proposta/` são deltas opcionais do estágio seguinte, ainda sem confirmação de aplicação ou medição posterior:

- Link-type ponto a ponto nos enlaces diretos entre switches.
- PortFast e BPDU Guard na porta de cliente.
- Restauração do uplink GUEST no CORE1 antes do salvamento.

Não copiar indiscriminadamente o mesmo bloco para todos os equipamentos. A VLAN, os papéis STP e as portas diferem.

## Depois de aplicar

```text
show interfaces status
show interfaces trunk
show vlan brief
show spanning-tree vlan 10
show spanning-tree vlan 20
copy running-config startup-config
```

Usar a VLAN correspondente em cada switch. Pressionar Enter no prompt do nome de arquivo de destino. Se surgir `% Invalid input`, registrar a linha e corrigir antes de prosseguir.
