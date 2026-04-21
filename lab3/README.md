# Lab 3 — GitOps: Deploy Automatico com ArgoCD

**Disciplina:** Cloud Computing — Engenharia da Computacao, SETREM
**Pre-requisito:** Lab 2 (CI/CD com GitHub Actions)
**Duracao:** 75 minutos (pode ser dividido em 2 encontros)
**Tecnologias introduzidas:** GitHub Container Registry (GHCR), ArgoCD

---

## Pre-requisitos

```sh
# Rancher Desktop rodando e kubectl apontando para o cluster local?
kubectl get nodes           # deve mostrar 1 node com STATUS Ready

# Lab 2 concluido? Confirme visitando:
# github.com/setrem-cloud/<equipe>-labs → aba Actions → ultimo run verde
```

> **Windows:** Use o **PowerShell**, nao o CMD. O prompt deve comecar com `PS C:\...>`.

---

## Objetivo

Fechar o ciclo completo de GitOps: um `git push` resulta em deploy automatico no Kubernetes, sem tocar no cluster manualmente.

```
Equipe faz push no repositorio de labs
      |
      v
GitHub Actions roda testes
      |
      v
Pipeline builda a imagem e publica no GHCR (registry)
      |
      v
Pipeline atualiza a tag da imagem no manifest K8s (commit automatico)
      |
      v
ArgoCD detecta o commit e sincroniza o cluster
      |
      v
Rancher Desktop atualiza os pods automaticamente
```

---

## Conceitos-chave

### O que muda em relacao ao Lab 2?

| Lab 2 (CI/CD) | Lab 3 (GitOps) |
|----------------|-----------------|
| Pipeline testa e builda | Pipeline testa, builda **e publica** a imagem |
| Imagem existe so no pipeline | Imagem vai para um **registry** (GHCR) |
| Deploy nao faz parte do lab | **ArgoCD** faz o deploy automaticamente |
| Foco: proteger o codigo | Foco: **entregar** o codigo |

### O que e um registry de imagens?

No Lab 2, o pipeline buildava a imagem para validar o Dockerfile, mas a imagem nao ia para lugar nenhum. Para o Kubernetes puxar a imagem, ela precisa estar em um **registry** — um servidor que armazena e distribui imagens de container.

**GHCR (GitHub Container Registry)** e o registry do proprio GitHub:
- Gratuito para repositorios publicos
- Autenticacao integrada (o pipeline ja tem acesso via `GITHUB_TOKEN`, sem configurar nada extra)
- Endereco: `ghcr.io/setrem-cloud/<equipe>-labs`

### O que e ArgoCD?

ArgoCD e uma ferramenta de GitOps para Kubernetes. Ele faz uma coisa simples:

> "Monitora um repositorio Git. Se os manifests YAML mudaram, aplica as mudancas no cluster."

O repositorio Git vira a **fonte unica de verdade**. Ninguem faz `kubectl apply` manualmente — tudo passa pelo Git.

Duas configuracoes importantes:
- **selfHeal: true** — se alguem alterar algo direto no cluster, o ArgoCD reverte para o que esta no Git
- **prune: true** — se um recurso for removido do Git, o ArgoCD remove do cluster tambem

---

## Onde fazer este lab

Este lab e feito no repositorio de **labs** da equipe na org, dentro da pasta `lab3/`:

```
github.com/setrem-cloud/<equipe>-labs/
  lab2/          ← lab anterior
  lab3/          ← este lab
    api/
    k8s/
    argocd/
    Dockerfile
    .dockerignore
  .github/
    workflows/
      ci.yml     ← pipeline atualizado para lab 3
```

> **Depois de aprender aqui,** a equipe vai aplicar o mesmo fluxo no repo do projeto (`setrem-cloud/<equipe>`) para o CloudPonto.

Nos exemplos, usamos a equipe **gamma**. Substitua pelo nome da sua equipe.

---

## Passo a Passo

### Parte 1 — Preparar o repositorio de labs (10 min)

1. Clonar o repo de labs da equipe (se ainda nao tiver localmente):

```sh
git clone https://github.com/setrem-cloud/gamma-labs.git
cd gamma-labs
```

2. Copiar os arquivos do lab3:

> **ATENCAO — leia antes de copiar:** O `.github/` vai para a **raiz** do repo `<equipe>-labs/`, **nao dentro de `lab3/`**. O GitHub Actions so reconhece workflows em `.github/workflows/` na raiz. Os demais arquivos vao dentro de `lab3/`.

