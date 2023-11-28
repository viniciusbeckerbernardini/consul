
# Service discovery

- Cenário comum em aplicações distribuídas:
	- APP A;
	- APP B;
	- APP C.
		- A -> B -> C;
			- Quando A chama B necessita saber o endereço do B.
			- Se houverem mais instâncias de B em detrimento de um load balancer e carga encima desse servidor. A precisa saber qual endereço chamar.
		- Dúvidas acerca:
					- Qual máquina chamar?
					- Qual porta utilizar?
					- Preciso saber o IP de cada instância?
					- Como ter certeza se aquela instância está saudavél?
					- Como saber se tenho permissão para acessar?
- Service Discovery:
	- Descobre máquinas disponíveis para acesso;
	- Segmentação de máquinas para garantir segurnaça;
	- Resolução via DNS;
	- Health Check ativo (verifica constantemente a saúde das máquinas, se não tiver ela é retirada para não correr riscos);
	- Valida permissões entre máquinas;

## Visão geral do Consul
- Hashicorp Consul:
	- Service Discovery;
	- Service Segmentation;
	- Load Balancer na Borda (Layer 7);
	- Key / Value Configuration (Banco de dados para variavéis de ambiente);
	- Opensource / Enterprise;
	- Recursos de segurança:
		- Mutual TLS;
		- Service Mesh;

## Service registry

- Service Registry:
	- Registro onde temos todos os serviços;
		- Vai saber o que está funcionando;
		- Processo de health check feito local;
			- Se acontecer algum problema retira do registro;
		- Gossip protocol:
			- Todo mundo conversa sobre quem ta fumeguiando no bagulho cupinxex;

## Health Check 

- Health check ativo:
	- Consul possui um agente (Consul Client) que fica rodando em cada servidor;
	- Fica acessando determinado endereço para validar se o endereço está respondendo;
	- Se não responder ele informa ao Consul Server que o negócio não tá mais bombiando;
	- Se não tiver mais bombiando o Consul Server tira ele da lista;

- Multi-region & Multi-cloud:
	- Pode ter aplicações em qualquer infra (aws, gcp, azure, etc);
	- Conceito de datacenter;

## Agent, Client e Server:

- Agent: Processo que fica sendo executadoem todos os nós do cluster.
	- Pode estar sendo executado em Client Mode ou em Server Mode.
- Client:
	- Registra os serviços localmente;
	- Health Check;
	- Encaminha as informações e configurações para o Server;
- Server:
	- Mantém o estado do cluester;
	- Registra os serviços;
	- Membership (quem é client e quem é server, define quem é o lider);
	- Retorno de queries (DNS  ou API);
	- Troca de informações entre datacenters, etc.

- Dev mode:
	- NUNCA utilize em produção;
	- Teste de features, exemplos;
	- Roda como servidor;
	- Não escala;
	- Registra tudo em memória.


## Obersevações e exemplos

- Em produção é recomendado ter pelo menos 3 máquinas server ou a partir de isso números impares;
- Raft protocol: as maquinas entram em consenso para determinar quem será o lider;
- Tem uma api REST;
- Tem um servidor DNS embutido;
- Se quiser informações do consul ou utiliza chamadas http ou dns;
- Pode utilizar SDK do consul que faz com ele varra o catalogo e entre os itens;

### Exemplo chamada http: 

``$ curl localhost:8500/v1/catalog/nodes``

retorno:

```
[
    {
        "ID": "58658427-29ef-4586-e4d1-48814e2ebbc8",
        "Node": "consul01",
        "Address": "127.0.0.1",
        "Datacenter": "dc1",
        "TaggedAddresses": {
            "lan": "127.0.0.1",
            "lan_ipv4": "127.0.0.1",
            "wan": "127.0.0.1",
            "wan_ipv4": "127.0.0.1"
        },
        "Meta": {
            "consul-network-segment": ""
        },
        "CreateIndex": 11,
        "ModifyIndex": 12
    }
]
```

- Datacenter é uma rede com membros de um cluster;
- Gossip protocol é usado efetivamente na mesma rede (datacenter);

### Exemplo chamada dns:

`` $ dig @localhost -p 8600 consul01.node.consul ``

retorno:

```
; <<>> DiG 9.16.44 <<>> @localhost -p 8600 consul01.node.consul
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 35648
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 2
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;consul01.node.consul.          IN      A

;; ANSWER SECTION:
consul01.node.consul.   0       IN      A       127.0.0.1

;; ADDITIONAL SECTION:
consul01.node.consul.   0       IN      TXT     "consul-network-segment="

;; Query time: 0 msec
;; SERVER: 127.0.0.1#8600(127.0.0.1)
;; WHEN: Mon Nov 27 10:55:14 UTC 2023
;; MSG SIZE  rcvd: 101
```

