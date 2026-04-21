# Labs Cloud Computing — SETREM

Materiais de referencia dos labs da disciplina Cloud Computing (Engenharia da Computacao, 7o Semestre, SETREM).

## Estrutura
```
lab1/                    ← K8s basico (hello-world + PostgreSQL no Rancher Desktop)
lab2/                    ← CI/CD com GitHub Actions (pipeline de testes + build)
lab3/                    ← GitOps com ArgoCD + GHCR (deploy automatico)
guia-github-org/         ← Setup da org GitHub para a turma
```

## Organization da Turma

**Org:** https://github.com/setrem-cloud

Cada equipe tem dois repositorios na org:

| Repositorio | Conteudo |
|-------------|----------|
| `<equipe>-labs` | Exercicios dos labs (lab2/, lab3/) |
| `<equipe>` | Projeto integrador CloudPonto |

## Progressao dos Labs

| Lab | Tema | Onde fazer | Pre-requisito |
|-----|------|-----------|---------------|
| 1 | Kubernetes basico | Individual (Rancher Desktop) | Nenhum |
| 2 | CI/CD (GitHub Actions) | `<equipe>-labs` na org | Lab 1 |
| 3 | GitOps (ArgoCD + GHCR) | `<equipe>-labs` na org | Lab 2 |

## Stack

- **Kubernetes local:** Rancher Desktop (nao Docker Desktop, Minikube ou Kind)
- **CI/CD:** GitHub Actions
- **Registry:** GHCR (GitHub Container Registry)
- **GitOps:** ArgoCD
- **API dos labs:** Node.js 20
