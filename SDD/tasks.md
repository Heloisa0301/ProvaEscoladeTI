# TASKS — Zona Azul Digital 

Execute na ordem; cada tarefa só termina quando o "Pronto quando" é verificável. Gere `package.json`, `src/server.js` (um único arquivo), `tests/api.test.js`, `Dockerfile`, `Containerfile`, `.dockerignore`, `.gitignore` e `README.md`.

## Contrato resumido 
- Node.js 20, Express 4, better-sqlite3, tudo JSON em português. Erros `{"erro":"<codigo>"}`: 422 placa_invalida, entrada_invalida, data_invalida; 404 bilhete_nao_encontrado, rota_nao_encontrada; 409 bilhete_em_aberto, bilhete_ja_encerrado, bilhete_nao_aberto; 500 erro_interno. NUNCA 400.
- Variante: hora 450, fração 30 min (225), teto 5000 por bilhete, tolerância 15 min não descontada. Portas 8002 (`PORT`) e 8000 (`ALT_PORT`), UMA conexão SQLite compartilhada.
- Bilhete: 4 chaves (`id`, `placa`, `entrada`, `status`); encerrado em consultas tem 7; a resposta do POST de encerramento tem 6, sem `status`.

## Fundação
- [ ] **TK-01** Criar `package.json` (express ^4.19.2, better-sqlite3 ^11.0.0, scripts `start` e `test`), `.gitignore` e `.dockerignore` (`node_modules`, `.env`, `*.db`, `npm-debug.log`). Pronto quando: `npm install` conclui e instala só essas duas dependências.
- [ ] **TK-02** Esqueleto de `src/server.js`: `'use strict'`, constantes nomeadas da variante, UMA conexão SQLite (`:memory:` ou `DB_PATH`), schema com índice único parcial, `GET /healthz`, 404 geral em JSON, tratador de erro final (500 `erro_interno`) e os dois listeners em 0.0.0.0. Pronto quando: `/healthz` responde 200 nas duas portas e uma rota inexistente dá 404 `rota_nao_encontrada`.
- [ ] **TK-03** Funções de tempo: parse de `entrada` (regex e calendário real), formatação em -03:00 por deslocamento de 10800000 ms, validação de `data` e fronteiras do dia. Pronto quando: `2026-02-30`, hora 24, só data e texto livre são rejeitados e `2028-02-29` é aceito.

## Casos de uso
- [ ] **TK-04** UC-01 e UC-08, `POST /bilhetes`: `express.json()` só nesta rota, erro do parser e corpo inválido viram 422 `placa_invalida`, validação `^[A-Z0-9]{7}$`, ordem placa, entrada, conflito, 409 antes do INSERT sem consumir `id`. Pronto quando: CT-11 a CT-21 e CT-39 a CT-41 passam.
- [ ] **TK-05** UC-02 e UC-07, `POST /bilhetes/:id/encerramento`: `minutos` por floor, valor 0 até 15 min, `min(ceil(minutos / 30) * 225, 5000)` acima, resposta de 6 chaves, corpo ignorado, id fora de `^[1-9][0-9]*$` ou inexistente = 404, 409 `bilhete_ja_encerrado`. Pronto quando: CT-22 a CT-33 passam (31 min = 450, 660 = 4950, 661 = 5000).
- [ ] **TK-06** UC-05 e UC-09, `POST /bilhetes/:id/cancelamento` e `GET /bilhetes/:id`: 4 chaves para aberto e cancelado, 7 para encerrado, 404 e 409 `bilhete_nao_aberto`, corpo ignorado no cancelamento. Pronto quando: CT-34 a CT-38 passam.
- [ ] **TK-07** UC-03 e UC-06, `GET /bilhetes/ativos` (registrada ANTES de `/bilhetes/:id`) e `GET /bilhetes?placa=` com `ORDER BY entrada_ms DESC, id DESC`, placa exata, `[]` para vazia, malformada ou repetida, e todos sem `placa`. Pronto quando: CT-42 a CT-47 passam.
- [ ] **TK-08** UC-04, `GET /relatorios/diario`: só `encerrado` com `saida` no dia em -03:00, média por `floor((2 * soma + n) / (2 * n))`, dia vazio 0, 0, 0, `data` ausente, repetida ou inválida = 422. Pronto quando: CT-01 a CT-10 passam (médias 47, 48, 47 e 411 nos CT-02, CT-03, CT-04 e CT-06).

## Testes
- [ ] **TK-09** `tests/api.test.js`, harness: processo filho (`process.execPath`, `PORT=18002`, `ALT_PORT=18000`), espera de `/healthz` por até 10 s, encerramento no `after`, helpers de data e de abrir, encerrar e cancelar, URLs em `127.0.0.1`. Pronto quando: `node --test tests/api.test.js` roda e encerra sem processo órfão.
- [ ] **TK-10** Escrever os 50 `test('CT-NN ...')` literais do tests.md, na ordem, sem laço, só HTTP. Pronto quando: `grep -c "^test('CT-" tests/api.test.js` dá 50 e `npm test` passa todos.

## Entrega
- [ ] **TK-11** `Dockerfile` e `Containerfile` idênticos, texto integral do plan.md (`EXPOSE 8002 8000`, `CMD ["node", "src/server.js"]`). Pronto quando: `docker build` conclui, `/healthz` responde nas duas portas e `diff Dockerfile Containerfile` é vazio.
- [ ] **TK-12** `README.md` com visão geral, tabela de rotas com os erros, variáveis `PORT`, `ALT_PORT`, `DB_PATH`, e os comandos de execução local, testes, Docker e Podman do plan.md. Pronto quando: as quatro seções existem e os comandos batem com o `package.json`.
- [ ] **TK-13** Verificação final. Pronto quando: nenhum bloco de código nos `.md` passa de 20 linhas, nenhuma resposta tem chave `valor`, 400 ou HTML, e `npm test` passa 50 de 50.