- Arquivo config-dir lê todas as configurações do agente;

### Iniciar consul agent server

``$ docker exec -it consulserver01 sh``  
``$ ifconfig ``  
`` $ consul agent -server -bootstrap-expect=3 -node=consulserver01 -bind=172.22.0.2 -data-dir=/var/lib/consul -config-dir=/etc/consul.d ``  

- Adicionar um serviço ao outro:
  - consul join [ip]

### Subindo agent client:
`` $ consul agent -node=consulclient01 -bind=172.22.0.5 -data-dir=/var/lib/consul -config-dir=/etc/consul.d ``  

### Subindo agent client com retry join (já sobe no cluster):
`` $ consul agent -node=consulclient02 -bind=172.22.0.6 -data-dir=/var/lib/consul -config-dir=/etc/consul.d -retry-join=172.22.0.4 [pode passar mais -retry-join] ``  

### Registrar serviço: 

- Dentro do config.d criar arquivo .json

```
{
	"service":{
		"id":"nginx",
		"name":"nginx",
		"tags":["web"],
		"port":80
	}
}
```

`` $ consul reload ``

- Registrar !== Subir
- Podemos subir 2 serviços iguais ele identificara que o serviço possui 2 ips disponíveis

`` $ curl localhost:8500/v1/catalog/services ``  

retorno: 

```
    {
    "consul":[],
    "nginx":["web"]
    }
```

`` $  consul catalog nodes -service nginx ``  

retorno: 

```
Node            ID        Address     DC
consulclient01  15dcd430  172.22.0.5  dc1
```


``$  consul catalog nodes -detailed ``

retorno: 

```
Node            ID                                    Address     DC   TaggedAddresses                                                           Meta
consulclient01  15dcd430-de62-a2bd-c942-89da4acbc37f  172.22.0.5  dc1  lan=172.22.0.5, lan_ipv4=172.22.0.5, wan=172.22.0.5, wan_ipv4=172.22.0.5  consul-network-segment=
consulserver01  a7b59fb7-fdc3-4365-be3a-7f96c1a26455  172.22.0.4  dc1  lan=172.22.0.4, lan_ipv4=172.22.0.4, wan=172.22.0.4, wan_ipv4=172.22.0.4  consul-network-segment=
consulserver02  b9793be9-2436-e4b5-2103-44af25eccabb  172.22.0.2  dc1  lan=172.22.0.2, lan_ipv4=172.22.0.2, wan=172.22.0.2, wan_ipv4=172.22.0.2  consul-network-segment=
consulserver03  381e1573-48de-e90d-ba53-e2612636bdf7  172.22.0.3  dc1  lan=172.22.0.3, lan_ipv4=172.22.0.3, wan=172.22.0.3, wan_ipv4=172.22.0.3  consul-network-segment=
```

Podemos subir serviços com mesmo nome mas com tags diferentes como exemplo PRD, HMG e QEA.

- Todos os serviços ficam sincronizados entre o client e o server. Não é necessário fazer consultas fora da máquina

- Podemos adicionar health checks para que o consul não mostre serviços fora do ar.


Comando simplficado em detrimento da criação de um server.json especificando as configurações:

- `` $ consul agent -config-dir=/etc/consul.d/ `` 


### Consultando serviço via chamada DNS

- `` $ dig @localhost -p 8600 web.nginx.service.consul ``  

retorno: 

```
; <<>> DiG 9.16.44 <<>> @localhost -p 8600 web.nginx.service.consul
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 65204
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;web.nginx.service.consul.      IN      A

;; ANSWER SECTION:
web.nginx.service.consul. 0     IN      A       172.22.0.5

;; Query time: 0 msec
;; SERVER: 127.0.0.1#8600(127.0.0.1)
;; WHEN: Mon Nov 27 11:43:32 UTC 2023
;; MSG SIZE  rcvd: 69
```

- Podemos subir serviços com mesmo nome mas com tags diferentes como exemplo PRD, HMG e QEA.

- Todos os serviços ficam sincronizados entre o client e o server. Não é necessário fazer consultas fora da máquina

- Podemos adicionar health checks para que o consul não mostre serviços fora do ar.

#### Comando simplficado em detrimento da criação de um server.json especificando as configurações:
- `` $ consul agent -config-dir=/etc/consul.d/ ``

- Para produção essencial:
    - Trabalhar com criptografia entre os servidores;
    - Trabalhar com tls