# OLGA Web — Especificação Técnica
**Versão 1.1**

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

A planilha é única e permanente. Cada evento convive nas mesmas abas, identificado pela coluna `evento` presente em todas elas.

### 2.1 Aba `filmes`

Cada linha é um filme. A coluna A é a chave de segurança para filtragem por evento.

| Col | Campo | Tipo | Papel que edita |
|-----|-------|------|-----------------|
| A | evento | texto | admin |
| B | código festival | texto | admin / programacao |
| C | HD | texto | admin |
| D | título | texto | programacao / producao |
| E | idioma | texto | programacao / producao |
| F | leg DCP | texto | projecao |
| G | mostra | texto | programacao / producao |
| H | VF | texto | programacao / producao |
| I | lente | texto | projecao |
| J | som DCP | texto | projecao |
| K | creator | texto | projecao |
| L | kdm? | enum | programacao / producao (esperado) · projecao (sim, não) |
| M | status | enum | projecao |
| N | status legenda | enum | projecao · programacao / producao |
| O | obs | texto livre | projecao |
| P | CPL | texto | projecao |
| Q | resolução DCP | texto | projecao |
| T | duração DCP | hora (H:mm) | projecao |
| V | dcpsize | número | projecao |
| X | fps DCP | número | projecao |
| Y | CPL DA OV | texto | projecao |
| Z | LINK DCP | URL | admin · programacao / producao |
| AB | legenda / transcrição | URL / texto | programacao / producao |

**Colunas não utilizadas pelo site** (dados de arquivo local, não DCP):
resolução arquivo (R), duração arquivo (S), filesize (U), fps arquivo (W), col AA — lidas somente, nunca escritas pelo site.

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
- `esperado` ← programacao / producao podem selecionar
- `sim` ← projecao pode selecionar
- `não` ← projecao pode selecionar
- *(vazio)*

---

### 2.2 Aba `sessões`

Cada linha é uma sessão. A coluna P é a chave de segurança para filtragem por evento.

| Col | Campo | Tipo | Observação |
|-----|-------|------|------------|
| A | código sessão | texto (10 chars, caixa alta) | chave primária |
| B | cinema | texto | referência à col B de `cinemas` |
| C | dia | data (dd/mm/yyyy) | |
| D | hora | hora (HH:mm) | |
| E | filme 1 | código | referência à col B de `filmes` |
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
| P | evento | texto | chave de segurança — automático |
| Q | mostra | texto | dropdown gerado da col G do evento ativo |
| R | nome da sessão | texto livre | |

#### Código de sessão — geração automática

O site sugere um código seguindo o padrão:

```
[PREFIXO DO EVENTO] + [SIGLA DO CINEMA] + [SEQUENCIAL 2 dígitos]
ex: CC26MIS01 · ECN_CIN03
```

Regras de validação ao sair do campo:
- Convertido automaticamente para caixa alta
- Limitado a 10 caracteres
- Verificado contra registros existentes do evento (duplicata rejeitada antes de salvar)

O programador pode editar livremente dentro dessas regras.

---

### 2.3 Aba `cinemas`

Cada linha é uma sala. Uma sala pode reaparecer em eventos diferentes (linhas distintas com o mesmo nome mas evento diferente).

| Col | Campo | Tipo | Obrigatório |
|-----|-------|------|-------------|
| A | evento | texto | sim (automático) |
| B | nome da sala | texto | **sim — único obrigatório** |
| C | complexo | texto | não |
| D | modelo do servidor | texto | não |
| E | modelo do projetor | texto | não |
| F | serial do servidor | texto | não |
| G | contatos | texto livre | não |
| H | datas de uso | texto serializado | não — ver formato abaixo |
| I | horário funcionamento | texto | não — ex: "10h–23h" |
| J | capacidade | número | não |
| K | assentos acessíveis | número | não |
| L | Cineassist | sim/não | não — sistema Dolby de acessibilidade |
| M | monitores — nomes | texto livre | não |
| N | monitores — contatos | texto livre | não |

#### Formato de datas de uso (col H)

Texto serializado com períodos separados por vírgula. Cada período é uma data única ou um intervalo:

```
03/10–05/10, 08/10, 11/10–14/10
```

O site apresenta como **chips editáveis** no formulário — cada chip é um período, clicável para editar e com `×` para remover. O botão "+ adicionar período" abre um mini-formulário com data início e data fim (fim opcional para dias únicos).