> **Sobre o pipeline do lab2:** O ci.yml do lab3 **substitui** o do lab2 (mesmo caminho: `.github/workflows/ci.yml`). Depois dessa copia, o repo tera apenas o pipeline do lab3 — que e mais completo (inclui push para GHCR e atualizacao do manifest). Os testes do lab2 continuam funcionando pois o job `test` permanece.

```sh
# Ajuste o caminho abaixo para onde voce clonou o gabarito
# Linux/Mac:  ~/Documents/ec-saas-pacheco
# Windows:    $HOME\Documents\ec-saas-pacheco

mkdir lab3
cp -r /caminho/para/ec-saas-pacheco/lab3/api          lab3/
cp -r /caminho/para/ec-saas-pacheco/lab3/k8s          lab3/
cp -r /caminho/para/ec-saas-pacheco/lab3/argocd       lab3/
cp    /caminho/para/ec-saas-pacheco/lab3/Dockerfile    lab3/
cp    /caminho/para/ec-saas-pacheco/lab3/.dockerignore lab3/

# Substituir o pipeline do lab2 pelo do lab3 (mais completo)
cp -r /caminho/para/ec-saas-pacheco/lab3/.github       .    # <- VAI PARA A RAIZ

# Verificar que o workflow ficou no lugar certo
ls .github/workflows/ci.yml
```

Estrutura final esperada:

```
<equipe>-labs/
├── .github/workflows/ci.yml   ← raiz (pipeline do lab3 substitui o do lab2)
├── lab2/                       ← lab anterior (mantido)
└── lab3/
    ├── api/
    ├── k8s/
    ├── argocd/
    ├── Dockerfile
    └── .dockerignore
```

3. **Editar `lab3/k8s/deployment.yaml`** — substituir pela equipe:

```yaml
image: ghcr.io/setrem-cloud/gamma-labs:latest
```

4. **Editar `lab3/argocd/application.yaml`** — substituir pela equipe:

```yaml
repoURL: https://github.com/setrem-cloud/gamma-labs.git
```

E verificar que o `path` aponta para `lab3/k8s`:

```yaml
path: lab3/k8s
```

5. Commit e push:

```sh
git add .
git commit -m "ci: lab3 - GitOps com ArgoCD e GHCR"
git push origin main
```

6. **Tornar o pacote publico** (apos o primeiro pipeline rodar):
   - GitHub > org `setrem-cloud` > aba **Packages** > `gamma-labs`
   - Package settings > Danger Zone > Change visibility > **Public**

> **Por que publico?** Para o Kubernetes puxar a imagem sem precisar configurar credenciais. Em producao, usariamos um `imagePullSecret`.

---

### Parte 2 — Observar o pipeline completo (10 min)

Acesse a aba **Actions** no repositorio de labs. O pipeline tem 3 jobs:

| Job | PR | Merge na `main` |
|-----|----|-----------------|
| `test` | Roda | Roda |
| `build-and-push` | Builda (sem push) | Builda **e publica** no GHCR |
| `update-manifests` | Pulado | Atualiza tag no `deployment.yaml` e faz commit |

> **Por que build sem push no PR?** Para validar o Dockerfile *antes* do merge — se ele estiver quebrado, o PR nao pode entrar. O push para o registry so faz sentido quando o codigo ja passou pela revisao e foi aceito na `main`.

**Observe o ultimo job (`update-manifests`):** ele altera o `deployment.yaml` com a tag exata do commit. Isso e o que o ArgoCD vai detectar.

**Pontos de verificacao (apos push na `main`):**
- Os 3 jobs devem estar verdes
- Na aba **Packages** da org, a imagem deve aparecer
- No historico de commits, deve ter um commit automatico do `github-actions[bot]`

---

### Parte 3 — Instalar ArgoCD no Rancher Desktop (15 min)

O ArgoCD roda **dentro** do seu cluster Kubernetes local.

```sh
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Aguardar pods ficarem prontos (~2 min)
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=180s
kubectl get pods -n argocd
# Todos devem estar Running
```

### Acessar o painel web do ArgoCD

```sh
# Expor o painel na porta 8443
# Este comando ocupa o terminal — deixe-o rodando
kubectl port-forward svc/argocd-server -n argocd 8443:443
```

> **Iniciante:** O `port-forward` fica rodando em loop — e normal o terminal parecer "travado". Abra uma **nova janela do terminal** (nao feche esta) para continuar os proximos comandos.

Na nova janela, obter a senha do admin:

