# Rodar localmente

## Pré-requisitos

- Node.js 20 ou mais novo
- Git
- Opcional: `npm i -g @sap/cds-dk` para ter o comando `cds` global. Sem ele, use os scripts `npm run ...` ou `npx cds ...`,
  que usam a versão instalada no projeto.

## 1. Backend (`resenhachat`)

```powershell
git clone https://github.com/ResenhaChat/resenhachat.git
cd resenhachat
npm install
npm run watch
```

Sobe em `http://localhost:4004` com banco em memória e dados de teste. Os dados voltam ao original a cada reinício.

Para notificações push, crie `resenhachat/.env` (não vai para o git):

```
VAPID_PUBLIC_KEY=<gerada com: npx web-push generate-vapid-keys>
VAPID_PRIVATE_KEY=<idem>
VAPID_SUBJECT=mailto:seu-email
APP_URL=http://localhost:8080/index.html
```

Sem o `.env`, o chat funciona normalmente e só o push fica desligado.

## 2. Frontend (`resenhachat.github.io`)

```powershell
git clone https://github.com/ResenhaChat/resenhachat.github.io.git
cd resenhachat.github.io
npm install
npm start
```

Abre em `http://localhost:8080/index.html`. O proxy local encaminha `/service` e `/auth` para o backend; o WebSocket conecta direto em
`ws://localhost:4004/ws`.

## Usuários de teste

| E-mail | Usuário | Senha |
|---|---|---|
| davi@resenha.dev | davi | 123456 |
| ana@resenha.dev | ana | 123456 |
| joao@resenha.dev | joao | 123456 |

Para ver o tempo real, entre com um usuário no navegador normal e com outro numa janela anônima.

## Validação

```powershell
# backend
npm run typecheck
npx cds compile srv -s all --to edmx > $null

# frontend
npm run ts-typecheck
npm run lint
npm run build
```
