# Configurar um cluster OpenMPI com 4 instâncias EC2 pela interface web da AWS

Este guia configura três instâncias EC2 como nós de processamento (*workers*) e utiliza uma quarta instância EC2 já existente, com code-server e OpenMPI, como nó de controle.

> **Arquitetura:** o nó de controle compila e inicia os processos MPI; os três workers executam os processos. A comunicação entre eles ocorre pela rede privada da AWS.

## 1. Planejar as instâncias

| Configuração | Nó de controle existente | Cada worker |
|---|---|---|
| Sistema operacional | Manter o atual | Ubuntu Server 24.04 LTS |
| Tipo | Manter o atual | `t3.small` para testes iniciais |
| OpenMPI | Já instalado | Instalar OpenMPI |
| Usuário SSH | Verificar o usuário atual | `ubuntu` |
| Rede | IP privado | IP privado de cada worker |

Use a mesma região e, preferencialmente, a mesma VPC e sub-rede para as quatro instâncias. A `t3.small` é um ponto de partida para testes acadêmicos, mas pode ser insuficiente para cargas intensivas. Instâncias T3 usam créditos de CPU; para experimentos prolongados, avalie outros tipos compatíveis com seu orçamento e com as restrições da conta.

## 2. Criar um grupo de segurança para o cluster

