# Relatório — Estrutura Kubernetes/GitOps do YCommunication

Data: 2026-08-29
Repositório: `homelab-gitops` (branch `main`)

## 1. Arquivos criados

```
ycommunication/
├── base/
│   ├── ycommunication-api/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── kustomization.yaml
│   ├── ycommunication-web/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── kustomization.yaml
│   ├── ingress-hml/
│   │   ├── ingress.yaml
│   │   └── kustomization.yaml
│   ├── ingress-prd/
│   │   ├── ingress.yaml
│   │   └── kustomization.yaml
│   └── rabbitmq/                    (PROPOSTA — não referenciado em nenhum overlay, ver seção 6)
│       ├── deployment.yaml
│       ├── service.yaml
│       ├── pvc.yaml
│       └── kustomization.yaml
└── overlays/
    ├── hml/
    │   ├── namespace.yaml
    │   └── kustomization.yaml
    └── prd/
        ├── namespace.yaml
        └── kustomization.yaml
```

Padrão seguido: idêntico ao `ykanban/` (namespace declarativo por overlay, patch estratégico para `SPRING_PROFILES_ACTIVE`, override de imagem via campo `images:` do Kustomize) combinado com o padrão de dois domínios do `sorteio/` (host separado para frontend e API, em vez de path `/api`).

## 2. Resultado das validações `kubectl kustomize`

`kubectl` (com Kustomize v5.8.1 embutido) está disponível nesta VM de agente — usado apenas como ferramenta cliente de templating, **sem conexão com nenhum cluster** e sem instalar Kubernetes/K3s na VM. Nenhum `kubectl apply` foi executado.

```
kubectl kustomize ycommunication/overlays/hml   # OK, renderizou sem erros
kubectl kustomize ycommunication/overlays/prd   # OK, renderizou sem erros
```

Checklist confirmado no YAML renderizado:

| Verificação | HML | PRD |
|---|---|---|
| `namespace` | `ycommunication-hml` ✅ | `ycommunication-prd` ✅ |
| Imagem API | `ghcr.io/dreis2003/ycommunication-api:hml-ed60990` ✅ | `ghcr.io/dreis2003/ycommunication-api:prd-latest` ✅ |
| Imagem Web | `ghcr.io/dreis2003/ycommunication-web:hml-e7cd684` ✅ | `ghcr.io/dreis2003/ycommunication-web:prd-latest` ✅ |
| `SPRING_PROFILES_ACTIVE` | `hml` ✅ | `prd` ✅ (via patch) |
| `CORS_ALLOWED_ORIGINS` | `https://ycommunication-hml.yakuzastudio.com.br` ✅ | `https://ycommunication.yakuzastudio.com.br` ✅ (via patch) |
| Ingress hosts | `ycommunication-hml.yakuzastudio.com.br`, `api-ycommunication-hml.yakuzastudio.com.br` ✅ | `ycommunication.yakuzastudio.com.br`, `api-ycommunication.yakuzastudio.com.br` ✅ |
| `ingressClassName` | `traefik` ✅ | `traefik` ✅ |

## 3. Imagens configuradas

**HML** — tags imutáveis (SHA), conforme já publicado no GHCR:
- `ghcr.io/dreis2003/ycommunication-api:hml-ed60990`
- `ghcr.io/dreis2003/ycommunication-web:hml-e7cd684`

**PRD** — `prd-latest`, ainda não publicada (só existirá após merge do PR `hml→main` em cada repositório e execução do workflow na branch `main`). **Não tentei validar se essas imagens existem no GHCR — elas ainda não existem, conforme esperado nesta etapa.**

## 4. Variáveis de ambiente / Secret `ycommunication-api-secrets`

O Deployment do backend referencia `secretKeyRef` para um Secret chamado `ycommunication-api-secrets` (mesmo padrão do `ykanban-api-secrets`/`ms-sorteio-secrets` — criado manualmente no cluster, fora do GitOps declarativo). Nenhum valor real foi commitado.