O dashboard usa essa informação para alertar quando uma sessão está sendo agendada fora do período de uso da sala.

#### Monitores (cols M e N)

Múltiplos monitores são serializados com ` / ` como separador:

```
col M:  Ana Silva / Pedro Costa
col N:  99999-0000 / 88888-1111
```

O formulário apresenta como linhas dinâmicas (nome + contato por linha). O campo nome tem autocomplete contra a aba `equipe` do evento ativo — ao selecionar uma sugestão, o campo de contato é preenchido automaticamente com o telefone da pessoa. O usuário pode ignorar as sugestões e preencher livremente.

---

### 2.4 Aba `equipe`

Cada linha é um membro da equipe do evento. Alimentada pelo papel `producao`.

| Col | Campo | Tipo | Obrigatório |
|-----|-------|------|-------------|
| A | evento | texto | sim (automático) |
| B | nome | texto | **sim** |
| C | função | texto | não |
| D | telefone | texto | não |
| E | email | texto | não |

Lista editável no módulo de equipe — nova entrada via formulário, edição inline, deleção pelo papel `producao` ou `admin`.

Os nomes da equipe alimentam o autocomplete do campo monitores em `cinemas`. O vínculo é apenas de conveniência — não há chave estrangeira real.

---

### 2.5 Aba `config`

Controle de acesso. Cada linha é uma autorização. Uma pessoa pode ter múltiplas linhas (múltiplos eventos ou papéis).

| Col | Campo | Exemplo |
|-----|-------|---------|
| A | email | joao@gmail.com |
| B | papel | projecao |
| C | evento | cc2026 |

**Papéis válidos:**

| Papel | Descrição |
|-------|-----------|
| `admin` | Acesso total a todos os eventos. Col C ignorada — usar `*` |
| `projecao` | Edita campos técnicos de DCP no evento autorizado |
| `programacao` | Edita campos editoriais, sessões e cinemas no evento autorizado |
| `producao` | Mesmos poderes de `programacao` + gestão da equipe |

---

## 3. Modelo de permissões

### 3.1 Matriz papel × operação

| Operação | admin | projecao | programacao | producao |
|----------|:-----:|:--------:|:-----------:|:--------:|
| Inserir filme | ✓ | ✓ | — | — |
| Deletar filme | ✓ | — | — | — |
| Editar campos de programação (D E G H N Z AB) | ✓ | — | ✓ | ✓ |
| Editar campos de projeção (F I J K M O P Q T V X Y) | ✓ | ✓ | — | — |
| Editar kdm "esperado" | ✓ | — | ✓ | ✓ |
| Editar kdm "sim/não" | ✓ | ✓ | — | — |
| Inserir sessão | ✓ | — | ✓ | ✓ |
| Deletar sessão | ✓ | — | — | — |
| Editar sessão (B C D E–N P Q R) | ✓ | — | ✓ | ✓ |
| Inserir / editar cinema | ✓ | — | ✓ | ✓ |
| Deletar cinema | ✓ | — | — | — |
| Inserir / editar / deletar equipe | ✓ | — | — | ✓ |
| Trocar evento ativo | ✓ | — | — | — |
| Ler / editar config | ✓ | — | — | — |

### 3.2 Escopo por evento

Toda leitura e escrita é filtrada pelo evento ativo da sessão do usuário:

- `filmes` → col A
- `sessões` → col P
- `cinemas` → col A
- `equipe` → col A

Comportamento no login:

```
1 evento autorizado  →  entra direto
2+ eventos           →  tela de seleção de evento
0 eventos            →  tela "acesso não autorizado"
Admin                →  seletor de evento sempre visível na UI
```

### 3.3 Segunda camada — validação no Apps Script

Antes de qualquer escrita, o Apps Script verifica:

1. Token Google do request (via tokeninfo)
2. Email extraído consultado na aba `config`
3. Evento da escrita comparado com o evento autorizado para o usuário
4. Campo sendo escrito comparado com os campos permitidos para o papel

Qualquer falha retorna HTTP 403 sem escrever.

---

## 4. Autenticação — Google OAuth 2.0

### 4.1 Biblioteca

Google Identity Services (GIS):

```html
<script src="https://accounts.google.com/gsi/client"></script>
```

