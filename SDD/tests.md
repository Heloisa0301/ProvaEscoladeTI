# TESTS — Zona Azul Digital 

Casos de borda e de contrato. Gere `tests/api.test.js` com exatamente **50** chamadas literais `test('CT-NN ...', ...)`, na ordem da tabela, mais os demais arquivos do plan.md. Todo CT cita método, rota, corpo e resposta esperada.

**TOTAL DE CENÁRIOS: 50** (CT-01 a CT-50).

## Contrato resumido 
- Node.js 20, Express 4, better-sqlite3, tudo JSON, rotas e campos em português; portas 8002 e 8000 com o mesmo banco. Erros `{"erro":"<codigo>"}`: 422 placa_invalida, entrada_invalida, data_invalida; 404 bilhete_nao_encontrado, rota_nao_encontrada; 409 bilhete_em_aberto, bilhete_ja_encerrado, bilhete_nao_aberto. NUNCA 400.
- Valor: `minutos` = floor(segundos / 60); `minutos` <= 15 → 0, senão `min(ceil(minutos / 30) * 225, 5000)` (tolerância não descontada). Média do relatório ao inteiro mais próximo, 0,5 sobe.
- Bilhete: 4 chaves (`id`, `placa`, `entrada`, `status`); encerrado em consultas tem 7 (+ `saida`, `minutos`, `valor_centavos`); o POST de encerramento responde 6, sem `status`. Chaves ausentes são omitidas. `{id}` válido = `^[1-9][0-9]*$`.

## Notação e execução
- `A(N)` = agora − N min em `AAAA-MM-DDTHH:MM:SS-03:00`; `AS(N)` = agora − N s; `F(N)` = agora + N min; `U(N)` = `A(N)` em UTC com `Z`; `W(N)` = `A(N)` sem offset; `D(N)` = `AAAA-MM-DD` de hoje + N dias em -03:00; `hoje` = `D(0)`. Calcule `t = Date.now()` uma vez.
- Placa do CT-NN: `P` + NN com 6 dígitos (CT-22 = `P000022`); a 2ª placa do cenário usa `Q`, a 3ª `R`. "Encerra" e "cancela" = `POST /bilhetes/{id}/encerramento` ou `/cancelamento` do bilhete recém-aberto.
- CT-01 a CT-10 rodam PRIMEIRO, em instância nova e nessa ordem; os demais não dependem do estado (verifique por `id`, nunca por tamanho de lista). Não rode perto de 00:00 (-03:00). Datas fixas só nos CT-08 a CT-10.

