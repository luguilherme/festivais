# OLGA Web — Especificação Técnica
**Versão 1.0 — base para Fase 1**

---

## 1. Visão geral

Sistema web de gestão de exibição para festivais de cinema, evoluindo o OLGA Suite (leitor local de XLSX) para uma plataforma online colaborativa com leitura e escrita em tempo real, autenticação por papel e controle de acesso por evento.

**Stack:**
- Frontend: HTML + CSS + JavaScript puro (sem framework)
- Hospedagem: GitHub Pages (HTTPS estático)
- Dados: Google Sheets (uma planilha permanente, multi-evento)
- Auth: Google OAuth 2.0 (via Google Identity Services)
- Proxy de escrita: Google Apps Script (Web App publicado)

---

## 2. Estrutura da planilha

### 2.1 Aba `filmes`

Cada linha é um filme. A coluna A é a chave de segurança para filtragem por evento.

| Col | Campo | Tipo | Papel que edita |
|-----|-------|------|-----------------|
| A | evento | texto | admin |
| B | código festival | texto | admin / programação |
| C | HD | texto | admin |
| D | título | texto | programação |
| E | idioma | texto | programação |
| F | leg DCP | texto | projeção |
| G | mostra | texto | programação |
| H | VF | texto | programação |
| I | lente | texto | projeção |
| J | som DCP | texto | projeção |
| K | creator | texto | projeção |
| L | kdm? | enum | programação (esperado) / projeção (sim, não) |
| M | status | enum | projeção |
| N | status legenda | enum | projeção / programação |
| O | obs | texto livre | projeção |
| P | CPL | texto | projeção |
| Q | resolução DCP | texto | projeção |
| T | duração DCP | hora (HH:mm) | projeção |
| V | dcpsize | número | projeção |
| X | fps DCP | número | projeção |
| Y | CPL DA OV | texto | projeção |
| Z | LINK DCP | URL | admin / programação |
| AB | legenda / transcrição | URL / texto | programação |

**Colunas não utilizadas pelo site** (dados de arquivo local, não DCP):
G_arquivo (col R), resolução arquivo (col R), duração arquivo (col S), filesize (col U), fps arquivo (col W), col AA — lidas somente, nunca escritas pelo site.

#### Valores controlados (dropdowns)

**status (col M):**
- DCP OK
- ERRO
- LINK PARA DOWNLOAD
- CANCELADO
- *(vazio)*

**status legenda (col N):**
- OK
- PRECISA DE ELETRÔNICA
- *(vazio)*

**kdm? (col L):**
- esperado ← programação pode selecionar
- sim ← projeção pode selecionar
- não ← projeção pode selecionar
- *(vazio)*

---

### 2.2 Aba `sessões`

Cada linha é uma sessão. A coluna P é a chave de segurança para filtragem por evento.

| Col | Campo | Tipo | Observação |
|-----|-------|------|------------|
| A | sessão | texto | chave primária |
| B | cinema | texto | |
| C | dia | data (dd/mm/yyyy) | |
| D | hora | hora (HH:mm) | |
| E | filme 1 | código | referência à col B de filmes |
| F | filme 2 | código | |
| G | filme 3 | código | |
| H | filme 4 | código | |
| I | filme 5 | código | |
| J | filme 6 | código | |
| K | filme 7 | código | |
| L | filme 8 | código | |
| M | filme 9 | código | |
| N | filme 10 | código | |
| O | total de filmes | fórmula COUNTA | o site nunca escreve nesta coluna |
| **P** | **evento** | texto | **chave de segurança — novo campo** |

---

### 2.3 Aba `config`

Controle de acesso. Cada linha é uma autorização. Uma pessoa pode ter múltiplas linhas (múltiplos eventos ou papéis).

| Col | Campo | Exemplo |
|-----|-------|---------|
| A | email | joao@gmail.com |
| B | papel | projecao |
| C | evento | cc2026 |

**Papéis válidos:**
- `admin` — acesso total a todos os eventos (col C ignorada, usar `*`)
- `projecao` — edita campos técnicos de DCP no evento autorizado
- `programacao` — edita campos editoriais e sessões no evento autorizado

---

## 3. Modelo de permissões

### 3.1 Matriz papel × operação