```sh
# Linux/Mac:
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

```powershell
# Windows (PowerShell):
$encoded = kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}"
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($encoded))
```

Acesse no navegador: **https://localhost:8443**
- Usuario: `admin`
- Senha: a que apareceu no comando acima

> **O navegador vai avisar sobre certificado inseguro** — e esperado (certificado auto-assinado). Clique em "Avancado" > "Prosseguir".

**Ponto de verificacao:** Voce esta logado no painel do ArgoCD (tela vazia, sem aplicacoes).

---

### Parte 4 — Conectar ArgoCD ao repositorio de labs (15 min)

Agora vamos dizer ao ArgoCD: "monitore o repositorio de labs da equipe e deploye no cluster".

```sh
kubectl apply -f lab3/argocd/application.yaml
```

**O que esse arquivo faz:**
- `source.repoURL` — repositorio de labs na org (`setrem-cloud/gamma-labs`)
- `source.path: lab3/k8s` — onde estao os manifests YAML
- `destination.server` — qual cluster (o proprio Rancher Desktop)
- `destination.namespace` — namespace da equipe (`equipe-gamma`)
- `syncPolicy.automated` — sincronizar automaticamente quando detectar mudancas

### Verificar no painel

1. Abra o ArgoCD (https://localhost:8443)
2. A aplicacao `lab3-api` deve aparecer
3. Status inicial: **OutOfSync** (ainda nao sincronizou)
4. Clique em **Sync** > **Synchronize**
5. Aguarde os pods subirem — status muda para **Synced** e **Healthy** (verde)

### Verificar no terminal

```sh
# Pods rodando
kubectl get pods -n equipe-gamma

# Servico exposto
kubectl get svc -n equipe-gamma

# Testar a API (Linux/Mac)
curl http://localhost/
curl http://localhost/healthz
```

> **Windows (PowerShell):** use `Invoke-RestMethod http://localhost/` no lugar de `curl`.

**Ponto de verificacao:** A API responde em `localhost:80` e o ArgoCD mostra tudo verde.

---

### Parte 5 — O momento magico: GitOps em acao (15 min)

Agora vem o ciclo completo. A equipe vai alterar o codigo e ver o deploy acontecer **sem tocar no cluster**.

#### 5.1 Fazer uma mudanca visivel

Edite `lab3/api/server.js` — mude a versao:

```javascript
// ANTES
versao: '1.0.0',

// DEPOIS
versao: '2.0.0',
```

#### 5.2 Commit e push

```sh
git add lab3/api/server.js
git commit -m "feat: atualizar versao para 2.0.0"
git push origin main
```

#### 5.3 Observar o ciclo completo

1. **GitHub Actions** (aba Actions): pipeline roda testes → builda imagem → publica no GHCR → atualiza tag no `deployment.yaml`
2. **ArgoCD** (painel web): detecta o novo commit → status vira **OutOfSync** → sincroniza automaticamente → pods atualizam
3. **Resultado**: a versao 2.0.0 esta no ar sem ninguem ter feito `kubectl apply`

```sh
# Acompanhar os pods atualizando (rolling update)
kubectl get pods -n equipe-gamma -w
# Ctrl+C para sair quando os pods estiverem Running

# Testar a nova versao (Linux/Mac)
curl http://localhost/
# versao deve mostrar "2.0.0"
```

> **Windows (PowerShell):** use `Invoke-RestMethod http://localhost/`.

#### 5.4 Testar o selfHeal (bonus)

```sh
# Deletar um pod "na mao"
kubectl delete pod -n equipe-gamma -l app=lab3-api --wait=false
```

O ArgoCD vai recriar o pod automaticamente (selfHeal). Observe no painel — ele detecta a diferenca e corrige.

**Ponto de verificacao:** A resposta mostra `versao: 2.0.0` — deploy automatico funcionou.

---

### Desafio Extra (para quem terminou)

Escolha UM:

**A — Push manual para o GHCR (entender o que o pipeline faz por baixo)**

1. Criar o PAT no GitHub:
   - **Settings > Developer Settings > Personal Access Tokens > Tokens (classic)**
   - Permissoes: `write:packages`, `read:packages`
   - Copiar o token gerado

2. Login no GHCR:

```sh
# Linux/Mac:
echo "SEU_TOKEN" | docker login ghcr.io -u SEU-USUARIO --password-stdin
```

```powershell
# Windows (PowerShell) — pipe nao funciona aqui, passe direto:
docker login ghcr.io -u SEU-USUARIO -p "SEU_TOKEN"
```

3. Build e push manual:

```sh
docker build -t ghcr.io/setrem-cloud/gamma-labs:manual -f lab3/Dockerfile lab3/
docker push ghcr.io/setrem-cloud/gamma-labs:manual
```

4. Verificar em: GitHub > org > aba **Packages**. A tag `manual` deve aparecer.

> **Por que o pipeline nao precisa de PAT?** Porque o `GITHUB_TOKEN` e gerado automaticamente pelo GitHub Actions com permissao para o proprio repositorio.