### 4.2 Fluxo de login

```
1.  Página carrega → verifica token em sessionStorage
2.  Token ausente ou expirado → exibe botão "Entrar com Google"
3.  Usuário clica → popup Google OAuth
4.  Callback recebe credential (JWT)
5.  Site decodifica JWT (base64) → extrai email
6.  Site lê aba config via Sheets API
7.  Monta perfil: { email, papeis: [{papel, evento}] }
8.  0 papéis → "acesso não autorizado"
9.  1 evento  → entra direto
10. 2+ eventos → tela de seleção de evento
11. Armazena { credential, perfil, eventoAtivo } em sessionStorage
```

### 4.3 Configuração no Google Cloud Console

- Projeto: `olga-web`
- APIs ativadas: Google Sheets API
- Credencial OAuth 2.0: tipo Aplicativo da Web
- Origens JavaScript autorizadas: `https://SEU-USUARIO.github.io`
- URIs de redirecionamento: não necessário (fluxo popup)

---

## 5. Leitura de dados — Sheets API

### 5.1 Autenticação

```javascript
fetch(url, {
  headers: { Authorization: `Bearer ${accessToken}` }
})
```

### 5.2 Endpoints principais

Base URL: `https://sheets.googleapis.com/v4/spreadsheets/{SPREADSHEET_ID}`

| Operação | Range | Parâmetros |
|----------|-------|------------|
| Ler filmes | `filmes!A:AB` | `valueRenderOption=FORMATTED_VALUE` |
| Ler sessões | `sessões!A:R` | `valueRenderOption=FORMATTED_VALUE` |
| Ler cinemas | `cinemas!A:N` | `valueRenderOption=FORMATTED_VALUE` |
| Ler equipe | `equipe!A:E` | `valueRenderOption=FORMATTED_VALUE` |
| Ler config | `config!A:C` | |

### 5.3 Filtragem por evento no cliente

A API retorna a aba inteira. Filtragem por evento acontece no JavaScript:

```javascript
const filmes  = rows.filter(r => r[0]  === eventoAtivo);
const sessoes = rows.filter(r => r[15] === eventoAtivo); // col P = índice 15
const cinemas = rows.filter(r => r[0]  === eventoAtivo);
const equipe  = rows.filter(r => r[0]  === eventoAtivo);
```

---

## 6. Escrita de dados — Apps Script proxy

### 6.1 Estrutura do script

```javascript
// Publicado como Web App:
// Executar como: conta admin
// Acesso: qualquer pessoa com conta Google

function doPost(e) {
  const body = JSON.parse(e.postData.contents);
  const { token, aba, operacao, dados, eventoAlvo } = body;

  const email = verificarToken(token);
  if (!email) return erro(401, 'Token inválido');

  const perfil = buscarPerfil(email);
  if (!perfil) return erro(403, 'Acesso não autorizado');

  if (perfil.papel !== 'admin' && perfil.evento !== eventoAlvo)
    return erro(403, 'Evento fora do escopo autorizado');

  if (!camposPermitidos(perfil.papel, aba, dados))
    return erro(403, 'Campo fora do escopo do papel');

  return executar(aba, operacao, dados);
}
```

### 6.2 Operações suportadas

| operacao | Descrição |
|----------|-----------|
| `UPDATE_CELL` | Atualiza célula por chave primária |
| `INSERT_ROW` | Insere nova linha |
| `DELETE_ROW` | Deleta linha por chave primária (admin only para filmes/sessões/cinemas) |

### 6.3 Payload padrão

```javascript
{
  token: "eyJ...",
  aba: "filmes",              // filmes · sessoes · cinemas · equipe
  operacao: "UPDATE_CELL",
  eventoAlvo: "cc2026",
  dados: {
    chave: "PB08",            // código do filme, sessão, nome da sala etc.
    campo: "M",               // coluna da planilha
    valor: "DCP OK"
  }
}
```

---

## 7. Datas e horas — garantia de formato

### 7.1 Estratégia de escrita

O Apps Script usa `valueInputOption: "USER_ENTERED"`. As colunas já têm formato definido na planilha — o Sheets interpreta a string no locale pt-BR.

