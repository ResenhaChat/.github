# Deploy

## Frontend — GitHub Pages

- Todo push na `main` de `resenhachat.github.io` roda o workflow **Deploy UI5 App to GitHub Pages**: `npm run build` e publica `dist/`
  na branch `gh-pages`.
- Settings → Pages: **Deploy from a branch**, branch `gh-pages`, pasta `/ (root)`.
- Settings → Actions → General → Workflow permissions: **Read and write permissions**.
- `webapp/config/runtime-config.json` aponta para o backend em produção:
  ```json
  { "chatService": "https://<rota-do-cap>/service/Chat/", "authBase": "https://<rota-do-cap>/auth", "wsUrl": "" }
  ```
  `wsUrl` vazio = derivado do `chatService` (`wss://<rota>/ws`).

## Backend — BTP Cloud Foundry + HANA Cloud

- MTA `Resenha`: app `Resenha-srv` (Node, 256 MB), container HDI `Resenha-db` e deployer. Não usa XSUAA: a autenticação é própria.
- Pré-requisitos no BTP: instância HANA Cloud ligada no subaccount e cota para um app Node no space.
- Build e deploy manual:
  ```powershell
  cd resenhachat
  npm run build
  cf deploy mta_archives\Resenha.mtar
  ```
- Segredos do app (nunca no `mta.yaml`):
  ```powershell
  cf set-env Resenha-srv JWT_SECRET <32+ caracteres aleatórios>
  cf set-env Resenha-srv VAPID_PUBLIC_KEY <chave pública>
  cf set-env Resenha-srv VAPID_PRIVATE_KEY <chave privada>
  cf set-env Resenha-srv VAPID_SUBJECT mailto:<e-mail>
  cf set-env Resenha-srv APP_URL https://resenhachat.github.io/index.html
  cf restage Resenha-srv
  ```
  Sem `JWT_SECRET` válido o backend se recusa a subir em produção.
- CORS: o backend aceita `https://resenhachat.github.io` e `http://localhost:*`; outras origens via `CORS_ORIGINS` (separadas por vírgula).

## Secrets e variables do GitHub (repo `resenhachat`)

Usados pelos workflows de deploy do backend (mesmo modelo do projeto BenditaLook).

| Tipo | Nome | Conteúdo |
|---|---|---|
| Secret | `CF_USERNAME`, `CF_PASSWORD` | Usuário do BTP |
| Secret | `JWT_SECRET` | Segredo do JWT |
| Secret | `VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY` | Chaves do Web Push |
| Secret | `UI5_REPO_TOKEN` | Token fine-grained com Contents read/write em `resenhachat.github.io` |
| Variable | `CF_ORG`, `CF_SPACE`, `CF_API` | Destino no Cloud Foundry |
| Variable | `HANA_INSTANCE` | Nome da instância HANA (se houver mais de uma) |
| Variable | `VAPID_SUBJECT` | `mailto:` do responsável |
| Variable | `UI5_REPOSITORY` | `ResenhaChat/resenhachat.github.io` |
| Variable | `UI5_PAGES_URL` | `https://resenhachat.github.io/` |

## Notificações no celular

- Android (Chrome): instalar pelo menu do app e ativar as notificações.
- iPhone (iOS 16.4+): Safari → Compartilhar → Adicionar à Tela de Início; abrir pelo ícone e só então ativar as notificações.
