# gestio-school-deploy

Pipelines reusáveis de deploy das APIs do **gestio-school** (Cloud Functions gen2).
Fork do `kodigilo/kodigilo-deploy` — o original segue servindo os demais
produtos; alterações daqui NÃO voltam pra lá.

## Workflows

| Workflow | Uso |
|---|---|
| `deploy-firebase-api.yml` | padrão (maioria das APIs) |
| `deploy-firebase-api-with-domain-functions.yml` | acls |
| `deploy-firebase-api-with-domain-functions-prebuilt.yml` | payments (compila no runner, sem devDeps no Cloud Build) |
| `deploy-firebase-api-with-siteid.yml` | genesis |
| `deploy-firebase-api-with-redis.yml` | ead, pedagogico, settings, templates, v3-peoples |
| `deploy-firebase-api-4gb.yml` | atendimento (variante 4GB, DATABASE_URL por TCP) |

## Diferenças em relação ao kodigilo-deploy

Pool de banco enxuto para serverless — conexão vive só enquanto a function
precisa dela:

- `connection_limit` na `DATABASE_URL` vem do input `dbConnLimit` (**default 2**;
  era fixo 5);
- env `DB_IDLE_TIMEOUT` (input `dbIdleTimeout`, **default 60s**) e
  `DB_MIN_IDLE=0` — lidos pelo pool do `@gestio-school/pkg` >= 0.3.2, que fecha
  conexões ociosas em vez de segurá-las por 30 min (default do driver mariadb).

APIs que geram arquivo/relatório e precisem de mais folga sobem os inputs no
`firebase.yml` delas:

```yaml
    with:
      nomeDaFuncao: v1_templates
      dbConnLimit: "4"
      dbIdleTimeout: "300"
```

> **Atenção:** para os repos da org usarem estes workflows, em
> Settings → Actions → General → Access deste repo deve estar
> "Accessible from repositories owned by the organization".

## Runner self-hosted (Mac mini `mac03`)

Todos os workflows têm um job `probe` (ubuntu-latest, segundos) que consulta a
API de runners da org: se houver runner `macOS` online (esperando até 2 min),
o `build_and_deploy` roda em `[self-hosted, macOS, ARM64]`; senão cai para
`ubuntu-latest`. Requisitos:

- secret `RUNNERS_READ_TOKEN` (no repo da API ou na org): PAT fine-grained
  com resource owner na org e permissão de organização **Self-hosted runners:
  Read-only**. Os callers usam `secrets: inherit`, então o reusável lê o
  secret sem declaração nem repasse — o caller não decide nada sobre runner.
  Sem o secret tudo roda no GitHub, com warning na sonda;
- o Mac mini não precisa de gcloud instalado: o `setup-gcloud` baixa o SDK
  para o tool cache do runner (só o primeiro run paga o download);
- os scripts em `scripts/` são bash 3.2/BSD-compatíveis (macOS); o `sed -i`
  nos workflows usa sufixo `.bak` pelo mesmo motivo;
- `env.yaml` e `firebase.json` são apagados ao final no self-hosted, já que o
  workspace persiste entre runs.
