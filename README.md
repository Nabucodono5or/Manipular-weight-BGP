# Manipular-weight-BGP

## **Descrição**
Configuração usando roteadores Cisco, onde temos dois links redundantes que aprendem rotas do AS 65009 e usa como AS de transito o AS 65002.

Foram configurados como rotas preferencias de comunicação o link  `f0/0`  saindo do `RT10`, nosso router que está aprendendo os endereços do AS 65009 e também está divulgando os seus endereços de forma local. 

Para a configuração foi configurado weight para que as rotas aprendidas direcionassem a saída para o link principal do router RT10, e para que as rotas divulgadas deste roteador fossem comunicada através do link principal, configuramos um BGP prepend.

Por fim, foram configurados filtros através de `route-map`, para que somente fossem divulgadas as rotas locais e não o tornasse o roteador RT10 em um caminho transitório para o AS 65009, assim como foram configurados que somente alguns endereços fossem divulgados externamente.


## **Topologia**

Segue a topologia utilizada.

![[Pasted image 20250317102935.png]]
![topologia](Pasted%20image%2020250317102935.png)

## **Endereçamento**

Endereçamento ipv4 utilizado na topologia.

| DISPOSITIVO | INTERFACE | ENDEREÇO IPV4  |
| ----------- | --------- | -------------- |
| RT10        | F0/0      | 172.16.1.1/30  |
| RT10        | F0/1      | 172.16.2.1/30  |
| RT10        | Loopback0 | 10.1.1.1/24    |
| RT10        | Loopback1 | 10.2.10.1/24   |
| RT10        | Loopback2 | 200.19.19.1/32 |
| RT10        | Loopback3 | 200.20.20.1/32 |
| RT10        | Loopback4 | 200.21.21.1/32 |
| RT10        | Loopback5 | 192.168.5.1/32 |
| RT11        | F0/0      | 172.16.2.2/30  |
| RT11        | F1/0      | 192.168.1.1/30 |
| RT11        | F0/1      | 10.1.1.2/30    |
| RT12        | F0/0      | 10.2.2.2/30    |
| RT12        | F0/1      | 172.16.2.2/30  |
| RT12        | F1/0      | 192.168.1.2/30 |
| RT12        | Loopback2 | 6.6.6.6/32     |
| RT13        | F0/0      | 10.2.2.1/30    |
| RT13        | F0/1      | 10.1.1.1/30    |
| RT13        | Loopback0 | 8.8.8.8/32     |
| RT13        | Loopback1 | 8.8.9.9/32     |

## **Configuração**

Seguem as configurações de interfaces e roteamento dos roteadores, assim como também os principais filtros.

### RT13

```julia
conf t
hostname RT13
interface FastEthernet 0/1
description RT13 to AS65002
ip address 10.1.1.1 255.255.255.252
no shut
exit
interface FastEthernet 0/0
description RT13 to AS65002 BKP
ip address 10.2.2.1 255.255.255.252
no shut
exit
interface loopback 0
ip address 8.8.8.8 255.255.255.255
no shut
exit
interface loopback 1
ip address 8.8.9.9 255.255.255.255
no shut
exit
router bgp 65009
bgp router-id 9.9.9.9
network 10.1.1.0 mask 255.255.255.252
network 10.2.2.0 mask 255.255.255.252
network 8.8.8.8 mask 255.255.255.255
network 8.8.9.9 mask 255.255.255.255
exit
ip prefix-list REDE-LOCAL seq 10 permit 8.8.8.8/32
ip prefix-list REDE-LOCAL seq 20 permit 8.8.9.9/32
route-map BLOQUEIO-REDE-LOCAL-OUT deny 10
match ip address prefix-list REDE-LOCAL
exit
route-map BLOQUEIO-REDE-LOCAL-OUT permit 20
end

```

O **RT13** arqu faz o papel de um ISP que está fornecendo redes de acesso a internet, essas redes são as que estão configuradas como loopback. 
Veja que também fora configurado filtros para que o roteador não divulgasse redes que não fosse dele.

### RT11

