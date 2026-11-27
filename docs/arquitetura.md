# Arquitetura

```
 Navegador / app instalado (PWA)                     BTP Cloud Foundry
┌───────────────────────────────┐           ┌──────────────────────────────────────┐
│ UI5 (resenhachat.github.io)   │  HTTPS    │ CAP (resenhachat)                    │
│  • OData V4  /service/Chat ───┼──────────►│  • ChatService (OData V4)            │
│  • /auth (login, refresh)  ───┼──────────►│  • /auth  (JWT próprio)              │
│  • WebSocket /ws?token=... ◄──┼──────────►│  • /ws    (RealtimeHub: presença,    │
│  • Service worker (push)      │           │            digitando, avisos)         │
└──────────────▲────────────────┘           │  • Web Push (VAPID)  ──┐             │
               │                            └────────────┬───────────┼─────────────┘
               │ notificação                             │           │
               └──── serviço de push do navegador ◄──────┼───────────┘
                                                         ▼
                                                  HANA Cloud (HDI)
```

## Backend — `resenhachat`

- **SAP CAP 9** em Node + TypeScript, `cds-typer` para tipos. SQLite em memória no desenvolvimento, HANA Cloud em produção (MTA).
- **Camadas** (`srv/`): `routes` → `controllers` → `services` (regra de negócio, `Either`) → `repositories` (único lugar com CQL),
  `models`, `factories` (injeção de dependência).
- **Autenticação própria**: `POST /auth/register|login|refresh|logout`. Senha com bcrypt, access token JWT (1 h) e refresh token
  rotativo (30 dias, guardado com hash). Reuso de refresh token derruba todas as sessões do usuário.
- **Serviço `/service/Chat`** (só usuário autenticado): `People`, `Me`, `MySettings`, `Conversations`, `Members`, `Messages`,
  `PublicChannels` e as actions `SendMessage`, `MarkAsRead`, `StartDirect`, `CreateConversation`, `AddMembers`, `RemoveMember`,
  `SetMemberRole`, `LeaveConversation`, `JoinChannel`, `PushPublicKey`, `SubscribePush`, `UnsubscribePush`, `SendPushTest`.
  Toda leitura é filtrada pelo usuário: conversa e mensagens só de quem é membro; e-mail e senha de terceiros nunca saem.
- **Papéis**: quem cria um grupo ou canal vira dono. Dono e admins editam a conversa e gerenciam membros; só o dono promove e
  rebaixa admins. Canal com "só admins publicam" bloqueia o envio no servidor. Cada mudança vira um aviso do sistema na conversa,
  gravado como chave i18n + parâmetros e traduzido na leitura para o idioma de quem lê (`Accept-Language`).
- **Tempo real**: toda escrita passa pelo OData (validada e transacional). Depois do commit, o service publica um evento no
  `RealtimeHub`, que avisa pelo WebSocket só quem pode ver. O front relê pelo OData. Eventos: `presence.snapshot`, `presence.changed`,
  `message.created`, `conversation.read`, `conversation.created`, `conversation.updated`, `members.changed`, `user.updated`, `typing`
  (contrato em `srv/realtime/events.ts`).
- **Mídia**: binário no banco. Upload em 2 passos: action `CreateUpload` → `PUT /Uploads(ID)/Content` (stream com limite contado:
  foto 10 MB, vídeo e arquivo 25 MB) → `SendMessage` com `Upload_ID`. Fotos e avatares são recodificados com `sharp` (webp, sem
  metadados, limite de pixels); arquivos saem sempre como anexo (`octet-stream`, `nosniff`); executáveis são recusados.
  Leitura (`Attachments(ID)/Content|Thumbnail`, `People|Me|Conversations(ID)/Avatar`) aceita o **token de mídia** (`?mt=`, JWT só para
  GET de stream, 24 h), para `<img src>` funcionar; o parâmetro é removido da URL antes do log.
- **Push**: mensagem nova gera Web Push para membros offline, não silenciados e com notificações ligadas.

## Frontend — `resenhachat.github.io`

- **SAP UI5** freestyle em TypeScript, namespace `apps.dflc.resenha`, identidade visual própria (fonte Inter, gradiente azul (cor do logo),
  tema claro/escuro).
- **MVC modular**: views XML só com layout e binding; controllers e seções (`controller/chat/*Section.ts`) tratam eventos;
  `service/` encapsula OData (`ChatService`, `PushService`), WebSocket (`RealtimeService`) e upload binário (`MediaService`, XHR com progresso); `auth/AuthService` é o único com `fetch` para JSON.
- **OData V4 primeiro**: listas, perfil e configurações ligados direto no modelo OData; JSONModel só para estado de tela
  (sessão, presença, rascunho).
- **PWA**: `manifest.webmanifest`, service worker `sw.js` (push e clique na notificação), instalação no Chrome/Edge e passo a passo
  no iPhone.

## Decisões

| Tema | Decisão |
|---|---|
| Login | Cadastro próprio (sem conta BTP para cada usuário) |
| Tempo real | WebSocket próprio no mesmo processo do CAP; presença em memória (1 instância) |
| Mídia | Binário no banco com limite de tamanho e compressão de imagem |
| Escala | Mais de 1 instância exigirá Redis pub/sub para o WebSocket |