| Operação | admin | projecao | programacao |
|----------|-------|----------|-------------|
| Inserir filme | ✓ | ✓ | — |
| Deletar filme | ✓ | — | — |
| Editar campos de programação (D E G H N Z AB) | ✓ | — | ✓ |
| Editar campos de projeção (F I J K M O P Q T V X Y) | ✓ | ✓ | — |
| Editar kdm "esperado" | ✓ | — | ✓ |
| Editar kdm "sim/não" | ✓ | ✓ | — |
| Inserir sessão | ✓ | — | ✓ |
| Deletar sessão | ✓ | — | — |
| Editar sessão (B C D E–N P) | ✓ | — | ✓ |
| Trocar evento ativo | ✓ | — | — |
| Ler config | ✓ | — | — |

### 3.2 Escopo por evento

Toda leitura e escrita é filtrada pelo evento ativo da sessão do usuário. A coluna A (`filmes`) e a coluna P (`sessões`) são as chaves de filtro.

- Admin: pode trocar de evento livremente via seletor na UI
- Outros papéis: evento fixo conforme `config`, selecionável apenas se tiver mais de uma entrada

### 3.3 Segunda camada — Apps Script

O Apps Script valida novamente no servidor antes de escrever:
1. O token Google do request é verificado (tokeninfo)
2. O email extraído é consultado na aba `config`
3. O evento da escrita é comparado com o evento autorizado
4. O campo sendo escrito é comparado com os campos permitidos para o papel

Se qualquer verificação falhar, retorna HTTP 403 sem escrever.

---

## 4. Autenticação — Google OAuth 2.0

### 4.1 Biblioteca

Google Identity Services (GIS) — biblioteca oficial atual, substitui a antiga gapi.auth2.

```html
<script src="https://accounts.google.com/gsi/client"></script>
```

### 4.2 Fluxo de login

```
1. Página carrega → verifica token em sessionStorage
2. Token ausente ou expirado → exibe botão "Entrar com Google"
3. Usuário clica → popup Google OAuth
4. Callback recebe credential (JWT)
5. Site decodifica JWT (base64) → extrai email
6. Site lê aba config via Sheets API com service key pública
7. Monta perfil: { email, papeis: [{papel, evento}] }
8. Se papeis.length === 0 → tela "acesso não autorizado"
9. Se papeis.length === 1 → entra direto no evento
10. Se papeis.length > 1 → tela de seleção de evento
11. Armazena { credential, perfil, eventoAtivo } em sessionStorage
```

### 4.3 Configuração no Google Cloud Console

Recursos necessários:
- Projeto novo (ex: `olga-web`)
- APIs ativadas: **Google Sheets API**
- Credencial OAuth 2.0: tipo **Aplicativo da Web**
- Origens JavaScript autorizadas: `https://SEU-USUARIO.github.io`
- URIs de redirecionamento: não necessário (fluxo popup, não redirect)
- Client ID: exposto no HTML (é público por design do OAuth)

---

## 5. Leitura de dados — Sheets API

### 5.1 Autenticação da leitura

Leituras usam o token OAuth do usuário logado no header:

```javascript
fetch(url, {
  headers: { Authorization: `Bearer ${accessToken}` }
})
```

O `accessToken` é obtido do credential GIS após login.

### 5.2 Endpoints principais

Base URL: `https://sheets.googleapis.com/v4/spreadsheets/{SPREADSHEET_ID}`

| Operação | Endpoint | Parâmetros |
|----------|----------|------------|
| Ler filmes do evento | `/values/filmes!A:AB` | `valueRenderOption=FORMATTED_VALUE` |
| Ler sessões do evento | `/values/sessões!A:P` | `valueRenderOption=FORMATTED_VALUE` |
| Ler config | `/values/config!A:C` | |
| Ler cinemas distintos | derivado de sessões, col B | |

### 5.3 Filtragem por evento no cliente

A API do Sheets não filtra por valor de célula — retorna a aba inteira. A filtragem por evento acontece no JavaScript após a leitura:

```javascript
const filmes = rows.filter(r => r[0] === eventoAtivo);
const sessoes = rows.filter(r => r[15] === eventoAtivo); // col P = índice 15
```

---

## 6. Escrita de dados — Apps Script proxy

### 6.1 Por que proxy e não Sheets API direta