## Cenários
| CT | UC | Requisição(ões) | Resposta esperada |
| --- | --- | --- | --- |
| CT-01 | 04 | `GET /relatorios/diario?data=hoje` (nenhum bilhete criado) | 200 `{"data":hoje,"total_bilhetes":0,"faturamento_centavos":0,"tempo_medio_minutos":0}` |
| CT-02 | 04 | `POST /bilhetes` `{"placa":"P000002","entrada":A(47)}`; encerra; relatório de hoje | encerra 200 com `minutos` 47, `valor_centavos` 450; relatório 1, 450, 47 |
| CT-03 | 04 | `POST /bilhetes` `{"placa":"P000003","entrada":A(48)}`; encerra; relatório de hoje | valor 450; relatório 2, 900, 48 (média 47,5 sobe) |
| CT-04 | 04 | `POST /bilhetes` `{"placa":"P000004","entrada":A(47)}`; encerra; relatório de hoje | relatório 3, 1350, 47 (média 47,33 desce) |
| CT-05 | 04 | abre `P000005` com `A(5)` (fica aberto); abre `Q000005` e cancela; relatório de hoje | aberto e cancelado não contam: continua 3, 1350, 47 |
| CT-06 | 04 | `POST /bilhetes` `{"placa":"P000006","entrada":A(1500)}`; encerra; relatório de hoje | `minutos` 1500, valor 5000 (teto); relatório 4, 6350, 411 (410,5 sobe; conta pelo dia da `saida`); exatamente 4 chaves, as três últimas inteiras |
| CT-07 | 04 | `GET /relatorios/diario?data=D(-1)` e `?data=D(1)` | 200 com `data` igual ao parâmetro e 0, 0, 0 |
| CT-08 | 04 | `?data=2026-02-30` e `?data=2026-02-29` (2026 não é bissexto) | as duas 422 `{"erro":"data_invalida"}` |
| CT-09 | 04 | `?data=2026-10-5`, `05/10/2026`, `abc`, `2026-13-01`, `data=` vazio e sem parâmetro | todas 422 `{"erro":"data_invalida"}` |
| CT-10 | 04 | `?data=2028-02-29` (bissexto) | 200 com `data` "2028-02-29" e 0, 0, 0 |
| CT-11 | 01 | `POST /bilhetes` `{"placa":"ABC1D23"}` | 201; exatamente `id`, `placa`, `entrada`, `status`; `id` inteiro >= 1; `status` "aberto"; `entrada` em -03:00 a menos de 5 s de agora |
| CT-12 | 01 | `POST /bilhetes` com placa `ABC1D2` (6), `ABC1D234` (8), `abc1d23`, `ABC-D23`, `ABC 123`, `ÁBC1D23` | cada uma 422 `{"erro":"placa_invalida"}` |
| CT-13 | 01 | `POST /bilhetes` `{}`, `{"placa":null}`, `{"placa":1234567}` e `["ABC1D23"]` | cada um 422 `{"erro":"placa_invalida"}` |
| CT-14 | 01 | `POST /bilhetes` sem corpo e sem `Content-Type`; e com `Content-Type: application/json` e corpo `{"placa":` | as duas 422 `{"erro":"placa_invalida"}` (nunca 400) |
| CT-15 | 01 | `POST /bilhetes` `{"placa":"P000015","foo":"bar","id":99}` | 201; sem chave `foo`; `id` diferente de 99 |
| CT-16 | 01 | `POST /bilhetes` `{"placa":"1234567"}` e `{"placa":"ABCDEFG"}` | 201 nas duas (só dígitos e só letras são válidas) |
| CT-17 | 01 | `POST /bilhetes` com `P000017` + `entrada` `A(20)`; `Q000017` + `U(20)`; `R000017` + `W(1440)`; `S000017` + `A(20)` com `.123456` inserido antes de `-03:00` | 201 nas quatro; `entrada` igual a `A(20)` (preservada), `A(20)` (UTC convertido), `A(1440)` (sem offset assume -03:00) e à string enviada com `.123456` (preservada com a fração) |
| CT-18 | 01 | `POST /bilhetes` placa `P000018` com `entrada` `"ontem"`, `D(-1)` (só data), `1730000000` e `""` | cada uma 422 `{"erro":"entrada_invalida"}` |
| CT-19 | 01 | `POST /bilhetes` `{"placa":"P000019","entrada":null}` | 201; `entrada` a menos de 5 s de agora |
| CT-20 | 01 | `POST /bilhetes` `{"placa":"P000020","entrada":F(60)}` (futura); encerra | 201; encerra 200 com `minutos` 0 e `valor_centavos` 0 |
| CT-21 | 01 | `{"placa":"abc","entrada":"xx"}`; abre `P000021`; repete `{"placa":"P000021","entrada":"xx"}` | 422 `placa_invalida` (placa vence); 201; 422 `entrada_invalida` (não 409) |
| CT-22 | 02 | abre `P000022` com `A(14)` e `Q000022` com `A(15)`; encerra cada | `minutos` 14 e 15; `valor_centavos` 0 e 0 (tolerância; 15 ainda é grátis) |
| CT-23 | 02 | abre com `A(16)` e `A(30)`; encerra cada | `minutos` 16 e 30; valor 225 e 225 (um passo além da tolerância; fração exata cobra 1 fração) |
| CT-24 | 02 | abre com `A(31)`, `A(45)` e `A(60)`; encerra cada | valor 450, 450 e 450 (+1 min cobra a seguinte; descontar a tolerância daria 225) |
| CT-25 | 02 | abre com `A(61)` e `A(90)`; encerra cada | valor 675 e 675 |
| CT-26 | 02 | abre `P000026` com `A(91)`; encerra | `minutos` 91, valor 900 |
| CT-27 | 02 | abre `P000027` com `A(660)`; encerra | `minutos` 660, valor 4950 (22 frações, logo abaixo do teto) |
| CT-28 | 02 | abre com `A(661)` e `A(4320)`; encerra cada | valor 5000 e 5000 (bruto 5175 ou mais: o teto entra, por bilhete) |
| CT-29 | 02 | abre `P000029` com `AS(950)` (15 min 50 s); encerra | `minutos` 15, valor 0 (floor dos minutos) |
| CT-30 | 02 | `POST /bilhetes` `{"placa":"P000030","entrada":A(45)}`; `POST /bilhetes/{id}/encerramento` com corpo `{"qualquer":"coisa"}` | 200; exatamente `id`, `placa`, `entrada`, `saida`, `minutos`, `valor_centavos` (sem `status`, sem `valor`); 45 e 450, inteiros; `entrada` igual a `A(45)`; `saida` em -03:00; corpo ignorado |
| CT-31 | 02 | `POST /bilhetes/999999/encerramento`, `/bilhetes/0/...`, `/bilhetes/01/...` e `/bilhetes/abc/...` | cada uma 404 `{"erro":"bilhete_nao_encontrado"}` |
| CT-32 | 02 | abre `P000032`; encerra; encerra de novo; abre `Q000032`; cancela; encerra | os dois segundos pedidos 409 `{"erro":"bilhete_ja_encerrado"}` |
| CT-33 | 02 | abre `P000033` com `A(45)`; encerra; `GET /bilhetes/{id}` | 200; exatamente `id`, `placa`, `entrada`, `status`, `saida`, `minutos`, `valor_centavos`; `status` "encerrado" |
| CT-34 | 05 | abre `P000034`; `POST /bilhetes/{id}/cancelamento` com `Content-Type: application/json` e corpo `{`; `GET /bilhetes/{id}` | 200 com exatamente `id`, `placa`, `entrada`, `status` "cancelado" (sem `saida`, `minutos`, `valor_centavos`; corpo ignorado); o GET devolve o mesmo |
| CT-35 | 05 | `POST /bilhetes/999999/cancelamento`, `/bilhetes/0/...`, `/bilhetes/01/...` e `/bilhetes/abc/...` | cada uma 404 `{"erro":"bilhete_nao_encontrado"}` |
| CT-36 | 05 | abre `P000036`, encerra, cancela; abre `Q000036`, cancela, cancela de novo | os dois últimos pedidos 409 `{"erro":"bilhete_nao_aberto"}` |
| CT-37 | 09 | abre `P000037`; `GET /bilhetes/{id}` | 200; exatamente `id`, `placa`, `entrada`, `status`; `status` "aberto" |
| CT-38 | 09 | `GET /bilhetes/999999`, `/bilhetes/0`, `/bilhetes/01` e `/bilhetes/abc` | cada uma 404 `{"erro":"bilhete_nao_encontrado"}` |
| CT-39 | 08 | abre `P000039`; repete o mesmo corpo; repete com `entrada` `A(5)` | 201; as duas repetições 409 `{"erro":"bilhete_em_aberto"}` |
| CT-40 | 08 | abre `P000040`, encerra, abre de novo; abre `Q000040`, cancela, abre de novo | cada nova abertura 201 com `id` maior que o anterior |
| CT-41 | 08 | abre `P000041` (`id` n); repete; abre `Q000041` | 201; 409; 201 com `id` n+1 (o 409 não consome `id`; placas independentes) |
| CT-42 | 03 | abre `P000042` (X); abre e encerra `Q000042` (Y); abre e cancela `R000042` (Z); `GET /bilhetes/ativos` | 200, array com X e sem Y nem Z; cada item com exatamente 4 chaves e `status` "aberto" |
| CT-43 | 03 | abre `P000043` com `A(120)` (a) e `Q000043` com `A(60)` (b); `GET /bilhetes/ativos` | b aparece antes de a (entrada mais recente primeiro) |
| CT-44 | 03 | abre `P000044` e `Q000044` com a mesma `A(90)`; `GET /bilhetes/ativos` | o de `id` maior aparece antes (desempate por `id` decrescente) |
| CT-45 | 06 | placa `P000045`: abre `A(180)` e encerra; abre `A(120)` e cancela; abre `A(60)`; `GET /bilhetes?placa=P000045` | 3 itens, `status` "aberto", "cancelado", "encerrado" nessa ordem; encerrado com 7 chaves, os outros com 4 |
| CT-46 | 06 | `GET /bilhetes?placa=ZZZ9Z99`, `?placa=abc` e `?placa=` | 200 `[]` nas três (não é erro) |
| CT-47 | 06 | abre `P000047` e `Q000047`; `GET /bilhetes?placa=P000047`; `GET /bilhetes` | 1 item com `placa` "P000047"; sem `placa` lista os dois `id` |
| CT-48 | 10 | `GET /healthz` nas portas 18002 e 18000; abre `P000048` na 18002; `GET /bilhetes/{id}` na 18000 | 200 `{"status":"ok"}` nas duas; 201; 200 com o mesmo bilhete (mesmo banco) |
| CT-49 | 10 | `GET /rota-inexistente` | 404 `{"erro":"rota_nao_encontrada"}` |
| CT-50 | 10 | `POST /bilhetes` `{"placa":"P000050"}` (201); `GET /bilhetes/ativos` (200); `GET /bilhetes/999999` (404); repete o POST (409); `POST /bilhetes` `{"placa":"x"}` (422) | todas as respostas com `Content-Type` iniciado em `application/json` |

**TOTAL DE CENÁRIOS: 50.**