```julia
conf t
hostname RT11
interface FastEthernet 0/1
description RT11 to AS65009
ip address 10.1.1.2 255.255.255.252
no shut
exit
interface FastEthernet 0/0
description RT11 to AS65001
ip address 172.16.1.2 255.255.255.252
no shut
exit
interface FastEthernet 1/0
description RT11 to RT12
ip address 192.168.1.1 255.255.255.252
no shut
exit
router bgp 65002
bgp router-id 2.2.2.2
network 192.168.1.0 mask 255.255.255.255
neighbor 192.168.1.2 remote-as 65002
neighbor 10.1.1.1 remote-as 65009
neighbor 172.16.1.1 remote-as 65001
end

```

No `RT11`, apesar de termos o ip `192.168.1.1` na interface `f1/0` nada fora configurado nele em questão de roteamento. Mas a ideia é usar um **IGP** para o compartilhamento de rotas, porém em razão do perigo de perda do atributo **AS_PATH**, isso foi descartado para esse projeto. 

### RT12

```julia
conf t
hostname RT12
interface FastEthernet 0/1
description RT12 to AS65001
ip address 172.16.2.2 255.255.255.252
no shut
exit
interface FastEthernet 0/0
description RT12 to AS65009
ip address 10.2.2.2 255.255.255.252
no shut
exit
interface FastEthernet 1/0
description RT12 to RT11
ip address 192.168.1.2 255.255.255.252
no shut
exit
router bgp 65002
bgp router-id 2.2.2.2
network 192.168.1.0 mask 255.255.255.255
neighbor 192.168.1.1 remote-as 65002
neighbor 10.2.2.1 remote-as 65009
neighbor 172.16.2.1 remote-as 65001
end
```

O mesmo com relação ao RT11 se declara aqui.

### RT10

```julia
conf t
hostname RT10
interface FastEthernet 0/1
description RT10 to AS65002
ip address 172.16.2.1 255.255.255.252
no shut
exit
interface FastEthernet 0/0
description RT10 to AS65002
ip address 172.16.1.1 255.255.255.252
no shut
exit
ip prefix-list REDE-LOCAL seq 10 deny 192.168.0.0/16 ge 17
ip prefix-list REDE-LOCAL seq 15 deny 10.0.0.0/8
ip prefix-list REDE-LOCAL seq 20 deny 10.0.0.8/8 ge 9
ip prefix-list REDE-LOCAL seq 25 deny 8.8.8.8/32
ip prefix-list REDE-LOCAL seq 30 deny 8.8.9.9/32
ip prefix-list REDE-LOCAL seq 90 permit 0.0.0.0/0 le 32
route-map BLOQUEIO-REDEBKP-LOCAL-OUT permit 10
match ip address prefix-list REDE-LOCAL
set as-path prepend 65001 65001
exit
route-map BLOQUEIO-REDEPRI-LOCAL-OUT permit 10
match ip address prefix-list REDE-LOCAL
exit
route-map REDE-FORA-PRI-IN permit 10
set weight 3000
exit
router bgp 65001
bgp router-id 1.1.1.1
network 172.16.1.0 mask 255.255.255.252
network 172.16.2.0 mask 255.255.255.252
neighbor 172.16.2.2 remote-as 65002
neighbor 172.16.2.2 route-map BLOQUEIO-REDEBKP-LOCAL-OUT out
neighbor 172.16.1.2 remote-as 65002
neighbor 172.16.1.2 route-map BLOQUEIO-REDEPRI-LOCAL-OUT out
neighbor 172.16.1.2 route-map REDE-FORA-PRI-IN in
aggregate-address 192.168.0.0 255.255.0.0 summary-only
end


conf t
router bgp 65001
network 10.0.0.0 mask 255.0.0.0
network 200.19.19.1 mask 255.255.255.255
exit
interface loopback 0
ip address 10.1.1.1 255.255.255.0
no shut
exit
interface loopback 1
ip address 10.2.10.1 255.255.255.0
no shut
exit
interface loopback 2
ip address 200.19.19.1 255.255.255.255
no shut
end

conf t
interface loopback 3
ip address 200.20.20.1 255.255.255.255
no shut
exit
interface loopback 4
ip address 200.21.21.1 255.255.255.255
no shut
exit
interface loopback 5
ip address 192.168.1.5 255.255.255.0
no shut
end

```