A Sheets API permite escrita direta com o token do usuário, mas isso exigiria que cada usuário tivesse permissão de editor na planilha — o que daria acesso irrestrito pela interface do Google Sheets. O proxy Apps Script mantém a planilha com acesso restrito (só admin e o script) e implementa a lógica de permissão centralizada.

### 6.2 Estrutura do Apps Script

```javascript
// Publicado como Web App:
// Executar como: sua conta (admin)
// Acesso: qualquer pessoa com conta Google

function doPost(e) {
  const body = JSON.parse(e.postData.contents);
  const { token, aba, operacao, dados, eventoAlvo } = body;

  // 1. Verificar token Google
  const email = verificarToken(token);
  if (!email) return erro(401, 'Token inválido');

  // 2. Verificar permissão na config
  const perfil = buscarPerfil(email);
  if (!perfil) return erro(403, 'Acesso não autorizado');

  // 3. Verificar escopo de evento
  if (perfil.papel !== 'admin' && perfil.evento !== eventoAlvo)
    return erro(403, 'Evento fora do escopo autorizado');

  // 4. Verificar campos permitidos para o papel
  if (!camposPermitidos(perfil.papel, aba, dados))
    return erro(403, 'Campo fora do escopo do papel');

  // 5. Executar operação
  return executar(aba, operacao, dados);
}
```

### 6.3 Operações suportadas

| operacao | Descrição |
|----------|-----------|
| `UPDATE_CELL` | Atualiza célula específica por código de filme ou nome de sessão |
| `INSERT_ROW` | Insere nova linha (filme ou sessão) |
| `DELETE_ROW` | Deleta linha por código (admin only) |

### 6.4 Payload padrão do site para o script

```javascript
{
  token: "eyJ...",           // credential JWT do usuário
  aba: "filmes",             // ou "sessoes"
  operacao: "UPDATE_CELL",
  eventoAlvo: "cc2026",
  dados: {
    chave: "PB08",           // código do filme ou nome da sessão
    campo: "M",              // coluna da planilha
    valor: "DCP OK"
  }
}
```

---

## 7. Datas e horas — garantia de formato

### 7.1 Estratégia

A planilha mantém o formato das colunas de data e hora. O site sempre escreve valores já formatados como string, deixando o Sheets interpretar com base no formato existente da coluna.

| Campo | Formato de escrita | Exemplo |
|-------|--------------------|---------|
| dia (sessões col C) | `"dd/mm/yyyy"` | `"03/10/2026"` |
| hora (sessões col D) | `"HH:mm"` | `"19:30"` |
| duração DCP (filmes col T) | `"H:mm"` | `"1:42"` |

O Apps Script usa `valueInputOption: "USER_ENTERED"` na escrita, que permite ao Sheets interpretar strings de data/hora conforme o locale da planilha (pt-BR).

### 7.2 Leitura

O site sempre lê com `valueRenderOption=FORMATTED_VALUE`, que retorna o valor já formatado como aparece na célula. O parser do OLGA Suite atual (que já lida com `HH:mm` e `dd/mm/yyyy`) é reaproveitado sem modificações.

---

## 8. Módulos do frontend

### 8.1 Páginas / rotas

Implementadas como seções visíveis/ocultas dentro de um único HTML (SPA sem router externo), com a URL hash como estado:

| Hash | Módulo |
|------|--------|
| `#dashboard` | Dashboard do evento |
| `#acervo` | Busca e edição de filmes |
| `#grade` | Grade visual de programação |
| `#ficha` | Ficha de sessão / composição |
| `#config` | Admin — gestão de usuários e eventos |

### 8.2 Dashboard (`#dashboard`)

Quatro buckets de status derivados da leitura combinada das colunas M e N:

| Bucket | Condição | Cor |
|--------|----------|-----|
| ✓ OK | M = "DCP OK" e N = "OK" | verde |
| ⚠ Pendência | M = "DCP OK" e N = "PRECISA DE ELETRÔNICA" | amarelo |
| ✗ Erro | M = "ERRO" | vermelho |
| ↓ Aguardando | M = "LINK PARA DOWNLOAD" ou M vazio | cinza azulado |

Alertas adicionais:
- KDM esperado sem confirmação: col L = "esperado"
- Filmes na programação sem entrada em filmes (código órfão)

