# CONSTITUTION — Zona Azul Digital 

Regras invariantes para TODO arquivo gerado. DEVE, NUNCA, SEMPRE e PROIBIDO são obrigatórios. Gerar: `package.json`, `src/server.js`, `tests/api.test.js`, `Dockerfile`, `Containerfile`, `.dockerignore`, `.gitignore`, `README.md`.

## Resumo do contrato (completo no spec.md)
- Stack: Node.js 20, Express 4, better-sqlite3. Rotas, campos e erros em português, sem alias.
- Rotas: POST /bilhetes (201), POST /bilhetes/{id}/encerramento, POST /bilhetes/{id}/cancelamento, GET /bilhetes/ativos, GET /bilhetes/{id}, GET /bilhetes?placa=, GET /relatorios/diario?data=AAAA-MM-DD, GET /healthz.
- Erros `{"erro":"<codigo>"}`: 422 placa_invalida, entrada_invalida, data_invalida; 404 bilhete_nao_encontrado, rota_nao_encontrada; 409 bilhete_em_aberto, bilhete_ja_encerrado, bilhete_nao_aberto; 500 erro_interno.
- Variante: hora 450 centavos, fração 30 min (225 centavos), teto 5000, tolerância 15 min, portas 8002 e 8000.
- Valor: minutos <= 15 dá 0; senão min(ceil(minutos / 30) * 225, 5000). Exemplos: 31 min = 450; 661 min = 5000.

## Regras
- **R-01** Dinheiro SEMPRE em centavos inteiros; NUNCA ponto flutuante nem a chave `valor`.
- **R-02** Toda resposta, inclusive erros e rota inexistente, DEVE ser JSON; PROIBIDO HTML.
- **R-03** NUNCA responder 400: entrada inválida é 422, inexistente 404, conflito 409. O 500 `erro_interno` é só para falha interna inesperada, sem stack no corpo.
- **R-04** Saídas literais, entradas tolerantes: só `placa` (POST /bilhetes) e `data` (relatório) são obrigatórias; campos extras são ignorados.
- **R-05** Chaves ausentes são omitidas, NUNCA `null` (aberto e cancelado não têm `saida`, `minutos`, `valor_centavos`).
- **R-06** POST /bilhetes valida nesta ordem: placa, entrada, conflito. `/bilhetes/ativos` DEVE ser registrada antes de `/bilhetes/:id`.
- **R-07** O parser JSON só roda em POST /bilhetes e seu erro vira 422 `placa_invalida`. Encerramento e cancelamento NÃO exigem corpo e ignoram qualquer body.
- **R-08** A variante DEVE ficar em constantes nomeadas no topo de `src/server.js`; PROIBIDO número mágico.
- **R-09** A tolerância NUNCA é descontada dos minutos. `minutos` = floor(segundos / 60). Média em inteiros por `floor((2 * soma + n) / (2 * n))`: 0,5 sobe, NUNCA half-even.
- **R-10** Fuso SEMPRE -03:00 fixo, sem biblioteca de datas; instantes em ms UTC.
- **R-11** Um processo em 0.0.0.0 nas portas `PORT` (8002) e `ALT_PORT` (8000), com UMA única conexão SQLite compartilhada; sem serviço externo; schema criado no boot.
- **R-12** `Dockerfile` e `Containerfile` idênticos, ambos com `EXPOSE` e `CMD`; NUNCA `npm ci` sem lockfile.
- **R-13** O `package.json` DEVE listar tudo que o código e os testes importam, e nada além.
- **R-14** O README DEVE ter execução local, testes, Docker, Podman e a tabela de rotas com os erros.
- **R-15** `'use strict'`, `const`/`let`, `===`, sem código morto, sem log de body ou placa.
- **R-16** NUNCA commitar segredos, `.env` ou `*.db`; `.gitignore` e `.dockerignore` listam `node_modules`, `.env`, `*.db`.
- **R-17** Um `test('CT-NN ...')` literal por CT do tests.md, no mínimo o total declarado lá, sem laço; só HTTP, NUNCA importar `src/`; datas relativas a hoje.
- **R-18** Em caso de conflito entre os artefatos, a ordem de precedência é: constitution.md > spec.md > plan.md > tests.md > tasks.md. Nenhum arquivo de nível inferior pode alterar uma regra estabelecida por arquivo de nível superior.

## Verificação
| Regra | Como conferir |
| --- | --- |
| R-01, R-05 | CT-71 e CT-107 |
| R-02, R-03, R-07 | CT-30, CT-77 e CT-105 |
| R-11 | CT-104 (bilhete criado na 18002 aparece na 18000) |
| R-12, R-17 | `diff Dockerfile Containerfile` vazio; `grep -c "^test('CT-" tests/api.test.js` = total do tests.md |