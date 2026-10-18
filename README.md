# Quiz – Análise informativa (Direito do Trabalho)

Landing page em quiz, arquivo único (`index.html`), HTML/CSS/JS puro, sem build. Basta hospedar o arquivo (Netlify, Vercel, Cloudflare Pages, S3, etc.).

## Configuração (topo do `<script>` em `index.html`)

| Constante | O que fazer |
|---|---|
| `WEBHOOK_URL` | Trocar `COLE_AQUI_A_URL_DO_WEBHOOK_N8N` pela URL de produção do nó **Webhook** do n8n (método POST). Se não for uma URL `http(s)`, o envio falha com mensagem amigável (nenhum lead é "engolido" silenciosamente). |
| `PIXEL_ID` | Trocar `COLE_AQUI_O_PIXEL_ID` pelo ID numérico do Pixel. Enquanto não for numérico, o Pixel fica desativado. |

Outros pontos a trocar: foto (`<img id="foto">`, `src`), nome/OAB (`OAB/UF 000.000`, hero e rodapé), link da Política de Privacidade (`#politica-de-privacidade`), `og:image`, e a meta `noindex` (remova para indexar).

**CORS no n8n:** o navegador faz um preflight por causa do `Content-Type: application/json`. No nó Webhook, em *Options → Allowed Origins (CORS)*, informe o domínio da landing page (ou `*` para testar).

## Pixel / CAPI

- `PageView` ao carregar, `ViewContent` ao clicar em "Iniciar análise gratuita".
- `Lead` só dispara **após o webhook responder 2xx**, com `eventID` = `event_id` do payload. Use o mesmo `event_id` (e `fbp`/`fbc`) no evento da Conversions API para deduplicar.

## Payload (POST JSON)

Exemplo real, capturado no teste de um lead quente (valores fictícios). Campos de topo, todos sempre presentes e em snake_case: `nome`, `whatsapp`, `email`, `cidade`, `respostas`, `score`, `classificacao`, `alertas`, `acao_em_andamento`, `resumo_texto`, `utm_source`, `utm_medium`, `utm_campaign`, `utm_content`, `utm_term`, `fbclid`, `fbp`, `fbc`, `url`, `user_agent`, `timestamp`, `event_id`. Vazios chegam como `""` (ou `[]` em `alertas`). As chaves internas de `respostas` são os enunciados em português, não snake_case.

```json
{
  "nome": "Maria da Silva",
  "whatsapp": "5511912345678",
  "email": "maria@exemplo.com",
  "cidade": "São Paulo/SP",
  "respostas": {
    "Situação atual": "Fui demitido(a)",
    "Tempo desde a saída da empresa": "Há menos de 6 meses",
    "Tipo de vínculo": "Carteira assinada (CLT)",
    "Tempo de trabalho na empresa": "De 1 a 3 anos",
    "Situações ocorridas": ["Horas extras sem pagamento", "FGTS não depositado", "Assédio moral"],
    "Faixa salarial": "De R$ 2.000 a R$ 4.000",
    "Rescisão e documentos assinados": "Recebi valores, mas não entendi o cálculo",
    "Advogado ou ação anterior": "Não, ainda não",
    "Documentos disponíveis": ["Carteira de trabalho", "Holerites / contracheques", "Termo de rescisão (TRCT)"],
    "Preferência de contato": "O quanto antes"
  },
  "score": 81,
  "classificacao": "quente",
  "alertas": [],
  "acao_em_andamento": false,
  "resumo_texto": "Novo lead - análise trabalhista\nNome: Maria da Silva\nClassificação: QUENTE (score 81/100)\nSituação atual: Fui demitido(a)\nTempo desde a saída da empresa: Há menos de 6 meses\nTipo de vínculo: Carteira assinada (CLT)\nTempo de trabalho na empresa: De 1 a 3 anos\nSituações ocorridas: Horas extras sem pagamento; FGTS não depositado; Assédio moral\nFaixa salarial: De R$ 2.000 a R$ 4.000\nRescisão e documentos assinados: Recebi valores, mas não entendi o cálculo\nAdvogado ou ação anterior: Não, ainda não\nDocumentos disponíveis: Carteira de trabalho; Holerites / contracheques; Termo de rescisão (TRCT)\nPreferência de contato: O quanto antes\nCidade/UF: São Paulo/SP",
  "utm_source": "facebook",
  "utm_medium": "",
  "utm_campaign": "cmp",
  "utm_content": "",
  "utm_term": "",
  "fbclid": "ABC123",
  "fbp": "",
  "fbc": "fb.1.1792340463208.ABC123",
  "url": "http://localhost:8765/?utm_source=facebook&utm_campaign=cmp&fbclid=ABC123",
  "user_agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/154.0.0.0 Safari/537.36",
  "timestamp": "2026-10-18T16:21:03.209Z",
  "event_id": "35017370-33fb-48d3-8d9b-31d21d3f60bb"
}
```

Observações:
- `respostas` usa os enunciados curtos como chave; perguntas puladas (quem "ainda trabalha" não vê *tempo desde a saída* nem *rescisão*) não aparecem.
- `classificacao`: `quente` | `morno` | `frio` | `fora_do_prazo`. `alertas` pode conter `prazo_proximo` (saiu entre 1 e 2 anos), `acao_em_andamento`, `fora_do_prazo`.
- `resumo_texto` é texto puro com quebras de linha, pronto para uma mensagem de WhatsApp/Slack.
- `event_id` é gerado uma vez por lead e reaproveitado nas retentativas, então o n8n pode usá-lo como chave de idempotência.

## Regras de score (função `qualificar`, comentada no código)

Soma de pontos (0–100): tempo desde a saída (até 25), vínculo (até 15), tempo de empresa (até 15), situações relatadas (5 por item, até 20), documentos (2 por item, até 10), rescisão (até 8), sem ação anterior (+5), ainda trabalha (+10), ação em andamento (−30), ação encerrada (−10). **Quente ≥ 65, morno ≥ 40, frio < 40.** Saída há mais de 2 anos = `fora_do_prazo`, sempre (o lead é capturado, com mensagem honesta de que o prazo geral pode ter se encerrado).

## Envio

Timeout de 10 s por tentativa, 3 tentativas automáticas (1 s e 2 s de espera). Se todas falharem, o formulário mantém os dados, mostra um aviso e o botão vira "Tentar novamente". O botão fica bloqueado durante o envio.

## Compliance (Provimento 205/2021 da OAB)

Texto informativo, sem promessa de resultado nem valores; rodapé com aviso, nome e OAB. **Revise o texto com o advogado** e preencha a OAB real. O Pixel carrega antes do consentimento LGPD (comum em tráfego pago); ajuste com um banner de cookies se o jurídico exigir.

## Teste local

`python -m http.server 8000` e abra `http://localhost:8000/?utm_source=teste`. Para testar o envio sem n8n, aponte `WEBHOOK_URL` para um endpoint como webhook.site.