Filtros: por mostra (col G), por cinema (via sessões)

### 8.3 Acervo (`#acervo`)

Busca e edição de filmes. Herda a lógica de busca atual (título, código, CPL, idioma, obs). Adiciona:
- Campos editáveis inline conforme papel do usuário
- Formulário de inserção de novo filme
- Ingestão via pasta DCP (File System Access API — leitura de CPL XML)

### 8.4 Grade (`#grade`)

Três modos de visualização:
- **Grid** — visualização atual (sala × horário), por dia
- **Por cinema** — todas as sessões de uma sala, todos os dias
- **Tabela** — lista ordenada por data/hora, imprimível

### 8.5 Ficha (`#ficha`)

Composição por filtros em cascata:

```
[ Tipo ] → [ filtros dependentes ] → [ Gerar ]

Sessão única   →  [ sessão ▼ ]
Dia × Cinema   →  [ dia ▼ ] [ cinema ▼ ]
Dia inteiro    →  [ dia ▼ ]
Evento inteiro →  (sem filtro)
```

Impressão: cada sessão tem `page-break-inside: avoid`. Sessões longas (dia inteiro, evento) quebram página entre sessões automaticamente.

### 8.6 Config (`#config`)

Visível apenas para admin. Exibe e edita a aba `config` diretamente (lista de usuários, papéis e eventos autorizados).

---

## 9. Ingestão via pasta DCP

Usa a **File System Access API** (`showDirectoryPicker()`). Disponível em Chrome/Edge; no Safari e Firefox requer fallback para upload de arquivo individual.

Campos extraídos do CPL XML:

| Campo XML | Campo planilha |
|-----------|----------------|
| `ContentTitleText` | — (comparação com título existente) |
| `Id` (UUID do CPL) | P (CPL) |
| `ContentKind` | — (informativo) |
| `EditRate` + `Duration` | T (duração DCP) |
| `ScreenAspectRatio` | Q (resolução DCP) |

Fluxo:
1. Usuário abre pasta DCP
2. Site percorre arquivos procurando `*_cpl.xml`
3. Extrai campos, apresenta formulário de confirmação
4. Usuário revisa e confirma
5. Site envia para Apps Script → escreve na planilha

---

## 10. Fases de desenvolvimento

### Fase 1 — Leitura online
- Setup Google Cloud (OAuth + Sheets API)
- Migração do carregamento: de XLSX local para fetch Sheets API
- Login Google + filtragem por evento
- Todos os três módulos existentes (ficha, grade, busca) funcionando com dados online

### Fase 2 — Dashboard
- Página inicial com buckets de status
- Alertas de KDM e filmes órfãos
- Filtros por mostra e cinema

### Fase 3 — Ficha composta
- Filtros em cascata
- Renderização multipágina
- CSS de impressão refinado

### Fase 4 — Escrita
- Apps Script (setup + deploy)
- Edição inline de campos no Acervo
- Inserção de novos filmes e sessões
- Ingestão via pasta DCP

### Fase 5 — Grade expandida
- Modo por cinema
- Modo tabela
- Módulo Config (gestão de usuários)

---

## 11. Repositório e deploy

```
olga-web/
├── index.html          ← aplicação completa
├── README.md
├── SPEC.md             ← este documento
└── apps-script/
    └── codigo.gs       ← código do Apps Script (referência)
```

**Deploy:** push para branch `main` → GitHub Pages publica automaticamente em `https://SEU-USUARIO.github.io/olga-web/`

**Variáveis de configuração** (no topo do `index.html`, sem necessidade de arquivo `.env`):
```javascript
const CONFIG = {
  SPREADSHEET_ID: "1abc...xyz",
  GOOGLE_CLIENT_ID: "123456-abc.apps.googleusercontent.com",
  APPS_SCRIPT_URL: "https://script.google.com/macros/s/.../exec"
};
```

Estas três constantes são os únicos valores que mudam entre ambientes. O `SPREADSHEET_ID` e o `APPS_SCRIPT_URL` não são segredos (a planilha tem acesso restrito por permissão, não por obscuridade de URL). O `GOOGLE_CLIENT_ID` é público por design do OAuth.

---

*Documento gerado como base para o início da Fase 1. Revisões devem ser commitadas junto com o código correspondente.*