| Campo | Formato de escrita | Exemplo |
|-------|--------------------|---------|
| dia (sessões C) | `"dd/mm/yyyy"` | `"03/10/2026"` |
| hora (sessões D) | `"HH:mm"` | `"19:30"` |
| duração DCP (filmes T) | `"H:mm"` | `"1:42"` |
| datas de uso (cinemas H) | texto serializado | `"03/10–05/10, 08/10"` |

### 7.2 Leitura

Sempre com `valueRenderOption=FORMATTED_VALUE`. O parser atual do OLGA Suite é reaproveitado sem modificações.

---

## 8. Módulos do frontend

### 8.1 Rotas (hash-based SPA)

| Hash | Módulo | Papéis com acesso |
|------|--------|-------------------|
| `#dashboard` | Dashboard do evento | todos |
| `#acervo` | Filmes — busca e edição | todos (escrita: projecao, programacao, producao) |
| `#sessoes` | Sessões — grade + formulário | programacao, producao, admin |
| `#cinemas` | Cinemas — lista + formulário | programacao, producao, admin |
| `#equipe` | Equipe — lista + formulário | producao, admin |
| `#grade` | Grade visual (leitura) | todos |
| `#ficha` | Ficha composta | todos |
| `#config` | Gestão de usuários | admin |

---

### 8.2 Dashboard (`#dashboard`)

#### Buckets de status — leitura combinada cols M × N

| Bucket | Condição | Cor |
|--------|----------|-----|
| ✓ OK | M = "DCP OK" e N = "OK" | verde |
| ⚠ Pendência de legenda | M = "DCP OK" e N = "PRECISA DE ELETRÔNICA" | amarelo |
| ✗ Erro | M = "ERRO" | vermelho |
| ↓ Aguardando | M = "LINK PARA DOWNLOAD" ou M vazio | cinza azulado |

#### Alertas adicionais

- KDM esperado sem confirmação: col L = "esperado"
- Filmes na programação sem registro em `filmes` (código órfão)
- Sessão agendada fora do período de uso da sala (cruzamento com col H de `cinemas`)

#### Filtros

- Por mostra (col G de `filmes`)
- Por cinema (via `sessões`)
- Por complexo (via `cinemas` col C)

---

### 8.3 Acervo (`#acervo`)

Busca e edição de filmes. Herda a lógica de busca atual (título, código, CPL, idioma, obs). Adiciona:

- Campos editáveis inline conforme papel do usuário (campos fora do escopo aparecem desabilitados)
- Formulário de inserção de novo filme
- Ingestão via pasta DCP (File System Access API)

---

### 8.4 Sessões (`#sessoes`)

Acesso duplo aos registros:

**Grade de programação** (modo edição)
- Mesma visualização da grade atual
- Cada bloco é clicável e abre o formulário pré-preenchido
- Indicador visual diferencia modo leitura do modo edição

**Lista filtrada**
- Filtrável por dia, cinema e mostra
- Cada linha tem botão de edição
- Botão "+ Nova sessão" fixo no topo

#### Formulário de sessão

```
Código *    [CC26MIS01  ]  ← sugerido, editável, 10 chars max, caixa alta
Nome        [           ]
Cinema *    [▼          ]  ← lista dos cinemas do evento ativo
Dia *       [📅         ]
Hora *      [⏰         ]
Mostra      [▼          ]  ← gerado dos valores únicos da col G do evento
Evento      [cc2026     ]  ← readonly, automático da sessão

Filmes
  1. [      ]  ← autocomplete por código ou título (3 chars)
  2. [      ]
  ...          ← [+ adicionar] até 10 · [× remover]

[Cancelar]  [Salvar]
```

---

### 8.5 Cinemas (`#cinemas`)

Lista de salas do evento com botão de edição por linha e "+ Nova sala".

#### Formulário de cinema

```
Nome da sala *  [              ]  ← único campo obrigatório
Complexo        [              ]
Servidor        [              ]
Projetor        [              ]
Serial          [              ]
Contatos        [              ]  ← textarea livre
Cineassist      [ ] sim
Horário         [              ]  ← ex: "10h–23h"
Capacidade      [              ]
Acessíveis      [              ]

Datas de uso
  [03/10–05/10 ×]  [08/10 ×]  [+ adicionar período]

Monitores
  Nome [          ]  Contato [          ]  ← autocomplete contra equipe
  Nome [          ]  Contato [          ]
  [+ adicionar monitor]

[Cancelar]  [Salvar]
```