## **Teste e Validação**

Segue os comando e as saídas esperadas:

### RT13

```julia
RT13#sho ip bgp summary | begin Neigh
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.1.1.2        4 65002     212     212       10    0    0 03:27:50        5
10.2.2.2        4 65002     212     212       10    0    0 03:27:42        5

RT13#sh run | se bgp
router bgp 65009
 no synchronization
 bgp router-id 9.9.9.9
 bgp log-neighbor-changes
 network 8.8.8.8 mask 255.255.255.255
 network 8.8.9.9 mask 255.255.255.255
 network 10.1.1.0 mask 255.255.255.252
 network 10.2.2.0 mask 255.255.255.252
 neighbor 10.1.1.2 remote-as 65002
 neighbor 10.1.1.2 route-map BLOQUEIO-REDE-LOCAL-OUT in
 neighbor 10.2.2.2 remote-as 65002
 neighbor 10.2.2.2 route-map BLOQUEIO-REDE-LOCAL-OUT in
 no auto-summary

RT13#sh route-map 
route-map BLOQUEIO-REDE-LOCAL-OUT, deny, sequence 10
  Match clauses:
    ip address prefix-lists: REDE-LOCAL 
  Set clauses:
  Policy routing matches: 0 packets, 0 bytes
route-map BLOQUEIO-REDE-LOCAL-OUT, permit, sequence 20
  Match clauses:
  Set clauses:
  Policy routing matches: 0 packets, 0 bytes


RT13#sh ip prefix-list 
ip prefix-list REDE-LOCAL: 2 entries
   seq 10 permit 8.8.8.8/32
   seq 20 permit 8.8.9.9/32

RT13#sh ip bgp neighbors 10.1.1.2 advertised-routes 
BGP table version is 10, local router ID is 9.9.9.9
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale
Origin codes: i - IGP, e - EGP, ? - incomplete

   Network          Next Hop            Metric LocPrf Weight Path
*> 8.8.8.8/32       0.0.0.0                  0         32768 i
*> 8.8.9.9/32       0.0.0.0                  0         32768 i
*> 10.1.1.0/30      0.0.0.0                  0         32768 i
*> 10.2.2.0/30      0.0.0.0                  0         32768 i
*> 172.16.1.0/30    10.1.1.2                               0 65002 65001 i
*> 172.16.2.0/30    10.1.1.2                               0 65002 65001 i
*> 200.19.19.1/32   10.1.1.2                               0 65002 65001 i
*> 200.20.20.1/32   10.1.1.2                               0 65002 65001 i
*> 200.21.21.1/32   10.1.1.2                               0 65002 65001 i

Total number of prefixes 9 
RT13#sh ip bgp neighbors 10.2.2.2 advertised-routes 
BGP table version is 10, local router ID is 9.9.9.9
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale
Origin codes: i - IGP, e - EGP, ? - incomplete

   Network          Next Hop            Metric LocPrf Weight Path
*> 8.8.8.8/32       0.0.0.0                  0         32768 i
*> 8.8.9.9/32       0.0.0.0                  0         32768 i
*> 10.1.1.0/30      0.0.0.0                  0         32768 i
*> 10.2.2.0/30      0.0.0.0                  0         32768 i
*> 172.16.1.0/30    10.1.1.2                               0 65002 65001 i
*> 172.16.2.0/30    10.1.1.2                               0 65002 65001 i
*> 200.19.19.1/32   10.1.1.2                               0 65002 65001 i
*> 200.20.20.1/32   10.1.1.2                               0 65002 65001 i
*> 200.21.21.1/32   10.1.1.2                               0 65002 65001 i

Total number of prefixes 9 

```

### RT11