**B — Rollback via Git**

1. Quebre a API (mude `/healthz` para retornar 500), faca push
2. Observe o ArgoCD deployar a versao quebrada
3. Reverta com `git revert HEAD` e faca push
4. Observe o ArgoCD deployar a versao corrigida — sem tocar no cluster

**C — Verificar o prune do ArgoCD**

1. Adicione um ConfigMap qualquer em `lab3/k8s/` e faca push
2. Observe o ArgoCD criar o ConfigMap no cluster
3. Remova o ConfigMap do `lab3/k8s/` e faca push
4. Observe o ArgoCD remover do cluster automaticamente (`prune: true`)

---

## Diagrama do ciclo completo

```
  +-----------+     push      +---------------------------+
  |           | ------------> |                           |
  |  VS Code  |               |  GitHub (org)             |
  |           |               |  setrem-cloud/            |
  +-----------+               |    gamma-labs              |
                              |                           |
                              |  1. Actions roda testes   |
                              |  2. Publica imagem no GHCR|
                              |  3. Atualiza tag no YAML  |
                              +-------------+-------------+
                                            |
                                   commit automatico
                                   (nova tag no manifest)
                                            |
                                            v
  +---------------+            +------------+-------------+
  |               |  detecta   |                          |
  |  Rancher      | <--------- |  ArgoCD                  |
  |  Desktop      |  sincro-   |  (monitora repo de labs) |
  |  (Kubernetes) |  niza      |                          |
  |               |            +--------------------------+
  +-------+-------+
          |
    pods atualizam
    automaticamente
```

> **Proximo passo:** Aplicar este mesmo fluxo no repo do projeto (`setrem-cloud/<equipe>`) para o CloudPonto real.

---

## Entrega

| Item | Como verificar |
|------|---------------|
| Lab no repo de labs da equipe | `github.com/setrem-cloud/<equipe>-labs` — pasta `lab3/` |
| Imagem publicada no GHCR | Aba Packages na org |
| Pipeline com 3 jobs funcionando | Aba Actions — ultimo run verde |
| ArgoCD com aplicacao Synced + Healthy | Screenshot do painel ArgoCD |
| Ciclo GitOps completo | Demonstrar ao professor: mudar codigo → push → pods atualizam sozinhos |

---

## Troubleshooting

| Erro | Causa provavel | Solucao |
|------|---------------|---------|
| Tela "Get started with GitHub Actions" | `.github/` ficou dentro de `lab3/` | `git mv lab3/.github .github && git commit -m "fix: mover workflow para raiz" && git push` |
| `npm ERR! ci can only install with an existing package-lock.json` | `package-lock.json` nao foi comitado | Rodar `cd lab3/api && npm install --package-lock-only` e comitar o arquivo |
| `COPY --from=builder /app/node_modules ... not found` | API sem `dependencies` no `package.json` | Dockerfile ja trata com `mkdir -p node_modules` apos `npm ci` |
| Pipeline falha no push para GHCR | Repositorio privado ou permissoes | Verificar que o repo e publico; o `GITHUB_TOKEN` ja tem acesso automatico |
| Imagem nao aparece em Packages | Primeiro push ainda nao completou | Aguardar pipeline terminar; depois tornar o pacote publico |
| ArgoCD nao detecta mudancas | Polling interval padrao e 3 min | Aguardar ou clicar **Refresh** no painel |
| Pods ficam `ImagePullBackOff` | Pacote GHCR ainda privado ou tag errada | Tornar publico em Package Settings; verificar tag no `deployment.yaml` |
| ArgoCD mostra `OutOfSync` mas nao aplica | Auto-sync pode demorar ate 3 min | Clicar **Sync** manual ou aguardar |
| `connection refused` no curl | Service ainda provisionando | Aguardar `kubectl get svc -n equipe-gamma` mostrar EXTERNAL-IP |
| Certificado inseguro no ArgoCD | Esperado — certificado auto-assinado | Clicar "Avancado" > "Prosseguir" no navegador |
| `kubectl port-forward` encerra sozinho | Terminal fechou ou timeout | Rodar o comando novamente |
| Membro sem acesso ao repo | Nao esta no Team da equipe | Professor adiciona via org Settings > Teams |

---

## Limpeza pos-lab

```sh
kubectl delete -f lab3/argocd/application.yaml
kubectl delete namespace argocd
kubectl delete namespace equipe-gamma
```

> **Nota:** O repositorio na org e os pacotes no GHCR continuam disponiveis. O aprendizado deste lab sera aplicado no repo do projeto (`setrem-cloud/<equipe>`) nas proximas etapas.
