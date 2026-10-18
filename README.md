# Quiz – Análise informativa (Direito do Trabalho)

Landing page em quiz no formato de conversa (balões de chat com a foto do Dr. Marcelo, progresso em segmentos), HTML/CSS/JS puro, sem build. Basta hospedar a pasta (Netlify, Vercel, Cloudflare Pages, S3, etc.): `index.html` + `avatar-dr-marcelo.webp` (foto redonda dos balões) + `logo-dr-marcelo.webp`.

## Configuração (topo do `<script>` em `index.html`)

| Constante | O que fazer |
|---|---|
| `WEBHOOK_URL` | Trocar `COLE_AQUI_A_URL_DO_WEBHOOK_N8N` pela URL de produção do nó **Webhook** do n8n (método POST). Se não for uma URL `http(s)`, o envio falha com mensagem amigável (nenhum lead é "engolido" silenciosamente). |
| `WHATSAPP_ESCRITORIO` | Número que enviará a mensagem automática (só dígitos, 55+DDD, ex.: `5592999999999`). Usado no botão "Abrir WhatsApp" da tela final. |
| `PIXEL_ID` | Trocar `COLE_AQUI_O_PIXEL_ID` pelo ID numérico do Pixel. Enquanto não for numérico, o Pixel fica desativado. |

Outros pontos a trocar: foto dos balões (arquivo `avatar-dr-marcelo.webp`, constante `AVATAR`) e logo (`logo-dr-marcelo.webp`), nome/OAB (`OAB/UF 000.000`, hero e rodapé), link da Política de Privacidade (`#politica-de-privacidade`), `og:image`, e a meta `noindex` (remova para indexar).

**CORS no n8n:** o navegador faz um preflight por causa do `Content-Type: application/json`. No nó Webhook, em *Options → Allowed Origins (CORS)*, informe o domínio da landing page (ou `*` para testar).

## Pixel / CAPI

| Evento | Quando | eventID |
|---|---|---|
| `PageView` | ao carregar | (padrão do Pixel) |
| `ViewContent` | clique em "Iniciar análise gratuita" | - |
| `Diagnostico10`, `Diagnostico25`, `Diagnostico50`, `Diagnostico75` (custom) | primeira vez que a barra de progresso atinge 10/25/50/75% (uma vez cada, mesmo se o usuário voltar). Parâmetro `progresso`. | `<id da visita>_Diagnostico<N>` |
| `Lead` (padrão) e `LeadDiagnostico` (custom) | só **após o webhook responder 2xx** (dados de contato enviados) | `event_id` do payload (o mesmo nos dois) |

O Meta não aceita "%" no nome do evento, por isso `Diagnostico10` etc. Se não quiser contar `Lead` e `LeadDiagnostico` em dobro, apague a linha `track('Lead', ...)` no script. Para a CAPI, use o mesmo nome e `event_id` de cada evento (e `fbp`/`fbc`) para deduplicar. O percentual é o da barra: com 12 etapas, 10% = 2ª resposta, 25% = 3ª, 50% = 6ª, 75% = 9ª (varia para quem ainda trabalha).

## Payload (POST JSON)

Exemplo real, capturado no teste de um lead quente (valores fictícios). Campos de topo, todos sempre presentes e em snake_case: `nome`, `whatsapp`, `email` (sempre vazio, mantido por compatibilidade), `cidade`, `estado`, `pais`, `ip` (cidade/estado/país/IP aproximados, obtidos por IP no navegador via ipwho.is; vazios se o serviço falhar ou for bloqueado), `respostas`, `score`, `classificacao`, `alertas`, `acao_em_andamento`, `resumo_texto`, `utm_source`, `utm_medium`, `utm_campaign`, `utm_content`, `utm_term`, `fbclid`, `fbp`, `fbc`, `url`, `user_agent`, `timestamp`, `event_id`. Vazios chegam como `""` (ou `[]` em `alertas`). As chaves internas de `respostas` são os enunciados em português, não snake_case.

```json
{
  "nome": "Maria da Silva",
  "whatsapp": "5511912345678",
  "email": "",
  "cidade": "Manaus",
  "estado": "Amazonas",
  "pais": "Brasil",
  "ip": "177.1.2.3",
  "respostas": {
    "Situação atual": "Fui demitido(a)",
    "Tempo desde a saída da empresa": "Há menos de 6 meses",
    "Tipo de vínculo": "Carteira assinada (CLT)",
    "Tempo de trabalho na empresa": "De 1 a 3 anos",
    "Situações ocorridas": ["Horas extras sem pagamento", "FGTS não depositado", "Assédio moral"],
    "Faixa salarial": "De R$ 2.000 a R$ 4.000",
    "Rescisão e documentos assinados": "Recebi valores, mas não entendi o cálculo",
    "Advogado ou ação anterior": "Não, ainda não",
    "Documentos disponíveis": ["Carteira de trabalho", "Holerites / contracheques", "Termo de rescisão (TRCT)"]
  },
  "score": 81,
  "classificacao": "quente",
  "alertas": [],
  "acao_em_andamento": false,
  "resumo_texto": "Novo lead - análise trabalhista\nNome: Maria da Silva\nClassificação: QUENTE (score 81/100)\nSituação atual: Fui demitido(a)\nTempo desde a saída da empresa: Há menos de 6 meses\nTipo de vínculo: Carteira assinada (CLT)\nTempo de trabalho na empresa: De 1 a 3 anos\nSituações ocorridas: Horas extras sem pagamento; FGTS não depositado; Assédio moral\nFaixa salarial: De R$ 2.000 a R$ 4.000\nRescisão e documentos assinados: Recebi valores, mas não entendi o cálculo\nAdvogado ou ação anterior: Não, ainda não\nDocumentos disponíveis: Carteira de trabalho; Holerites / contracheques; Termo de rescisão (TRCT)\nLocal (por IP): Manaus/Amazonas - Brasil",
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

## Localização por IP e privacidade

Cidade, estado, país e IP são obtidos no navegador via **ipwho.is** (terceiro) **somente depois que o usuário marca o consentimento**, e o texto do consentimento informa essa coleta. Se o serviço falhar ou for bloqueado, o lead segue sem localização. Cite o ipwho.is e a coleta de IP/localização na Política de Privacidade. Alternativa mais robusta: remover o `geoLookup()` e resolver a localização no n8n a partir do IP da requisição.

## Envio

Timeout de 10 s por tentativa, 3 tentativas automáticas (1 s e 2 s de espera). Se todas falharem, o formulário mantém os dados, mostra um aviso e o botão vira "Tentar novamente". O botão fica bloqueado durante o envio.

## Compliance (Provimento 205/2021 da OAB)

Texto informativo, sem promessa de resultado nem valores; rodapé com aviso, nome e OAB. **Revise o texto com o advogado** e preencha a OAB real. O Pixel carrega antes do consentimento LGPD (comum em tráfego pago); ajuste com um banner de cookies se o jurídico exigir.

## Teste local

`python -m http.server 8000` e abra `http://localhost:8000/?utm_source=teste`. Para testar o envio sem n8n, aponte `WEBHOOK_URL` para um endpoint como webhook.site.