1. Abra o [console EC2 da AWS](https://console.aws.amazon.com/ec2/).
2. No menu lateral, entre em **Network & Security → Security Groups**.
3. Clique em **Create security group**.
4. Configure:
   - **Security group name:** `OpenMPI-Cluster-SG`
   - **Description:** `Comunicacao privada entre nos OpenMPI`
   - **VPC:** a mesma da instância de controle.
5. Em **Inbound rules**, adicione:

   | Tipo | Origem | Finalidade |
   |---|---|---|
   | All traffic | `OpenMPI-Cluster-SG` (o próprio grupo) | Comunicação entre os nós |
   | SSH (TCP 22) | Seu IP público `/32` | Administração direta, se necessária |

   A regra SSH pode ser dispensável nos workers se o acesso for feito exclusivamente pela instância de controle e houver um método de acesso inicial disponível. Mantenha o tráfego do cluster restrito ao próprio grupo de segurança.

6. Mantenha a saída padrão liberada, se permitido pela política da conta.
7. Clique em **Create security group**.

**Não** configure `All traffic` com origem `0.0.0.0/0`.

### Associar o grupo às quatro instâncias

Para cada uma das três instâncias worker e para a instância de controle:

1. Abra **EC2 → Instances**.
2. Selecione a instância.
3. Escolha **Actions → Security → Change security groups**.
4. Adicione `OpenMPI-Cluster-SG`, sem remover os grupos existentes que sejam necessários.
5. Salve.

Na instância de controle, preserve o grupo que permite acessar o code-server (por exemplo, porta 8080). Não altere suas regras de acesso sem necessidade.

## 3. Criar as três instâncias workers

No console EC2, clique em **Launch instances** e configure a primeira instância:

- **Name:** `mpi-worker-1`
- **AMI:** Ubuntu Server 24.04 LTS
- **Architecture:** 64-bit x86
- **Instance type:** `t3.small`
- **Key pair (login):** escolha ou crie uma chave SSH e guarde o arquivo `.pem` com segurança. Uma chave privada não pode ser baixada novamente após a criação.
- **VPC:** a mesma da instância de controle.
- **Subnet:** preferencialmente a mesma sub-rede.
- **Auto-assign public IP:** pode ser desativado se a conectividade necessária estiver disponível por meio da rede existente.
- **Firewall:** associe `OpenMPI-Cluster-SG`, mantendo também outros grupos necessários.
- **Storage:** o volume padrão do Ubuntu (por exemplo, 8–10 GiB) pode bastar para testes pequenos.

Repita para `mpi-worker-2` e `mpi-worker-3`.

Antes de lançar, confirme que a região e a zona de disponibilidade aceitam o tipo de instância e que há capacidade disponível. Depois, em **EC2 → Instances**, aguarde as três ficarem em `Running` e passarem nas verificações de status.

## 4. Anotar os IPs privados

Selecione cada instância e consulte os detalhes de rede. Anote os endereços IPv4 privados.

Exemplo **ilustrativo** — substitua pelos endereços reais:

| Máquina | Nome | IP privado |
|---|---|---|
| Controle | Instância existente | `172.31.16.115` |
| Worker 1 | `mpi-worker-1` | `172.31.16.120` |
| Worker 2 | `mpi-worker-2` | `172.31.16.121` |
| Worker 3 | `mpi-worker-3` | `172.31.16.122` |

Os endereços acima são exemplos, não devem ser copiados sem conferência. O nó de controle precisa alcançar os workers por seus IPs privados.

Se o controle estiver em outra VPC, será necessário configurar conectividade entre as redes. Para o primeiro experimento, use a mesma VPC.

## 5. Configurar SSH sem senha

O `mpirun` precisa iniciar processos remotamente sem solicitar senha interativamente.

No terminal do code-server, gere uma chave específica para o cluster:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/mpi_cluster -N ""
```

Isso cria uma chave privada e uma pública. Mantenha a chave privada no nó de controle; **não a compartilhe com os workers nem a envie para repositórios**.

Exiba a chave pública:

```bash
cat ~/.ssh/mpi_cluster.pub
```

Copie a linha inteira.

### Autorizar a chave nos três workers

É necessário fazer uma primeira conexão administrativa com cada worker. Uma opção é usar **EC2 Instance Connect** no console, quando disponível e compatível com a configuração:

1. Selecione `mpi-worker-1` no console EC2.
2. Clique em **Connect**.
3. Escolha EC2 Instance Connect, se disponível.
4. No terminal remoto, execute:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys
```

Cole a chave pública do nó de controle em uma nova linha, salve e execute:

```bash
chmod 600 ~/.ssh/authorized_keys
```

Repita nos outros dois workers. O EC2 Instance Connect depende de requisitos de rede, sistema operacional e permissões. Se não estiver disponível, use outro método de acesso inicial com a chave escolhida no lançamento.

### Configurar os nomes SSH no nó de controle

No terminal do code-server, execute:

```bash
cat >> ~/.ssh/config <<'EOF'
Host mpi-worker-1
    HostName IP_PRIVADO_WORKER_1
    User ubuntu
    IdentityFile ~/.ssh/mpi_cluster
    StrictHostKeyChecking accept-new

Host mpi-worker-2
    HostName IP_PRIVADO_WORKER_2
    User ubuntu
    IdentityFile ~/.ssh/mpi_cluster
    StrictHostKeyChecking accept-new

Host mpi-worker-3
    HostName IP_PRIVADO_WORKER_3
    User ubuntu
    IdentityFile ~/.ssh/mpi_cluster
    StrictHostKeyChecking accept-new
EOF

chmod 600 ~/.ssh/config
```

Substitua `IP_PRIVADO_WORKER_1`, `IP_PRIVADO_WORKER_2` e `IP_PRIVADO_WORKER_3` pelos IPs reais.

Teste as conexões:

```bash
ssh mpi-worker-1 hostname
ssh mpi-worker-2 hostname
ssh mpi-worker-3 hostname
```

Os comandos devem conectar sem pedir senha. Na primeira conexão, pode aparecer uma confirmação da identidade do host.

## 6. Instalar OpenMPI nos workers

No terminal do nó de controle, execute:

```bash
for host in mpi-worker-1 mpi-worker-2 mpi-worker-3; do
    ssh "$host" \
      'sudo apt-get update && sudo apt-get install -y openmpi-bin libopenmpi-dev'
done
```

Verifique as versões no controle e nos workers:

```bash
mpirun --version

for host in mpi-worker-1 mpi-worker-2 mpi-worker-3; do
    echo "=== $host ==="
    ssh "$host" 'which mpirun && mpirun --version'
done
```

Mantenha versões compatíveis e instalações equivalentes do OpenMPI em todos os nós. Os executáveis e bibliotecas necessários precisam estar disponíveis nos workers.

## 7. Criar o arquivo de hosts

No nó de controle, crie um diretório para os experimentos:

```bash
mkdir -p ~/mpi-cluster
cd ~/mpi-cluster
```

Crie o arquivo:

```bash
nano hosts
```

Conteúdo:

```text
mpi-worker-1 slots=2
mpi-worker-2 slots=2
mpi-worker-3 slots=2
```

Os valores `slots=2` são um ponto de partida para máquinas `t3.small`, que normalmente possuem 2 vCPUs. Isso não significa necessariamente dois núcleos físicos dedicados por instância.

Como o arquivo lista apenas os três workers, o comando abaixo distribuirá os processos somente entre eles.

## 8. Criar e executar um programa de teste MPI

Crie `hello.c`:

```bash
nano hello.c
```

Conteúdo:

```c
#include <mpi.h>
#include <stdio.h>
#include <unistd.h>

int main(int argc, char **argv) {
    int rank, size;
    char hostname[256];

    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);

    gethostname(hostname, sizeof(hostname));

    printf("Processo %d de %d executando em %s\n",
           rank, size, hostname);

    MPI_Finalize();
    return 0;
}
```

Compile no nó de controle:

```bash
mpicc hello.c -o hello
```

Copie o executável para cada worker:

```bash
for host in mpi-worker-1 mpi-worker-2 mpi-worker-3; do
    scp ~/mpi-cluster/hello "$host":/home/ubuntu/hello
done
```

Execute seis processos distribuídos pelos três workers:

```bash
mpirun --hostfile hosts -np 6 \
    --wdir /home/ubuntu /home/ubuntu/hello
```

Você deverá ver seis mensagens, com processos executando em três nomes de host distintos. A ordem das mensagens pode variar.

### Se houver erro

1. Confirme que `ssh mpi-worker-1 hostname` (e os demais) funciona sem senha.
2. Confirme que todos os nós usam versões compatíveis do OpenMPI.
3. Confira o grupo de segurança, a VPC e os IPs privados.
4. Verifique se o executável foi copiado para `/home/ubuntu/hello` em todos os workers.
5. Se o SSH funcionar, mas o MPI falhar, verifique as mensagens de erro do OpenMPI e a comunicação entre os nós.

## 9. Checklist de conclusão

- [ ] As três instâncias estão em `Running` e passaram nas verificações de status.
- [ ] Controle e workers estão na mesma VPC ou têm conectividade privada configurada.
- [ ] `OpenMPI-Cluster-SG` está associado às quatro instâncias.
- [ ] SSH do controle para os três workers funciona sem senha.
- [ ] OpenMPI está instalado e é compatível em todos os nós.
- [ ] O arquivo `hosts` lista os três workers corretamente.
- [ ] O teste MPI executa processos nos três workers.

## 10. Custos e segurança

Quando não estiver usando o cluster, pare as três instâncias pelo console EC2. Volumes EBS e outros recursos podem continuar gerando cobranças mesmo com as instâncias paradas.

Se estiver usando AWS Academy Learner Lab, confirme as permissões, regiões, tipos de instância e limites impostos pelo laboratório.

Não exponha todas as portas do cluster à Internet. Restrinja a comunicação entre os nós ao grupo de segurança do cluster e mantenha protegida a chave privada SSH.

## Referência

- [Documentação oficial do OpenMPI — Quick start e execução em múltiplos hosts](https://docs.open-mpi.org/en/main/launching-apps/quickstart.html)
- [Console EC2 da AWS](https://console.aws.amazon.com/ec2/)
