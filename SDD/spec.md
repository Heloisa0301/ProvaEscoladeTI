# SPEC — Zona Azul Digital

Comportamento da API REST. Gere `src/server.js` (um único arquivo) conforme UC-01 a UC-10; os demais arquivos seguem o plan.md e o constitution.md. Todo critério de aceite (AC) é verificável por HTTP, exceto o AC-10.3.

## Contrato básico
- Node.js 20, Express 4, better-sqlite3; tudo JSON; rotas e campos em português; um processo nas portas 8002 e 8000 com UMA conexão SQLite compartilhada (`:memory:` por padrão, `DB_PATH` quando informado).
- Variante: hora 450 centavos, fração 30 min (225 centavos), teto 5000 por bilhete, tolerância 15 min.
- Erros `{"erro":"<codigo>"}`: 422 placa_invalida, entrada_invalida, data_invalida; 404 bilhete_nao_encontrado, rota_nao_encontrada; 409 bilhete_em_aberto, bilhete_ja_encerrado, bilhete_nao_aberto; 500 erro_interno (só falha interna). NUNCA 400.
- Bilhete em consultas e listas: `id`, `placa`, `entrada`, `status` (aberto, encerrado, cancelado). Aberto e cancelado têm 4 chaves; encerrado tem 7 (acrescenta `saida`, `minutos`, `valor_centavos`). EXCEÇÃO: a resposta imediata de POST /bilhetes/{id}/encerramento tem exatamente 6 chaves, sem `status` (AC-02.1). Chaves ausentes são omitidas, nunca `null`.
- Datas: `entrada` omitida e `saida` = `Date.now()` serializado como `AAAA-MM-DDTHH:MM:SS-03:00` (segundos, sem milissegundos).
- Entradas: `{id}` válido = `^[1-9][0-9]*$`; qualquer outro valor (0, 01, +1, 1.5, 1abc) ou id inexistente → 404 `bilhete_nao_encontrado`. Query params não definidos para a rota são ignorados; param repetido (array) é inválido: `placa` → `[]`, `data` → 422.

## UC-01 — Abrir bilhete (POST /bilhetes, corpo `{"placa","entrada"?}`)
- **AC-01.1** 201 com exatamente `id`, `placa`, `entrada`, `status` "aberto"; `entrada` omitida ou `null` = agora.
- **AC-01.2** Placa fora de `^[A-Z0-9]{7}$` (ausente, null, número, minúscula, hífen, espaço, 6 ou 8 caracteres), corpo ausente, vazio, malformado ou array → 422 `placa_invalida`. Só dígitos e só letras são válidas. Campos extras são ignorados (`id` é do servidor).
- **AC-01.3** `entrada` informada: `AAAA-MM-DDTHH:MM[:SS[.fff]]` com `Z`, `±HH:MM` ou sem offset (assume -03:00), em data e hora existentes; qualquer outra coisa (só data, texto, vazio, número) → 422 `entrada_invalida`. Se a string terminar em `-03:00`, a API a PRESERVA (mesmo sem segundos); nos demais casos devolve o mesmo instante normalizado. Entrada futura é aceita.
- **AC-01.4** Ordem de validação: placa, entrada, conflito. Placa e entrada inválidas = `placa_invalida`; entrada inválida em placa ocupada = `entrada_invalida`, não 409.

## UC-02 — Encerrar (POST /bilhetes/{id}/encerramento)
- **AC-02.1** Aberto → 200 com exatamente 6 chaves: `id`, `placa`, `entrada`, `saida` (agora), `minutos`, `valor_centavos`; sem `status`, sem `valor`. Não exige corpo; qualquer corpo é ignorado.
- **AC-02.2** `minutos` = floor((saida − entrada) / 60 s), mínimo 0; o valor segue o UC-07.
- **AC-02.3** Entrada futura pode ser encerrada: 200, `minutos` 0, `valor_centavos` 0.
- **AC-02.4** Encerrado ou cancelado → 409 `bilhete_ja_encerrado` (decisão deliberada: o enunciado só lista este código para encerramento); id inválido ou inexistente → 404.

