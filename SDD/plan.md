# PLAN — Zona Azul Digital

COMO construir. Gere `package.json`, `src/server.js` (um único arquivo), `tests/api.test.js`, `Dockerfile`, `Containerfile` (idêntico), `.dockerignore`, `.gitignore` e `README.md`.

## Contrato que o código DEVE seguir
- Rotas (JSON, em português): POST /bilhetes (201), POST /bilhetes/{id}/encerramento e /cancelamento (200), GET /bilhetes/ativos (registrar ANTES de /bilhetes/:id), GET /bilhetes/{id}, GET /bilhetes?placa=, GET /relatorios/diario?data=AAAA-MM-DD, GET /healthz (`{"status":"ok"}`).
- Erros `{"erro":"<codigo>"}`: 422 placa_invalida, entrada_invalida, data_invalida; 404 bilhete_nao_encontrado, rota_nao_encontrada; 409 bilhete_em_aberto, bilhete_ja_encerrado, bilhete_nao_aberto; 500 erro_interno. NUNCA 400 nem HTML.
- Variante: hora 450 centavos, fração 30 min (225), teto 5000 por bilhete, tolerância 15 min (não descontada); portas 8002 (`PORT`) e 8000 (`ALT_PORT`).
- Bilhete: `id`, `placa`, `entrada`, `status` (4 chaves; encerrado acrescenta `saida`, `minutos`, `valor_centavos`). O POST de encerramento responde 6 chaves, sem `status`. Chaves ausentes são omitidas, nunca `null`.

## Decisões técnicas
- **D-01 Stack e arquivos:** Node.js 20, Express 4, better-sqlite3, backend em UM arquivo. Justificativa: o better-sqlite3 é síncrono (checar a placa e inserir cabem numa transação) e um arquivo só evita divergência entre janelas de geração.
- **D-02 Persistência e portas:** UMA conexão SQLite, criada no boot e compartilhada por dois listeners em 0.0.0.0 (`PORT` 8002, `ALT_PORT` 8000; iguais = um listener só); `:memory:` por padrão, `DB_PATH` opcional. Tabela `bilhetes` (`id` AUTOINCREMENT, `placa`, `entrada_ms`, `entrada_txt`, `status`, `saida_ms`, `minutos`, `valor_centavos`) com índice único parcial em `placa` onde `status='aberto'`; o 409 é checado antes do INSERT e não consome `id`. Justificativa: cada `:memory:` é um banco distinto, e sem serviço externo o banco nasce limpo.
- **D-03 Dinheiro e tempo:** centavos inteiros; instantes em ms UTC, exibidos por `new Date(ms - 10800000).toISOString().slice(0, 19) + '-03:00'`. Justificativa: float acumula erro (0.1 + 0.2 ≠ 0.3) e o Brasil não tem horário de verão desde 2019.
- **D-04 Valor:** constantes nomeadas no topo (450, 30, 5000, 15, 225), sem número mágico; `minutos = max(0, floor((saida_ms - entrada_ms) / 60000))`; `minutos <= 15` → 0, senão `min(ceil(minutos / 30) * 225, 5000)`. Média do relatório `floor((2 * soma + n) / (2 * n))` (n = 0 → 0). Justificativa: a fórmula inteira sobe o 0,5 sem erro de float.
- **D-05 Entrada:** regex `/^(\d{4})-(\d{2})-(\d{2})T(\d{2}):(\d{2})(?::(\d{2})(\.\d+)?)?(Z|[+-]\d{2}:\d{2})?$/` e depois calendário real (mês 1-12, dia existente com bissexto, hora 0-23, min e seg 0-59). Sem offset = -03:00; se terminar em `-03:00` preserve a string em `entrada_txt`, senão devolva o instante formatado por D-03. Justificativa: `Date.parse` aceita formatos fora do contrato. A fração `(\.\d+)` aceita 1 ou mais dígitos: para o instante use só os 3 primeiros (complete com zeros à direita se houver menos, descarte o resto, NUNCA arredonde); a string preservada mantém todos os dígitos recebidos.
- **D-06 Erros e middlewares:** `express.json()` só em POST /bilhetes, e erro do parser (400, 413, 415) vira 422 `placa_invalida`; encerramento e cancelamento ignoram o corpo; `:id` só vale se casar `^[1-9][0-9]*$`, senão 404; depois vêm as rotas, o 404 geral em JSON e um tratador final (500 `erro_interno`, sem stack). POST /bilhetes valida placa, entrada e conflito, nessa ordem. Justificativa: o padrão do Express responde 400 em HTML.
- **D-07 Listas e relatório:** `ORDER BY entrada_ms DESC, id DESC`; `GET /bilhetes?placa=` compara igualdade exata (sem placa = todos; vazia, malformada ou repetida = `[]`). Relatório: `data` por `^\d{4}-\d{2}-\d{2}$` mais calendário real (ausente, repetida ou inválida = 422); `inicio = Date.UTC(ano, mes - 1, dia) + 10800000`, `fim = inicio + 86400000`, filtro `status = 'encerrado' AND saida_ms >= inicio AND saida_ms < fim`. Justificativa: o dia do relatório é o da `saida` em -03:00.
- **D-08 Testes:** `node:test` com processo filho (`process.execPath`, `PORT=18002`, `ALT_PORT=18000`, URLs em `127.0.0.1`), um `test('CT-NN ...')` literal por CT do tests.md (50). Justificativa: sem dependência extra e testa o contrato real por HTTP.
- **D-09 Imagem:** `node:20-bookworm`, `npm install --omit=dev`, usuário `node`. Justificativa: o better-sqlite3 compila com as ferramentas da imagem completa se faltar o binário, e `npm ci` quebraria sem `package-lock.json`.

## Entrega
- `Dockerfile` (texto integral; o `Containerfile` é idêntico):
```dockerfile
FROM node:20-bookworm
ENV NODE_ENV=production PORT=8002 ALT_PORT=8000
WORKDIR /app
COPY package.json ./
RUN npm install --omit=dev --no-audit --no-fund
COPY src ./src
COPY tests ./tests
EXPOSE 8002 8000
USER node
CMD ["node", "src/server.js"]
```
- `package.json`: `name` zona-azul-digital, `private` true, `engines.node` >=20, `scripts.start` = `node src/server.js`, `scripts.test` = `node --test tests/api.test.js`, dependências `express` ^4.19.2 e `better-sqlite3` ^11.0.0, nada mais.
- `.gitignore` e `.dockerignore`: `node_modules`, `.env`, `*.db`, `npm-debug.log`.
- `README.md`: visão geral, tabela de rotas com os erros, variáveis `PORT`, `ALT_PORT`, `DB_PATH` e estes comandos:
```bash
npm install && npm start
npm test
docker build -t zona-azul-digital . && docker run --rm -p 8002:8002 -p 8000:8000 zona-azul-digital
podman build -t zona-azul-digital -f Containerfile . && podman run --rm -p 8002:8002 -p 8000:8000 zona-azul-digital
```

## Critérios de aceite
- **AC-P1** `npm start` responde `GET /healthz` com 200 em até 5 s, nas portas 8002 e 8000.
- **AC-P2** `npm test` executa 50 testes e todos passam.
- **AC-P3** `Dockerfile` e `Containerfile` são idênticos, com `EXPOSE` e `CMD`; o `package.json` lista tudo que o código importa.