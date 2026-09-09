# Exercicios de Firewall: UFW versus iptables

> Os comandos abaixo sao respostas teoricas aos exercicios. Eles nao foram
> executados neste computador. Use `sudo` e confirme o ambiente antes de
> aplicar regras, especialmente em uma conexao SSH remota.

## Parte 1 - Verificacao e politicas padrao

### Exercicio 1 - Verificar o status

**UFW**

```bash
sudo ufw status verbose
```

**iptables**

```bash
sudo iptables -L -n -v --line-numbers
```

O comando do `iptables` lista as cadeias, politicas, regras, contadores,
enderecos e portas sem tentar resolver nomes DNS.

### Exercicio 2 - Habilitar e desabilitar

**UFW**

```bash
sudo ufw enable
sudo ufw disable
```

**iptables**

O `iptables` nao possui um comando unico de ativar ou desativar, pois ele
apenas manipula regras no kernel. Para deixar as cadeias principais aceitando
trafego e limpar as regras:

```bash
sudo iptables -F
sudo iptables -X
sudo iptables -P INPUT ACCEPT
sudo iptables -P OUTPUT ACCEPT
sudo iptables -P FORWARD ACCEPT
```

Em sistemas com IPv6, o equivalente deve ser aplicado com `ip6tables`.

### Exercicio 3 - Negar todo o trafego de entrada

**UFW**

```bash
sudo ufw default deny incoming
```

**iptables**

```bash
sudo iptables -P INPUT DROP
```

Essa politica pode interromper acessos atuais. Antes dela, libere o SSH ou
outras conexoes necessarias.

### Exercicio 4 - Permitir todo o trafego de saida

**UFW**

```bash
sudo ufw default allow outgoing
```

**iptables**

```bash
sudo iptables -P OUTPUT ACCEPT
```

## Parte 2 - Liberacao basica por porta e protocolo

### Exercicio 5 - Liberar SSH (TCP/22)

**UFW**

```bash
sudo ufw allow 22/tcp
```

**iptables**

```bash
sudo iptables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW -j ACCEPT
```

### Exercicio 6 - Liberar HTTP (TCP/80)

**UFW**

```bash
sudo ufw allow 80/tcp
```

**iptables**

```bash
sudo iptables -A INPUT -p tcp --dport 80 -m conntrack --ctstate NEW -j ACCEPT
```

### Exercicio 7 - Liberar TCP na porta 8080

**UFW**

```bash
sudo ufw allow 8080/tcp
```

**iptables**

```bash
sudo iptables -A INPUT -p tcp --dport 8080 -m conntrack --ctstate NEW -j ACCEPT
```

### Exercicio 8 - Liberar DNS (UDP/53)

**UFW**

```bash
sudo ufw allow 53/udp
```

**iptables**

```bash
sudo iptables -A INPUT -p udp --dport 53 -m conntrack --ctstate NEW -j ACCEPT
```

## Parte 3 - Bloqueio e rejeicao

### Exercicio 9 - Bloquear a porta 3306 (DROP)

**UFW**

```bash
sudo ufw deny 3306/tcp
```

**iptables**

```bash
sudo iptables -A INPUT -p tcp --dport 3306 -j DROP
```

O alvo `DROP` descarta o pacote sem enviar resposta ao remetente.

### Exercicio 10 - Rejeitar a porta 5432 (REJECT)

**UFW**

```bash
sudo ufw reject 5432/tcp
```

**iptables**

```bash
sudo iptables -A INPUT -p tcp --dport 5432 -j REJECT --reject-with tcp-reset
```

O alvo `REJECT` envia uma resposta ativa, indicando que a conexao foi
recusada.

### Exercicio 11 - Bloquear um endereco IP

**UFW**

```bash
sudo ufw deny from 1.2.3.4
```

**iptables**

```bash
sudo iptables -A INPUT -s 1.2.3.4 -j DROP
```

### Exercicio 12 - Liberar SSH somente para um IP

**UFW**

```bash
sudo ufw allow from 192.168.1.100 to any port 22 proto tcp
```

**iptables**

```bash
sudo iptables -A INPUT -p tcp -s 192.168.1.100 --dport 22 \
  -m conntrack --ctstate NEW -j ACCEPT
```

Se houver uma politica `INPUT ACCEPT`, e necessario tambem bloquear o SSH
para os demais enderecos:

```bash
sudo iptables -A INPUT -p tcp --dport 22 -j DROP
```

### Exercicio 13 - Liberar MySQL somente pela interface `eth1`

**UFW**

```bash
sudo ufw allow in on eth1 to any port 3306 proto tcp
```

**iptables**

```bash
sudo iptables -A INPUT -i eth1 -p tcp --dport 3306 \
  -m conntrack --ctstate NEW -j ACCEPT
```

### Exercicio 14 - Bloquear pacotes `INVALID`

**UFW**

O UFW nao oferece uma regra de alto nivel especifica para o estado
`INVALID`. A regra pode ser adicionada usando a configuracao de baixo nivel
do UFW ou, de forma direta, com iptables:

**iptables**

```bash
sudo iptables -A INPUT -m conntrack --ctstate INVALID -j DROP
```

## Parte 5 - Gerenciamento de regras

### Exercicio 15 - Listar regras com numeros

**UFW**

```bash
sudo ufw status numbered
```

**iptables**

```bash
sudo iptables -L INPUT -n -v --line-numbers
```

### Exercicio 16 - Deletar a regra de numero 5

**UFW**

```bash
sudo ufw delete 5
```

**iptables**

```bash
sudo iptables -D INPUT 5
```

O numero deve ser conferido imediatamente antes da exclusao, pois ele muda
quando uma regra anterior e removida.

### Exercicio 17 - Resetar tudo

**UFW**

```bash
sudo ufw reset
```

O comando solicita confirmacao, desativa o UFW e restaura a configuracao
padrao. Para confirmar automaticamente:

```bash
sudo ufw --force reset
```

**iptables**

```bash
sudo iptables -F
sudo iptables -X
sudo iptables -Z
sudo iptables -P INPUT ACCEPT
sudo iptables -P OUTPUT ACCEPT
sudo iptables -P FORWARD ACCEPT
```

Para limpar tambem tabelas que nao sao `filter`:

```bash
sudo iptables -t nat -F
sudo iptables -t nat -X
sudo iptables -t mangle -F
sudo iptables -t mangle -X
sudo iptables -t raw -F
sudo iptables -t raw -X
```

Em hosts que usam IPv6, repita a limpeza e as politicas com `ip6tables`.
Esses comandos removem regras ativas e podem expor temporariamente o host;
aplique-os somente com acesso local ou com um plano de recuperacao.

## Observacoes finais

- Regras do `iptables` sao normalmente perdidas apos a reinicializacao, a
  menos que sejam salvas por uma ferramenta como `iptables-persistent`.
- O UFW funciona como uma camada de gerenciamento e gera regras do netfilter.
- A ordem das regras importa: uma regra anterior pode aceitar ou descartar o
  pacote antes que uma regra posterior seja avaliada.
- Em producao, prefira restringir portas por IP, interface e estado de conexao
  quando isso for compatível com o servico.