```julia
RT11#sh run | se bgp
router bgp 65002
 no synchronization
 bgp router-id 2.2.2.2
 bgp log-neighbor-changes
 network 192.168.1.0 mask 255.255.255.255
 neighbor 10.1.1.1 remote-as 65009
 neighbor 172.16.1.1 remote-as 65001
 no auto-summary

RT11#sh ip bgp summary | begin Nei
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.1.1.1        4 65009     215     215       12    0    0 03:30:45        4
172.16.1.1      4 65001     220     222       12    0    0 03:29:59        5

RT11#sh ip bgp 
BGP table version is 12, local router ID is 2.2.2.2
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale
Origin codes: i - IGP, e - EGP, ? - incomplete

   Network          Next Hop            Metric LocPrf Weight Path
*> 8.8.8.8/32       10.1.1.1                 0             0 65009 i
*> 8.8.9.9/32       10.1.1.1                 0             0 65009 i
r> 10.1.1.0/30      10.1.1.1                 0             0 65009 i
*> 10.2.2.0/30      10.1.1.1                 0             0 65009 i
r> 172.16.1.0/30    172.16.1.1               0             0 65001 i
*> 172.16.2.0/30    172.16.1.1               0             0 65001 i
*> 200.19.19.1/32   172.16.1.1               0             0 65001 i
*> 200.20.20.1/32   172.16.1.1               0             0 65001 i
*> 200.21.21.1/32   172.16.1.1               0             0 65001 i


RT11#sh ip bgp neighbors 172.16.1.1 routes 
BGP table version is 12, local router ID is 2.2.2.2
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale
Origin codes: i - IGP, e - EGP, ? - incomplete

   Network          Next Hop            Metric LocPrf Weight Path
r> 172.16.1.0/30    172.16.1.1               0             0 65001 i
*> 172.16.2.0/30    172.16.1.1               0             0 65001 i
*> 200.19.19.1/32   172.16.1.1               0             0 65001 i
*> 200.20.20.1/32   172.16.1.1               0             0 65001 i
*> 200.21.21.1/32   172.16.1.1               0             0 65001 i

```

### RT12

```julia
RT12#sh ip bgp summary | begin Neig
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.2.2.1        4 65009     217     217       12    0    0 03:32:32        4
172.16.2.1      4 65001     219     217       12    0    0 03:31:28        5

RT12#sh ip bgp 
BGP table version is 12, local router ID is 2.2.2.2
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale
Origin codes: i - IGP, e - EGP, ? - incomplete

   Network          Next Hop            Metric LocPrf Weight Path
*> 8.8.8.8/32       10.2.2.1                 0             0 65009 i
*> 8.8.9.9/32       10.2.2.1                 0             0 65009 i
*> 10.1.1.0/30      10.2.2.1                 0             0 65009 i
r> 10.2.2.0/30      10.2.2.1                 0             0 65009 i
*> 172.16.1.0/30    172.16.2.1               0             0 65001 65001 65001 i
r> 172.16.2.0/30    172.16.2.1               0             0 65001 65001 65001 i
*> 200.19.19.1/32   172.16.2.1               0             0 65001 65001 65001 i
*> 200.20.20.1/32   172.16.2.1               0             0 65001 65001 65001 i
*> 200.21.21.1/32   172.16.2.1               0             0 65001 65001 65001 i


RT12#sh run | se bgp
router bgp 65002
 no synchronization
 bgp router-id 2.2.2.2
 bgp log-neighbor-changes
 network 192.168.1.0 mask 255.255.255.255
 neighbor 10.2.2.1 remote-as 65009
 neighbor 172.16.2.1 remote-as 65001
 no auto-summary


RT12#sh ip bgp neighbors 172.16.2.1 routes 
BGP table version is 12, local router ID is 2.2.2.2
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale
Origin codes: i - IGP, e - EGP, ? - incomplete

   Network          Next Hop            Metric LocPrf Weight Path
*> 172.16.1.0/30    172.16.2.1               0             0 65001 65001 65001 i
r> 172.16.2.0/30    172.16.2.1               0             0 65001 65001 65001 i
*> 200.19.19.1/32   172.16.2.1               0             0 65001 65001 65001 i
*> 200.20.20.1/32   172.16.2.1               0             0 65001 65001 65001 i
*> 200.21.21.1/32   172.16.2.1               0             0 65001 65001 65001 i

Total number of prefixes 5 

```

