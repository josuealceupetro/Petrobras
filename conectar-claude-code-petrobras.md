# Conectar o Claude Code no VSCode (rede Petrobras)

Guia de referência para autenticar o **Claude Code** no VSCode quando a rede corporativa
bloqueia o login automático. Inclui o diagnóstico do problema, os bloqueios envolvidos e o
**procedimento rápido** para gerar o código de autorização em outro computador/dispositivo.

> Última verificação: 10/11/2026 — conta usada: assinatura **Claude.ai (Pro/Max)**.

---

## 1. O problema (resumo)

Ao fazer login no Claude Code dentro do VSCode, ele tenta abrir o navegador e **receber o
código automaticamente** em um endereço local (`http://localhost:<porta>/callback`). Na rede
da Petrobras esse retorno **não chega até o VSCode**, então o login nunca conclui e parece
"travar" ou dar erro.

**Sintomas típicos:**
- A tela de login abre no navegador, mas depois de autorizar aparece algo como
  `localhost:<porta> — This site can't be reached / ERR_CONNECTION_REFUSED`.
- O VSCode continua "aguardando login" indefinidamente.
- Ao tentar o login por e-mail, clicar no **link** do e-mail não conecta (o login acontece
  fora da janela certa).

---

## 2. Por que acontece (os bloqueios)

| # | Bloqueio | Efeito |
|---|----------|--------|
| 1 | **Callback em `localhost` bloqueado** | O fluxo automático do Claude Code depende do navegador redirecionar para `http://localhost:<porta>/callback`. A rede/segurança corporativa impede esse retorno, então o código nunca chega ao VSCode. |
| 2 | **Link mágico do e-mail "escapa" da janela** | No login por e-mail, clicar no *link* abre o login em outro navegador/contexto. A sessão autentica lá, não onde você precisa. **Solução:** usar o *código* de verificação do e-mail, não o link. |
| 3 | **Navegadores isolados não enxergam o `localhost` da máquina** | Um navegador sandbox/remoto (ex.: o navegador interno do app Claude) tem um `localhost` próprio, que **não é** a máquina do VSCode. Por isso o fluxo automático também falha por ali. |

**Conclusão:** o fluxo automático (localhost) é inviável nesse ambiente. A saída é o
**fluxo manual de "colar o código"**, que não depende de `localhost` — a página mostra o
código na tela e você cola no VSCode.

---

## 3. Conceito-chave (por que dá para usar outro computador)

O código de autorização é gerado via OAuth com **PKCE**. O segredo (`code_verifier`) fica
**guardado no Claude Code do VSCode** que iniciou o login. A troca final do código por um
token é feita **pelo próprio VSCode**.

Ou seja: **a autorização no navegador pode ser feita em qualquer dispositivo** logado na
mesma conta Claude (um celular no 4G, um PC de casa, o navegador interno do app Claude…),
desde que você use a **URL do fluxo manual**. O código resultante (`código#state`) é colado
no VSCode, que completa tudo.

> Isso é o que permite "gerar o código em outro computador": a máquina bloqueada só precisa
> **receber e colar** o código; a autorização em si acontece onde a internet está liberada.

---

## 4. Procedimento rápido

### Passo A — No VSCode (máquina bloqueada)

1. Abra o painel **Claude Code** e inicie o login.
2. Em *"How do you want to log in?"* escolha **Claude.ai Subscription**
   (ou **Anthropic Console**, se for conta de API).
3. Vai aparecer a tela **"Continue in browser"** com:
   - um campo com a **URL de autorização** (`https://claude.com/cai/oauth/authorize?...`);
   - abaixo, o campo **"Or, paste your authorization code manually:"** (mostra `012345` em
     cinza — isso é só **exemplo**, não um limite de 6 dígitos).
4. **Copie a URL de autorização** e **deixe essa tela aberta** (não feche nem reinicie o
   login — ver §6).

### Passo B — Transforme a URL no "modo manual"

Na URL copiada, troque **apenas** o parâmetro `redirect_uri` para o callback que **exibe o
código**:

```
redirect_uri=https%3A%2F%2Fplatform.claude.com%2Foauth%2Fcode%2Fcallback
```

Mantenha **iguais** o `client_id`, os `scope`, o `code_challenge` e o `state`.

**Exemplo** (o que muda está destacado):

