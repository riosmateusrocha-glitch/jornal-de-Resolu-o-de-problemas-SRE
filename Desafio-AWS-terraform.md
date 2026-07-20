# AWS & Terraform Challenge: Troubleshooting VPC Routing & NAT Gateway Architectures

## 1. O Cenário (O Problema)
Durante a automação da infraestrutura de rede e computação na AWS via Terraform para suportar um cluster Kubernetes (EKS), as instâncias EC2 falharam catastróficamente ao executar o script de inicialização (`user_data`). As máquinas não conseguiram se integrar ao cluster porque ficaram completamente isoladas, incapazes de baixar pacotes e dependências essenciais da internet.

### Evidência extraída do arquivo `/var/log/user_data.log` via AWS Systems Manager (SSM):
```text
[ERROR] 2026-07-20 09:15:32 - curl: (7) Failed to connect to production.cloudflare.docker.com port 443 after 21045 ms: Connection timed out
[ERROR] 2026-07-20 09:16:00 - wget: download failed: Connection timed out
[FATAL] 2026-07-20 09:16:00 - Failed to download Kubernetes cluster joining components. Exiting setup.
```

## Arquitetura Inicial Declarada no Terraform:

- VPC: Bloco CIDR `10.0.0.0/16`.
- Sub-rede Privada: Bloco CIDR `10.0.1.0/24` (Onde as instâncias EC2 foram provisionadas).
- Internet Gateway (IGW): Criado e anexado à VPC.
- Security Group: Configurado com política de saída estrita de `egress` totalmente aberta (`0.0.0.0/0`).

## 2. Minha Análise e Resolução do Incidente
### Problema 1: Tabela de Rotas Incompleta (Timeout de Rede)

- Causa Raiz: Por padrão, a criação de uma VPC e sub-redes na AWS gera apenas a tabela de rotas com escopo `local` (ex: `10.0.0.0/16 -> local`). Mesmo com um Internet Gateway (IGW) anexado à VPC, os pacotes com destino à internet pública (`0.0.0.0/0`) disparados pelas instâncias EC2 não tinham um próximo salto (next hop) definido, resultando no descarte dos pacotes e gerando o `Connection timed out`.
- Solução Proposta: É necessário declarar explicitamente no Terraform a criação de tabelas de rotas e tabelas de associação, injetando a rota padrão `0.0.0.0/0` apontando para o gateway de saída apropriado.

## Problema 2: Violação de Isolamento de Sub-rede Privada

- Causa Raiz: As instâncias foram intencionalmente alocadas em uma sub-rede privada, possuindo apenas endereços de IP Privados (`10.0.1.0/24`). Roteadores da internet pública não trafegam IPs privados. Se a rota padrão `0.0.0.0/0` fosse simplesmente apontada para o IGW, as instâncias precisariam receber IPs públicos diretos, violando as premissas de segurança de produção de um cluster Kubernetes.
- Solução Proposta: Implementar uma arquitetura de sub-redes segregadas utilizando a técnica de Tradução de Endereços de Rede (NAT).

## 3. Arquitetura Alvo e Lógica no Terraform
Para solucionar o isolamento mantendo a segurança da infraestrutura, foi desenhada a seguinte topologia lógica implementada em Terraform:
1. Criação de Sub-rede Pública: Adição de uma zona de rede pública com rota padrão (`0.0.0.0/0`) direta para o **Internet Gateway (IGW)**.
2. Provisionamento do Elastic IP (EIP): Declaração de um IP público estático (`aws_eip`) dedicado a mascarar a saída da rede.
3. Provisionamento do NAT Gateway: Alocação do recurso `aws_nat_gateway` dentro da sub-rede pública recém-criada, vinculando-o ao Elastic IP e adicionando um bloco de dependência explícita (`depends_on`) para garantir a precedência do IGW.
4. Atualização do Roteamento Privado: Configuração do recurso `aws_route_table` da sub-rede privada, direcionando a rota de saída `0.0.0.0/0` para o ID do **NAT Gateway**.
5. Vínculo Final: Associação da tabela de rotas privada à respectiva sub-rede privada via `aws_route_table_association`.
Com esse fluxo, as instâncias EC2 privadas ganharam a capacidade de iniciar conexões com a internet para atualizar os pacotes do Kubernetes de forma totalmente mascarada pelo NAT Gateway, sem expor suas interfaces internas a ameaças externas.

### Avaliação Técnica do Mentor (SRE/Platform Engineering)
A resolução demonstra profunda maturidade técnica em redes de computadores aplicada a ambientes Cloud:

- Domínio do Modelo de Rede da AWS (VPC Core): Evidenciou claro entendimento sobre o comportamento stateful de Security Groups, diferenciando com precisão regras de *Ingress* e *Egress*, além de compreender a natureza implícita do ecossistema DHCP/DNS gerenciados da nuvem AWS.
- Segregação de Ambientes e Compliance de Segurança: Rejeitou a solução simplista de expor instâncias privadas com IPs públicos, adotando o padrão corporativo de NAT Gateway para manter o perímetro de segurança intacto.
- Mentalidade de Ciclo de Vida de Código (Terraform): A estruturação mental da ordem de dependência de criação de recursos (Sub-rede Pública $\rightarrow$ IGW $\rightarrow$ EIP $\rightarrow$ NAT Gateway $\rightarrow$ Route Tables) prova que o engenheiro domina a engine de grafos do Terraform, mitigando problemas comuns de concorrência (*race conditions*) em pipelines de CI/CD.

