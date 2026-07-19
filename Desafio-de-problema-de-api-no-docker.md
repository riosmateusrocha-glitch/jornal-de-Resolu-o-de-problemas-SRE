# DevOps/SRE Challenge: Troubleshooting Multi-Arch Delivery & Container Networking

## 1. O Cenário (O Problema)
Durante o deploy local de um novo microsserviço de Gateway de Pagamentos desenvolvido em Go, a infraestrutura definida via Docker Compose falhou ao inicializar no ambiente de Engenharia (MacBook Air M1 - ARM64).

### Comportamento no Terminal:
```text
$ docker-compose up -d
[+] Running 2/2
 ✔ Network payments-routing_default  Created
 ✔ Container payments-redis-1       Started
 ✔ Container payments-api-1         Started

$ docker ps
CONTAINER ID   IMAGE          COMMAND                  CREATED         STATUS                     PORTS      NAMES
a1b2c3d4e5f6   redis:alpine   "docker-entrypoint.s…"   5 seconds ago   Up 4 seconds               6379/tcp   payments-redis-1
f6e5d4c3b2a1   payments-api   "/app/gateway-api"       5 seconds ago   Exited (139) 2 seconds ago            payments-api-1

$ docker logs payments-api-1
exec /app/gateway-api: exec format error

```

## Sintomas Identificados:
Crash Crítico da API (payments-api-1): O container encerra imediatamente com o código de saída 139 e reporta no log o erro exec format error.

Falha de Conectividade com o Banco: Mesmo simulando a execução da API em modo debug na mesma rede, a tentativa de conexão com o banco de dados Redis utilizando a string padrão localhost:6379 resultava em Connection Refused.

## 2. Minha Análise e Resolução do Incidente
Problema 1: Incompatibilidade de Arquitetura de CPU (exec format error)
Causa Raiz: O erro exec format error (erro ENOEXEC no Kernel Linux) ocorre quando o sistema tenta executar um arquivo binário cujas instruções de máquina foram compiladas para uma arquitetura de processador diferente da do host. Neste cenário, o time de desenvolvimento (ou o pipeline de CI) gerou o binário compilado para AMD64 (x86_64), enquanto a máquina de execução possui um processador Apple Silicon ARM64.

### Solução Proposta:

Solução Local/Contenção: Especificar a flag platform: linux/amd64 dentro do serviço no arquivo docker-compose.yml para forçar o Docker a emular a arquitetura via Rosetta 2/QEMU.

Solução de Plataforma (Definitiva): Configurar a pipeline de CI/CD (ex: GitHub Actions) utilizando o Docker Buildx para gerar uma imagem Multi-Arch (multi-plataforma), ou ajustar a etapa de compilação da aplicação informando as variáveis GOOS=linux GOARCH=arm64 para o target correto.

Problema 2: Isolamento de Namespace de Rede do Docker (Connection Refused)
Causa Raiz: O Docker Compose cria, por padrão, uma rede isolada em modo bridge (neste caso, payments-routing_default). Cada container possui seu próprio namespace de rede e sua própria interface de loopback. Quando a API tentava alcançar o Redis usando localhost, ela estava buscando o serviço dentro de si mesma (127.0.0.1 do próprio container), onde o Redis não está rodando.

Solução Proposta: Como ambos os containers estão conectados à mesma rede bridge customizada, deve-se utilizar o DNS Interno do Docker. A string de conexão da API deve ser alterada de localhost:6379 para o nome do serviço ou container mapeado no Compose, por exemplo: redis://payments-redis-1:6379. O servidor de DNS embutido do Docker resolverá dinamicamente o IP privado correto do container Redis.

## 3. Avaliação Técnica do Mentor (SRE/Platform Engineering)
A resolução proposta atende rigorosamente aos padrões de Engenharia de Confiabilidade de Sistemas pelos seguintes fatores:

Domínio de Fundamentos de Sistemas Operacionais: Identificou com precisão a diferença entre instruções de nível de hardware (AMD64 vs ARM64) e como o subsistema de emulação do Docker interage com o Apple Silicon.

Entendimento de Abstração de Redes em Containers: Demonstrou clareza quanto ao isolamento de Namespaces de Rede no Linux. O entendimento de que redes bridge customizadas no Docker possuem resolução de DNS nativa (diferente da bridge padrão do sistema) evita o uso de IPs estáticos ou hacks de infraestrutura frágeis (host.docker.internal).

Abordagem de Engenharia de Plataforma: A indicação do uso de Docker Buildx para automação de imagens multi-arquitetura no pipeline demonstra visão de escalabilidade, garantindo que o software seja agnóstico ao hardware do desenvolvedor e do cluster de produção (AWS Graviton/ARM vs EC2 Intel).