Chaves esperadas nesse Secret (confirmadas lendo `application.yml` e o código-fonte de `ycommunication-api`, não apenas o `.env.example`):

| Chave no Secret | Env var no container | Obrigatória para o boot? |
|---|---|---|
| `DB_URL` | `SPRING_DATASOURCE_URL` / `SPRING_FLYWAY_URL` | ✅ sim — sem default |
| `DB_USERNAME` | `SPRING_DATASOURCE_USERNAME` / `SPRING_FLYWAY_USER` | ✅ sim |
| `DB_PASSWORD` | `SPRING_DATASOURCE_PASSWORD` / `SPRING_FLYWAY_PASSWORD` | ✅ sim |
| `JWT_SECRET` | `JWT_SECRET` | ✅ sim — falha no boot se tiver menos de 32 caracteres |
| `RABBITMQ_HOST` | `RABBITMQ_HOST` | Tem default `localhost` no código, mas **precisa ser sobrescrita** ou o pod nunca fica `Ready` (actuator `health` depende do RabbitMQ) |
| `RABBITMQ_PORT` | `RABBITMQ_PORT` | idem (default `5672`) |
| `RABBITMQ_USERNAME` | `RABBITMQ_USERNAME` | idem (default `guest`) |
| `RABBITMQ_PASSWORD` | `RABBITMQ_PASSWORD` | idem (default `guest`) |
| `YCOMM_ENCRYPTION_CURRENT_KEY_VERSION` | `YCOMM_ENCRYPTION_CURRENT_KEY_VERSION` | Não trava o boot (é lida sob demanda via `Environment`, não via `application.yml`), mas **obrigatória para o app funcionar de verdade** — sem ela, qualquer configuração de canal (SMTP/Telegram/WhatsApp/Webhook) falha com `InvalidEncryptionKeyException` |
| `YCOMM_ENCRYPTION_KEY_V1` | `YCOMM_ENCRYPTION_KEY_V1` | idem — gerar com `openssl rand -base64 32`, uma chave **diferente por ambiente** (HML ≠ PRD) |

`CORS_ALLOWED_ORIGINS` **não** está no Secret — é passada como valor literal direto no manifest (igual ao padrão do `ykanban`), com o domínio correto por ambiente já cravado no overlay.

### Achado importante: SMTP/Telegram/WhatsApp/Webhook NÃO são secrets de infraestrutura
Diferente do que a tarefa cogitava, o YCommunication **não** usa variáveis de ambiente para credenciais de SMTP/Telegram/WhatsApp/Webhook. Essas credenciais são configuradas por tenant via API/UI da própria aplicação e ficam **criptografadas no PostgreSQL** (AES-256-GCM, usando a `YCOMM_ENCRYPTION_KEY_V1` acima). Por isso não criei `SMTP_*`/`TELEGRAM_*`/`WHATSAPP_*`/`WEBHOOK_*` no Secret — seriam variáveis sem efeito nenhum no código atual.

### Observação: perfis `hml`/`prd` não existem como arquivo Spring
O repositório `ycommunication-api` só tem `application.yml` (config base) e `application-local.yml` (dev local) — **não existe** `application-hml.yml` nem `application-prd.yml`. Definir `SPRING_PROFILES_ACTIVE=hml`/`prd` é inofensivo (Spring Boot ativa o profile mas não encontra nenhum arquivo de override, então segue só com `application.yml` + variáveis de ambiente) e serve como rótulo/metadado — não é bloqueante, mas caso o time queira comportamento diferenciado por ambiente no futuro, será necessário criar esses arquivos no código-fonte (fora do escopo desta etapa).

### `SERVER_PORT`
Setado explicitamente como `"8080"` no Deployment, embora já seja o default da aplicação — mantido por clareza e para bater com `containerPort`.

## 5. Pendências manuais antes do primeiro deploy

Nenhuma destas foi executada por este agente (fora do escopo/autorização desta etapa):