---

### 8.6 Equipe (`#equipe`)

Lista editável. Cada linha: nome, função, telefone, email. Edição inline, nova entrada via formulário simples.

Os nomes alimentam o autocomplete de monitores em `cinemas`. Ao selecionar uma sugestão, o campo de contato é preenchido automaticamente com o telefone registrado. O usuário pode ignorar e preencher livremente.

---

### 8.7 Grade (`#grade`)

Três modos de visualização:

| Modo | Descrição |
|------|-----------|
| Grid | Visualização atual — sala × horário, por dia |
| Por cinema | Todas as sessões de uma sala, todos os dias |
| Tabela | Lista ordenada por data/hora, imprimível |

---

### 8.8 Ficha (`#ficha`)

#### Filtros em cascata

```
[ Tipo de ficha ▼ ]  →  [ filtros dependentes ]  →  [ Gerar ]

Sessão única     →  [ sessão ▼ ]
Dia × Sala       →  [ dia ▼ ] [ sala ▼ ]
Dia × Complexo   →  [ dia ▼ ] [ complexo ▼ ]
Dia inteiro      →  [ dia ▼ ]
Evento inteiro   →  (sem filtro extra)
```

"Dia × Complexo" gera ficha com todas as salas do complexo no dia selecionado, ordenadas por sala e horário. Inclui opcionalmente os monitores e contatos de cada sala.

#### Impressão

Cada sessão tem `page-break-inside: avoid`. Para fichas compostas (dia, complexo, evento), a quebra de página ocorre entre sessões automaticamente. O usuário vê prévia na tela e imprime quando quiser.

---

### 8.9 Config (`#config`)

Visível apenas para admin. Exibe e edita a aba `config` — lista de usuários com papel e evento. Permite adicionar e remover autorizações.

---

## 9. Ingestão via pasta DCP

Usa a **File System Access API** (`showDirectoryPicker()`). Disponível em Chrome/Edge. Fallback para Safari/Firefox: upload de arquivo individual.

#### Campos extraídos do CPL XML

| Campo XML | Campo planilha | Col |
|-----------|----------------|-----|
| `Id` (UUID do CPL) | CPL | P |
| `EditRate` + `Duration` | duração DCP | T |
| `ScreenAspectRatio` | resolução DCP | Q |
| `ContentTitleText` | — (comparação com título existente) | — |

#### Fluxo

```
1. Usuário abre pasta DCP via botão
2. Site percorre arquivos procurando *_cpl.xml
3. Extrai campos, apresenta formulário de confirmação
4. Usuário revisa e confirma
5. Apps Script escreve na planilha
```

---

## 10. Fases de desenvolvimento

### Fase 1 — Leitura online
- Setup Google Cloud (OAuth + Sheets API)
- Login Google + filtragem por evento
- Módulos ficha, grade e busca funcionando com dados online
- Sem escrita

### Fase 2 — Dashboard
- Buckets de status (M × N)
- Alertas de KDM, filmes órfãos, sessões fora do período de sala
- Filtros por mostra, cinema e complexo

### Fase 3 — Ficha composta
- Filtros em cascata
- Modo Dia × Complexo com monitores opcionais
- CSS de impressão multipágina

### Fase 4 — Escrita
- Apps Script (setup + deploy)
- Edição inline de filmes no Acervo
- Formulário de sessões (criação e edição)
- Formulário de cinemas com chips de datas e monitores com autocomplete
- Formulário de equipe
- Ingestão via pasta DCP

### Fase 5 — Grade expandida e Config
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

**Variáveis de configuração** (no topo do `index.html`):

```javascript
const CONFIG = {
  SPREADSHEET_ID:   "1abc...xyz",
  GOOGLE_CLIENT_ID: "123456-abc.apps.googleusercontent.com",
  APPS_SCRIPT_URL:  "https://script.google.com/macros/s/.../exec"
};
```

Estas três constantes são os únicos valores que mudam entre ambientes. Nenhuma delas é um segredo — a segurança da planilha é garantida pelas permissões do Google, não pela obscuridade das URLs.

---

*Versão 1.1 — inclui papéis producao e programacao, aba equipe, aba cinemas com datas serializadas e monitores com autocomplete, aba sessões com cols Q e R, modelo de ficha por complexo e matriz de permissões atualizada.*