```
# ANTES (fluxo automático — não serve):
https://claude.com/cai/oauth/authorize?code=true&client_id=...&response_type=code
   &redirect_uri=http%3A%2F%2Flocalhost%3A51391%2Fcallback          <-- localhost
   &scope=...&code_challenge=XXXX&code_challenge_method=S256&state=YYYY

# DEPOIS (fluxo manual — exibe o código):
https://claude.com/cai/oauth/authorize?code=true&client_id=...&response_type=code
   &redirect_uri=https%3A%2F%2Fplatform.claude.com%2Foauth%2Fcode%2Fcallback   <-- manual
   &scope=...&code_challenge=XXXX&code_challenge_method=S256&state=YYYY
```

> Algumas versões do Claude Code já oferecem direto uma opção de login "colar código"
> (headless). Se existir, use-a e pule a edição manual da URL.

### Passo C — Em um dispositivo com internet liberada

Use **qualquer** dispositivo logado na **mesma conta Claude** (celular no 4G, PC de casa,
ou o navegador interno do app Claude Desktop):

1. Abra a **URL do modo manual** (a do Passo B).
2. Faça login no Claude.ai **se pedido** — se for por e-mail, use o **código de
   verificação** do e-mail, **não** o link.
3. Na tela *"Claude Code would like to connect…"*, clique em **Authorize**.
4. A página **"Authentication code"** mostra o código no formato:

   ```
   <código>#<state>
   ```

   Clique em **Copy code** (copie o texto **inteiro**, inclusive o `#` e o que vem depois).

### Passo D — De volta no VSCode

1. Na tela **"Continue in browser"**, clique no campo
   **"Or, paste your authorization code manually:"**.
2. **Cole o código inteiro** (`código#state`).
3. Clique em **Continue**.
4. Deve aparecer a confirmação de login / sua conta conectada. ✅

---

## 5. Checklist rápido (resumo de bolso)

1. VSCode → Claude Code → **Claude.ai Subscription** → copie a URL de *"Continue in browser"*.
2. Troque `redirect_uri` para `https://platform.claude.com/oauth/code/callback`.
3. Abra essa URL num dispositivo com internet liberada, logado na mesma conta.
4. Login por **código** (não link) → **Authorize**.
5. Copie o `código#state` exibido.
6. Cole no campo manual do VSCode → **Continue**.

---

## 6. Detalhes importantes / armadilhas

- **Não reinicie o login no VSCode** depois de copiar a URL. Reiniciar gera um novo
  `state`/`code_challenge` e **invalida** qualquer código já gerado.
- **Códigos expiram rápido** (poucos minutos). Gere e cole logo. Se expirar, repita os
  Passos B–D (a URL manual com o mesmo `state` da sessão atual do VSCode gera um novo código).
- **Use o código, nunca o link** no login por e-mail.
- **Cole o código completo**, com o `#` e o `state`. Faltando a parte depois do `#`, a
  validação falha.
- O campo que mostra `012345` **não** exige 6 dígitos — é só um placeholder de exemplo.
- Mesma **conta** nos dois lados: o dispositivo que autoriza precisa estar logado na mesma
  conta Claude que o VSCode vai usar.

---

## 7. Solução de problemas

| Erro / sintoma | Causa provável | O que fazer |
|----------------|----------------|-------------|
| `localhost:<porta>` — *This site can't be reached* / `ERR_CONNECTION_REFUSED` | Você abriu/esperou o fluxo **automático** (localhost). | Use o **fluxo manual** (§4, Passo B). |
| Login por e-mail não conecta ao clicar no link | O link abre em outro contexto e a sessão não volta pro lugar certo. | Faça login com o **código** do e-mail. |
| "Invalid code" / código recusado ao colar | Código **expirado** ou **truncado**, ou o login do VSCode foi reiniciado. | Gere um **código novo** com o `state` atual do VSCode e cole inteiro, rápido. |
| Campo de colar parece aceitar só 6 dígitos | Confusão com o placeholder `012345`. | Cole mesmo assim o código longo inteiro — o campo aceita. |
| Pede login toda vez no dispositivo de autorização | Navegador sem sessão salva. | Normal; basta logar por código e autorizar. |

---

## 8. Parâmetros de referência da URL de autorização

- **Endpoint:** `https://claude.com/cai/oauth/authorize` (conta Claude.ai / chat)
- **`response_type`:** `code`
- **`redirect_uri` (manual):** `https://platform.claude.com/oauth/code/callback`
- **`code_challenge_method`:** `S256`
- **`scope`:** `org:create_api_key user:profile user:inference user:sessions:claude_code user:mcp_servers user:file_upload user:plugins`
- **`code_challenge` / `state`:** gerados pela sessão do Claude Code no VSCode —
  **não altere**; apenas reaproveite os da URL que o VSCode mostrou.
