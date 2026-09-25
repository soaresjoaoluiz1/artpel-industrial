# Integração — LP Industrial Art Pel → Planilha → CRM Dros

## Fluxo

```
[Visitante preenche form em index.html]
        │
        ▼
[JS valida campos (nome, empresa, CNPJ, WhatsApp DDD+9, e-mail, cidade/UF, 4 selects)]
        │
        ▼
[Captura tracking: UTMs, gclid, fbclid, _fbc, _fbp, referrer, landing, device, userAgent]
        │
        ▼
[Meta Pixel: 'Lead' event (só qualificados) OU 'LeadDesqualificado' custom (não qualificados)]
        │
        ▼
[POST no-cors → Apps Script (fire-and-forget)]
        │
        ├──▶ [Apps Script grava linha na aba INDUSTRIAL/LOJA da planilha]
        │
        ├──▶ [Se qualificado (valor >R$1k e NÃO lojista): POST → CRM Dros webhook]
        │           │
        │           ▼
        │    [CRM Dros cria lead na conta art-pel-embalagens-ltda, aplica tag "LP-INDUSTRIAL"]
        │
        ├──▶ [Marca coluna "Status CRM" da planilha: "Enviado ✓ (200)" ou "Desqualificado — motivo"]
        │
        ▼
[Redirect: qualificado → obrigado.html (com WhatsApp CTA + Pixel 'Lead' 2ª camada + 'CompleteRegistration')]
[Redirect: desqualificado → obrigado-nao.html (só PageView + 'LeadDesqualificado')]
```

## Regra de qualificação (definida pelo cliente)

**NÃO envia pro CRM se:**
- Valor médio de compra = "Até R$ 1.000" (abaixo do mínimo comercial)
- Segmento = "Loja ou atacadista de embalagens" (perfil da LP2 futura, tag LP-LOJAS)

**Todos os outros** → envia pro CRM com tag `LP-INDUSTRIAL`, `trabalha_anuncio=true`, todos os UTMs/tracking em `observations`.

## Setup — passo a passo (uma vez só)

### 1. Publicar o Apps Script

1. Abrir a planilha: `https://docs.google.com/spreadsheets/d/1Zb_u9bTo0rImqFpy9AjvS1rOL7lEPeJd8ez50XxkTnc/edit`
2. Menu **Extensões → Apps Script**
3. Apagar o `Code.gs` vazio, colar o conteúdo de [`apps-script.gs`](apps-script.gs), salvar (Ctrl+S)
4. Antes de publicar, revisar as constantes no topo:
   - `SHEET_ID` — já preenchido
   - `CRM_WEBHOOK_URL` — já apontando pra `drosagencia.com.br/crm/api/webhooks/sheets/art-pel-embalagens-ltda`
   - `CRM_WEBHOOK_SECRET` — deixar `''` a menos que o CRM Dros exija autenticação
5. Clicar em **Implantar → Nova implantação**
6. Ícone engrenagem → **Tipo: App da Web**
7. Configurar:
   - Descrição: `Art Pel LP webhook v1`
   - Executar como: **Eu (agenciadouc@gmail.com)**
   - Quem tem acesso: **Qualquer pessoa**
8. Clicar **Implantar** → autorizar (aceita Planilha + URL Fetch)
9. Copiar a **URL do App da Web** (formato `https://script.google.com/macros/s/AKfy.../exec`)

### 2. Colar URL no `index.html`

Editar `index.html`, encontrar:
```js
const APPS_SCRIPT_URL='';
```
E trocar por:
```js
const APPS_SCRIPT_URL='https://script.google.com/macros/s/SEU_ID_AQUI/exec';
```

Commit + push + `git pull` no cPanel/servidor.

### 3. Configurar Meta Pixel + GA4

Em `index.html`, `obrigado.html` e `obrigado-nao.html`, trocar:
- `PIXEL_ID` → ID real do Pixel Meta (número de 15-16 dígitos)
- `G-XXXXXXXXXX` → ID real do GA4 (formato G-...)

Faz isso nos 3 arquivos e commita.

### 4. Configurar WhatsApp comercial

Em `obrigado.html`, trocar:
```js
const WHATSAPP='5548999999999';
```
Pelo número real da Art Pel (formato 55 + DDD + número, só dígitos).

### 5. Testar end-to-end

1. Abrir `https://www.artpelembalagens.com.br/?utm_source=teste&utm_medium=email&utm_campaign=setup`
2. Rolar até o formulário, preencher com CNPJ válido, WhatsApp válido, valor médio > R$ 1.000
3. Enviar
4. Verificar:
   - Redirect pra `obrigado.html`
   - Console do browser sem erros
   - Aba INDUSTRIAL da planilha ganhou linha nova com "Status CRM: Enviado ✓ (200)"
   - CRM Dros conta `art-pel-embalagens-ltda` recebeu o lead com tag `LP-INDUSTRIAL`
   - Meta Events Manager mostra evento `Lead` disparado
5. Repetir com valor "Até R$ 1.000" → deve ir pra `obrigado-nao.html` e planilha marcar "Desqualificado — valor < R$ 1k" (NÃO chega no CRM)

## Colunas da planilha

Cada aba (INDUSTRIAL e LOJA) tem 28 colunas gravadas automaticamente:

**Dados do form:**
1. Timestamp
2. Nome
3. Empresa
4. CNPJ
5. WhatsApp
6. E-mail
7. Cidade/Estado
8. Segmento
9. Valor médio
10. Frequência
11. Personalização

**Tracking:**
12-16. UTM Source / Medium / Campaign / Content / Term
17. GCLID (Google Ads)
18. FBCLID (Facebook Ads)
19-20. FBC / FBP (cookies Meta Pixel — melhora Advanced Matching)
21. Referrer
22. Landing Page
23. User Agent
24. Device (mobile/desktop)

**Controle:**
25. Event ID (dedup Pixel entre index + obrigado)
26. Qualificado (Sim/Não)
27. Status CRM (Enviado ✓ / Desqualificado / Erro)
28. CRM Response (payload retornado pra debug)

## Backfill (opcional)

Se algum lead teve erro no CRM (ex: CRM caiu por 5min), pode reprocessar todos os "Não enviado" e "Erro" rodando manualmente o menu do Apps Script:

**Executar → `backfillCRM`**

Ele varre as 2 abas e reenvia só os que precisam, pulando os desqualificados.

## Referências

- **Planilha:** https://docs.google.com/spreadsheets/d/1Zb_u9bTo0rImqFpy9AjvS1rOL7lEPeJd8ez50XxkTnc/edit
- **CRM Dros:** https://drosagencia.com.br/crm (conta `art-pel-embalagens-ltda`)
- **Repo LP1:** https://github.com/soaresjoaoluiz1/artpel-industrial
- **Repo LP2:** https://github.com/soaresjoaoluiz1/artpel-lojas
