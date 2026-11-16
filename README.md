# Home Ledger Web

Frontend para a Home-Ledger-API, desenvolvido em Vue 3, TypeScript e Vite.

## Ligar à API

O Vite encaminha automaticamente os pedidos `/api/*` para `http://localhost:8081`, a porta publicada pelo serviço `api` no Docker Compose da Home-Ledger-API. O browser comunica com o Vite e não precisa de acesso CORS direto à API.

1. Inicia a base de dados e a API no projeto `Home-Ledger-API` com `docker compose up -d database api`.
2. Neste projeto, executa:

```powershell
Copy-Item .env.example .env
npm install
npm run dev
```

3. Abre o endereço indicado pelo Vite, normalmente `http://localhost:5173`.

Se a API estiver publicada noutra porta, altera `VITE_PROXY_TARGET` no `.env` e reinicia o Vite. `VITE_API_URL` pode ser definido quando a aplicação vai comunicar diretamente com um endereço público da API; nesse caso, o servidor da API tem de permitir CORS para a origem do frontend.

## Funcionalidades

Inclui filtros por datas e categoria, resumo financeiro, gráficos mensais e por categoria, histórico e formulários para criar despesas e receitas. Os pedidos POST enviam um cabeçalho `Idempotency-Key` único, conforme exigido pela API.