## UC-07 — Tolerância gratuita e valor (regra do UC-02)
- **AC-07.1** `minutos` <= 15 → valor 0; senão `min(ceil(minutos / 30) * 225, 5000)`. A tolerância NÃO é descontada.
- **AC-07.2** Limites: 14 e 15 → 0; 16 e 30 → 225; 31 e 60 → 450; 61 e 90 → 675; 91 → 900; 660 → 4950; 661, 1440 e 4320 → 5000. 15 min 50 s → `minutos` 15 → 0.

## UC-08 — Uma vaga por placa (regra do UC-01)
- **AC-08.1** Placa com bilhete `aberto` → 409 `bilhete_em_aberto`, mesmo com `entrada` diferente; o 409 não consome `id`.
- **AC-08.2** Após encerrar ou cancelar a mesma placa abre de novo (201, `id` novo); placas diferentes não interferem.

## UC-05 — Cancelar (POST /bilhetes/{id}/cancelamento)
- **AC-05.1** Aberto → 200 com `id`, `placa`, `entrada`, `status` "cancelado" (sem `saida`, `minutos`, `valor_centavos`); não exige corpo; encerrado ou cancelado → 409 `bilhete_nao_aberto`; id inválido ou inexistente → 404.

## UC-03, UC-06 e UC-09 — Consultas
- **AC-03.1** `GET /bilhetes/ativos` → 200 só com os `aberto`, por `entrada` decrescente (desempate `id` decrescente); vazio = `[]`. Registrar antes de `/bilhetes/:id`.
- **AC-06.1** `GET /bilhetes?placa=X` → 200 com todos os bilhetes da placa, qualquer status, mesma ordem; comparação exata (`===`), sem normalização nem busca parcial; placa nunca vista, malformada ou vazia → 200 `[]` (NÃO é erro, ao contrário do POST); sem `placa` → todos.
- **AC-09.1** `GET /bilhetes/{id}` → 200 com 4 chaves (aberto, cancelado) ou 7 (encerrado); id inválido ou inexistente → 404.

## UC-04 — Relatório (GET /relatorios/diario?data=AAAA-MM-DD)
- **AC-04.1** 200 com exatamente `data`, `total_bilhetes`, `faturamento_centavos`, `tempo_medio_minutos` (inteiros). Conta só `encerrado` cuja `saida`, em -03:00, cai na `data`; dia vazio → 0, 0, 0.
- **AC-04.2** Média ao inteiro mais próximo, 0,5 para cima (nunca half-even), em inteiros: `floor((2 * soma + n) / (2 * n))`. Ex.: [47, 48] → 48; [47, 48, 47] → 47; [47, 48, 47, 48, 48, 10] → 41.
- **AC-04.3** `data` ausente, vazia, `2026-10-5`, `05/10/2026`, `2026-02-30` ou `2026-02-29` → 422 `data_invalida`; `2028-02-29` → 200.

## UC-10 — Infraestrutura
- **AC-10.1** `GET /healthz` → 200 `{"status":"ok"}` nas duas portas, com o mesmo banco.
- **AC-10.2** Rota inexistente → 404 `{"erro":"rota_nao_encontrada"}`; toda resposta tem `Content-Type: application/json`; JSON malformado em POST /bilhetes → 422 `placa_invalida`.
- **AC-10.3** Falha interna → 500 `{"erro":"erro_interno"}` sem stack. Não é provocável por HTTP: não tem CT, e é PROIBIDO criar rota artificial para testá-lo.

Rastreabilidade (cenários no tests.md): UC-01 (CT-22 a CT-48), UC-02 e UC-07 (CT-49 a CT-76), UC-05 (CT-77 a CT-82), UC-09 (CT-83 a CT-85), UC-08 (CT-86 a CT-91), UC-03 (CT-92 a CT-96), UC-06 (CT-97 a CT-102), UC-04 (CT-01 a CT-21), UC-10 (CT-103 a CT-108).