1. **Criar bancos no PostgreSQL** (`192.168.40.80`):
   ```sql
   CREATE DATABASE db_ycommunication_hml;
   CREATE DATABASE db_ycommunication;
   ```
2. **Criar usuários PostgreSQL** com permissão nesses bancos:
   ```sql
   CREATE USER ycommunication_hml WITH PASSWORD '...';
   GRANT ALL PRIVILEGES ON DATABASE db_ycommunication_hml TO ycommunication_hml;

   CREATE USER ycommunication_prd WITH PASSWORD '...';
   GRANT ALL PRIVILEGES ON DATABASE db_ycommunication TO ycommunication_prd;
   ```
3. **Criar o Secret `ycommunication-api-secrets`** em cada namespace (executar na VM `k8s-lab`, com acesso ao cluster):
   ```bash
   kubectl create namespace ycommunication-hml   # se ainda não existir via ArgoCD
   kubectl create secret generic ycommunication-api-secrets \
     -n ycommunication-hml \
     --from-literal=DB_URL="jdbc:postgresql://192.168.40.80:5432/db_ycommunication_hml" \
     --from-literal=DB_USERNAME="ycommunication_hml" \
     --from-literal=DB_PASSWORD="<senha>" \
     --from-literal=JWT_SECRET="$(openssl rand -base64 48)" \
     --from-literal=RABBITMQ_HOST="<a definir - ver secao 6>" \
     --from-literal=RABBITMQ_PORT="5672" \
     --from-literal=RABBITMQ_USERNAME="<a definir>" \
     --from-literal=RABBITMQ_PASSWORD="<a definir>" \
     --from-literal=YCOMM_ENCRYPTION_CURRENT_KEY_VERSION="v1" \
     --from-literal=YCOMM_ENCRYPTION_KEY_V1="$(openssl rand -base64 32)"
   ```
   Repetir para `ycommunication-prd` com valores próprios (nunca reaproveitar `JWT_SECRET`/`YCOMM_ENCRYPTION_KEY_V1` entre HML e PRD).
4. **Copiar/criar o `ghcr-secret`** (imagePullSecret) nos namespaces `ycommunication-hml` e `ycommunication-prd` — os Deployments já referenciam `imagePullSecrets: [{name: ghcr-secret}]`, seguindo o padrão visto em `springboot-demo/deployment.yaml`. Sem isso, o pull das imagens do GHCR falhará (assumindo que os pacotes não sejam públicos).
5. **Criar as Applications no ArgoCD** (`ycommunication-hml`, `ycommunication-prd`) apontando para `homelab-gitops`, path `ycommunication/overlays/hml` e `ycommunication/overlays/prd` respectivamente. Não localizei nenhum manifest de `Application` versionado neste repositório para os apps existentes — confirmar com o time como as demais Applications foram registradas antes de replicar o processo.
6. **Criar os hostnames no Cloudflare Tunnel** (ou DNS equivalente) para os 4 domínios, apontando para `192.168.40.70`:
   - `ycommunication-hml.yakuzastudio.com.br`
   - `api-ycommunication-hml.yakuzastudio.com.br`
   - `ycommunication.yakuzastudio.com.br` (pode esperar até a promoção PRD)
   - `api-ycommunication.yakuzastudio.com.br` (idem)

## 6. RabbitMQ — decisão pendente

O YCommunication **exige** RabbitMQ para o pod ficar `Ready` (o worker de outbox e o `management.health.rabbit.enabled: true` do Actuator dependem disso). Hoje **não existe RabbitMQ em lugar nenhum do homelab** — nem em VM dedicada (como o Postgres), nem em nenhum dos apps de referência (`sorteio`, `ykanban`, `ikon`) no cluster.

Avaliei as 4 opções levantadas na tarefa:

| Opção | Avaliação |
|---|---|
| 1. RabbitMQ por namespace no Kubernetes | **Recomendada.** Simples de declarar via GitOps, isola HML de PRD (uma fila travada em HML não derruba PRD), não exige provisionar VM nova. Sem HA, mas aceitável para o porte atual (homelab). |
| 2. RabbitMQ compartilhado para o YCommunication | Não recomendada — misturaria tráfego de HML e PRD no mesmo broker, quebrando o isolamento que o resto da infra (namespaces, secrets, domínios) já mantém rigorosamente. |
| 3. RabbitMQ externo (VM dedicada, como o Postgres) | Viável e mais alinhado ao padrão de infra atual, mas exige provisionar uma VM nova — fora do que este agente pode fazer (a VM de agente não pode virar host de infra, e não há VM de RabbitMQ hoje). Fica como alternativa se o time preferir esse caminho. |
| 4. Usar RabbitMQ existente | Não se aplica — não há nenhum RabbitMQ existente no ambiente. |

**Não instalei RabbitMQ nesta etapa** (nem no cluster, nem na VM de agente) — segue como proposta. Criei os manifests da Opção 1 em `ycommunication/base/rabbitmq/` (`Deployment` com `rabbitmq:3-management-alpine`, `Service`, `PersistentVolumeClaim` de 2Gi), lendo `RABBITMQ_DEFAULT_USER`/`RABBITMQ_DEFAULT_PASS` do mesmo Secret `ycommunication-api-secrets`. **Esses arquivos não estão referenciados em nenhum `kustomization.yaml` de overlay** — portanto o ArgoCD não vai aplicá-los. Para ativar, após aprovação do time, basta adicionar `../../base/rabbitmq` à lista de `resources:` do overlay desejado (e ajustar `storageClassName` no `pvc.yaml` conforme o que existir no cluster `k8s-lab`, que não verifiquei).

## 7. Commit e push

- Commit: `feat: add ycommunication kubernetes manifests`
- Branch: `main` (repositório `homelab-gitops` não segue o fluxo hml/prd — é aplicado diretamente, como os demais apps de referência)
- Push: realizado para `origin/main`

## 8. Fora do escopo desta etapa (confirmado)

- ❌ PRs `hml→main` dos repositórios de aplicação — **não mergeados**.
- ❌ Imagens PRD — **não publicadas manualmente**.
- ❌ Kubernetes/K3s/ArgoCD/Traefik — **nada instalado** na VM `yakuza-dev-agent-01`.
- ❌ RabbitMQ — **não instalado** em lugar nenhum (proposta apenas, ver seção 6).
- ❌ Nenhum secret real foi commitado.
- ❌ Nenhuma alteração destrutiva ou de regra de negócio do YCommunication.
- ❌ Testes flaky de webhook (`WebhookDeliveryE2EIT`, `WebhookSsrfDeliveryBlockedIT`) — não tocados, seguem como risco conhecido documentado na etapa de CI/CD.

## 9. Próxima etapa recomendada — subir apenas HML

1. Executar os passos 1–4 da seção 5 (bancos, usuário Postgres, Secret, `ghcr-secret`) no namespace `ycommunication-hml` na VM `k8s-lab`.
2. Decidir a abordagem de RabbitMQ (seção 6) e, se for a Opção 1, adicionar `../../base/rabbitmq` ao `overlays/hml/kustomization.yaml` e commitar.
3. Criar a Application do ArgoCD apontando para `ycommunication/overlays/hml`.
4. Criar os 2 hostnames de HML no Cloudflare Tunnel/DNS.
5. Deixar o ArgoCD sincronizar e validar:
   ```bash
   kubectl get pods,svc,ingress -n ycommunication-hml
   kubectl logs -n ycommunication-hml deploy/ycommunication-api
   curl -s https://api-ycommunication-hml.yakuzastudio.com.br/actuator/health
   ```
6. Só depois de HML validado e estável: mergear os PRs `hml→main` (backend e frontend) para gerar `prd-latest`, então repetir os passos acima para `ycommunication-prd`.