### RT10

```julia

RT10#sh ip bgp summary | begin Neig
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
172.16.1.2      4 65002     226     224       29    0    0 03:33:24        4
172.16.2.2      4 65002     218     220       29    0    0 03:32:57        4

RT10#sh run | se bgp
router bgp 65001
 no synchronization
 bgp router-id 1.1.1.1
 bgp log-neighbor-changes
 network 10.0.0.0
 network 172.16.1.0 mask 255.255.255.252
 network 172.16.2.0 mask 255.255.255.252
 network 192.168.0.0 mask 255.255.0.0
 network 192.168.5.0
 network 200.19.19.1 mask 255.255.255.255
 network 200.20.20.1 mask 255.255.255.255
 network 200.21.21.1 mask 255.255.255.255
 aggregate-address 192.168.0.0 255.255.0.0 summary-only
 neighbor 172.16.1.2 remote-as 65002
 neighbor 172.16.1.2 route-map REDE-FORA-PRI-IN in
 neighbor 172.16.1.2 route-map BLOQUEIO-REDEPRI-LOCAL-OUT out
 neighbor 172.16.2.2 remote-as 65002
 neighbor 172.16.2.2 route-map BLOQUEIO-REDEBKP-LOCAL-OUT out
 no auto-summary

RT10#sh route-map 
route-map BLOQUEIO-REDEBKP-LOCAL-OUT, permit, sequence 10
  Match clauses:
    ip address prefix-lists: REDE-LOCAL 
  Set clauses:
    as-path prepend 65001 65001
  Policy routing matches: 0 packets, 0 bytes
route-map REDE-FORA-PRI-IN, permit, sequence 10
  Match clauses:
  Set clauses:
    weight 3000
  Policy routing matches: 0 packets, 0 bytes
route-map BLOQUEIO-REDEPRI-LOCAL-OUT, permit, sequence 10
  Match clauses:
    ip address prefix-lists: REDE-LOCAL 
  Set clauses:
  Policy routing matches: 0 packets, 0 bytes

RT10#sh ip prefix-list 
ip prefix-list REDE-LOCAL: 7 entries
   seq 5 deny 192.168.0.0/16
   seq 10 deny 192.168.0.0/16 ge 17
   seq 15 deny 10.0.0.0/8
   seq 20 deny 10.0.0.0/8 ge 9
   seq 25 deny 8.8.8.8/32
   seq 30 deny 8.8.9.9/32
   seq 90 permit 0.0.0.0/0 le 32


RT10#sho ip bgp 
BGP table version is 29, local router ID is 1.1.1.1
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale
Origin codes: i - IGP, e - EGP, ? - incomplete

   Network          Next Hop            Metric LocPrf Weight Path
*> 8.8.8.8/32       172.16.1.2                          3000 65002 65009 i
*                   172.16.2.2                             0 65002 65009 i
*> 8.8.9.9/32       172.16.1.2                          3000 65002 65009 i
*                   172.16.2.2                             0 65002 65009 i
*> 10.1.1.0/30      172.16.1.2                          3000 65002 65009 i
*                   172.16.2.2                             0 65002 65009 i
*> 10.2.2.0/30      172.16.1.2                          3000 65002 65009 i
*                   172.16.2.2                             0 65002 65009 i
*> 172.16.1.0/30    0.0.0.0                  0         32768 i
*> 172.16.2.0/30    0.0.0.0                  0         32768 i
*> 192.168.0.0/16   0.0.0.0                            32768 i
s> 192.168.5.0      0.0.0.0                  0         32768 i
*> 200.19.19.1/32   0.0.0.0                  0         32768 i
*> 200.20.20.1/32   0.0.0.0                  0         32768 i
*> 200.21.21.1/32   0.0.0.0                  0         32768 i

```

