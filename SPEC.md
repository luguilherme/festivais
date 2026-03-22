# OLGA Web — Especificação Técnica
**Versão 2.0**

---

## 1. Visão geral

Sistema web de gestão de exibição para festivais de cinema. Evolui o OLGA Suite (leitor local de XLSX) para uma plataforma online colaborativa com leitura e escrita em tempo real, controle de acesso por papel e evento, e exportação/importação XLSX para interoperabilidade com o fluxo de trabalho offline existente.

**Stack:**
- Frontend: HTML + CSS + JavaScript puro (sem framework)
- Hospedagem: GitHub Pages — repositório `festivais`, branch `main`
- Dados: arquivos JSON no próprio repositório (`/data/*.json`)
- Auth: Personal Access Token (PAT) do GitHub, por usuário
- Escrita: GitHub REST API (`PUT /repos/.../contents/data/*.json`)
- URL de produção: `https://SEU-USUARIO.github.io/festivais`

**Princípios de design:**
- Zero dependência de serviços externos além do GitHub
- JSON como formato de dado primário — legível, versionável, editável direto no browser
- XLSX como formato de interoperabilidade — import/export a qualquer momento
- Dado versionado: cada escrita gera um commit com histórico completo

---

## 2. Estrutura do repositório

```
festivais/
├── index.html          ← aplicação completa (SPA)
├── README.md
├── SPEC.md             ← este documento
└── data/
    ├── filmes.json
    ├── sessoes.json
    ├── cinemas.json
    ├── equipe.json
    └── config.json
```

Os arquivos JSON são arrays de objetos. Cada objeto é equivalente a uma linha da planilha original. O site lê via `fetch` público (GET sem auth) e escreve via GitHub API autenticada (PUT com PAT).

---

## 3. Estrutura dos dados

### 3.1 `filmes.json`

Cada objeto é um filme. Espelha as colunas da planilha XLSX original.

| Campo | Coluna XLSX | Tipo | Papel que edita |
|-------|-------------|------|-----------------|
| evento | A | string | admin |
| codigo | B | string | admin / programacao |
| hd | C | string | admin |
| titulo | D | string | programacao / producao |
| idioma | E | string | programacao / producao |
| legDCP | F | string | projecao |
| mostra | G | string | programacao / producao |
| vf | H | string | programacao / producao |
| lente | I | string | projecao |
| somDCP | J | string | projecao |
| creator | K | string | projecao |
| kdm | L | enum | programacao/producao (esperado) · projecao (sim, não) |
| status | M | enum | projecao |
| statusLeg | N | enum | projecao · programacao/producao |
| obs | O | string | projecao |
| cpl | P | string | projecao |
| resolucao | Q | string | projecao |
| duracao | T | string HH:mm | projecao |
| dcpsize | V | number | projecao |
| fps | X | number | projecao |
| cplOV | Y | string | projecao |
| linkDCP | Z | string | admin · programacao/producao |
| legTrans | AB | string | programacao/producao |

**Valores controlados:**

`status`: `DCP OK` · `DCP LG` · `ERRO` · `LINK PARA DOWNLOAD` · `AGUARDANDO` · `CANCELADO` · `CAIU` · *(vazio)*

`statusLeg`: `OK` · `PRECISA DE ELETRÔNICA` · *(vazio)*

`kdm`: `esperado` · `sim` · `não` · *(vazio)*

**Lógica de bucket (dashboard):**

| Condição de status | Condição de legenda | Bucket |
|--------------------|---------------------|--------|
| `DCP OK` ou `DCP LG` | `OK` ou vazio | ✓ OK |
| `DCP OK` ou `DCP LG` | qualquer outro valor | ⚠ Pendência |
| vazio · `AGUARDANDO` · contém `DOWNLOAD` | — | ↓ Aguardando |
| `CANCELADO` ou `CAIU` | — | ✕ Cancelados |
| qualquer outro valor | — | ✗ Erro |

---

### 3.2 `sessoes.json`

Cada objeto é uma sessão.

| Campo | Tipo | Observação |
|-------|------|------------|
| codigo | string (10 chars, caixa alta) | chave primária |
| nome | string | nome descritivo da sessão |
| cinema | string | referência ao campo `nome` em cinemas.json |
| dia | string dd/mm/yyyy | |
| hora | string HH:mm | |
| filmes | array de strings | até 10 códigos, referência ao campo `codigo` em filmes.json |
| evento | string | chave de segurança |
| mostra | string | |

**Geração do código:** o site sugere `[PREFIXO_EVENTO][SIGLA_CINEMA][SEQ_2_DIGITOS]`, ex: `CC26MIS01`. Limitado a 10 caracteres, sempre caixa alta, validado contra duplicatas antes de salvar.

---

### 3.3 `cinemas.json`

Cada objeto é uma sala. Uma sala pode ter entradas em múltiplos eventos.

| Campo | Tipo | Obrigatório |
|-------|------|-------------|
| evento | string | sim (automático) |
| nome | string | **sim — chave primária** |
| complexo | string | não |
| servidor | string | não |
| projetor | string | não |
| serial | string | não |
| contatos | string | não — texto livre |
| datas | string serializado | não — formato: `"03/10–05/10, 08/10"` |
| horario | string | não — ex: `"10h–23h"` |
| capacidade | number | não |
| acessiveis | number | não |
| cineassist | boolean | não — sistema Dolby de acessibilidade |
| monitoresNomes | string | não — separador: ` / ` |
| monitoresContatos | string | não — separador: ` / ` |

**Datas de uso:** texto serializado com períodos separados por vírgula, cada um podendo ser data única ou intervalo com `–`. A UI apresenta como chips editáveis.

**Admin pode duplicar sala para outro evento** — formulário abre pré-preenchido com dados da sala original, apenas campo `evento` muda para o destino.

---

### 3.4 `equipe.json`

Cada objeto é um membro da equipe do evento.

| Campo | Tipo | Obrigatório |
|-------|------|-------------|
| evento | string | sim (automático) |
| nome | string | **sim** |
| funcao | string | não |
| telefone | string | não |
| email | string | não |

Os nomes da equipe alimentam o autocomplete do campo monitores em `cinemas.json`.

---

### 3.5 `config.json`

Controle de acesso. Cada objeto é uma autorização. Uma pessoa pode ter múltiplos objetos (múltiplos eventos ou papéis).

| Campo | Exemplo |
|-------|---------|
| email | joao@gmail.com |
| papel | projecao |
| evento | cc2026 |

**Papéis válidos:**

| Papel | Descrição |
|-------|-----------|
| `admin` | Acesso total a todos os eventos. Campo `evento` ignorado — usar `*` |
| `projecao` | Edita campos técnicos de DCP no evento autorizado |
| `programacao` | Edita campos editoriais, sessões e cinemas no evento autorizado |
| `producao` | Mesmos poderes de `programacao` + gestão da equipe |

---

## 4. Autenticação — Personal Access Token

### 4.1 Configuração

Cada usuário com permissão de escrita gera um PAT no GitHub com escopo `repo`. O token é inserido no site uma única vez e armazenado em `localStorage` (criptografado com a senha do usuário, ou em texto se o usuário aceitar o risco).

O email declarado pelo usuário no primeiro acesso é cruzado com `config.json` para determinar papel e eventos autorizados.

### 4.2 Fluxo de acesso

```
1. Usuário abre o site
2. Se token salvo → valida via GET /user (GitHub API)
3. Se inválido ou ausente → tela de configuração:
   - Campo de email
   - Campo de token GitHub
4. Site lê config.json via fetch público
5. Cruza email com config → monta perfil:
   { email, papeis: [{papel, evento}] }
6. 0 autorizações → "acesso não autorizado"
7. 1 evento → entra direto
8. 2+ eventos → pills de seleção de evento(s)
9. Admin → vê todos os eventos, pode combinar
```

### 4.3 Leitura de dados

Arquivos JSON são públicos — lidos via `fetch` sem auth:

```javascript
fetch('https://SEU-USUARIO.github.io/festivais/data/filmes.json')
```

### 4.4 Escrita de dados

Via GitHub REST API com o PAT do usuário:

```javascript
// GET para obter SHA atual do arquivo (obrigatório para PUT)
GET /repos/SEU-USUARIO/festivais/contents/data/filmes.json

// PUT para sobrescrever com novo conteúdo
PUT /repos/SEU-USUARIO/festivais/contents/data/filmes.json
{
  "message": "Atualiza filmes — cc2026 [usuário]",
  "content": "<base64 do JSON atualizado>",
  "sha": "<sha obtido no GET>"
}
```

Cada escrita gera um commit com mensagem descritiva, criando histórico completo de todas as alterações.

### 4.5 Validação de permissão (client-side)

A validação de permissão é feita no frontend antes de qualquer escrita:

1. O evento sendo escrito está nos eventos autorizados do usuário?
2. O campo sendo editado está no escopo do papel do usuário?

Se qualquer verificação falhar, a escrita não é enviada e o usuário vê uma mensagem de erro.

*Nota: como não há servidor, a segurança depende de o PAT do usuário ter permissões restritas no repositório. Para maior segurança em produção, considerar branch protection e PATs com escopo limitado a conteúdo específico.*

---

## 5. Modelo de permissões

### 5.1 Matriz papel × operação

| Operação | admin | projecao | programacao | producao |
|----------|:-----:|:--------:|:-----------:|:--------:|
| Inserir filme | ✓ | ✓ | — | — |
| Deletar filme | ✓ | — | — | — |
| Editar campos de programação | ✓ | — | ✓ | ✓ |
| Editar campos de projeção | ✓ | ✓ | — | — |
| Editar kdm "esperado" | ✓ | — | ✓ | ✓ |
| Editar kdm "sim/não" | ✓ | ✓ | — | — |
| Inserir sessão | ✓ | — | ✓ | ✓ |
| Deletar sessão | ✓ | — | — | — |
| Editar sessão | ✓ | — | ✓ | ✓ |
| Inserir / editar cinema | ✓ | — | ✓ | ✓ |
| Deletar cinema | ✓ | — | — | — |
| Duplicar cinema para outro evento | ✓ | — | — | — |
| Inserir / editar / deletar equipe | ✓ | — | — | ✓ |
| Ler / editar config | ✓ | — | — | — |

### 5.2 Escopo por evento

Toda leitura e escrita é filtrada pelo(s) evento(s) ativo(s) da sessão do usuário. A filtragem usa o campo `evento` presente em todos os arquivos JSON.

**Seleção de evento:** pills clicáveis na barra abaixo do header. Admin pode combinar múltiplos eventos. Não-admin vê só os eventos autorizados e pode combinar entre eles se tiver mais de um.

---

## 6. Interoperabilidade XLSX

### 6.1 Exportar XLSX

Botão disponível em qualquer módulo. Gera um arquivo `.xlsx` com abas:
- `filmes` — todos os filmes dos eventos visíveis, colunas na ordem original
- `sessoes` — todas as sessões dos eventos visíveis
- `cinemas` — todas as salas dos eventos visíveis
- `equipe` — toda a equipe dos eventos visíveis

Formatos de data e hora preservados (`dd/mm/yyyy`, `HH:mm`). Compatível com o OLGA Suite offline.

### 6.2 Importar XLSX

Fluxo de importação pontual, disponível para admin:

```
1. Usuário carrega .xlsx
2. Site lê e compara com JSON atual
3. Exibe diff: "X filmes novos · Y modificados · Z sem alteração"
4. Usuário revisa e confirma
5. Site escreve só as diferenças nos JSONs via GitHub API
```

Permite incorporar trabalho feito offline (no XLSX local) para o banco compartilhado sem reescrever dados que já existem online.

### 6.3 Edição direta do JSON no GitHub

Para quem preferir trabalhar no dado bruto:
- Acessar `github.com/SEU-USUARIO/festivais/blob/main/data/filmes.json`
- Clicar no ícone de lápis (editar)
- Localizar o objeto pelo código, copiar, colar, inserir nova linha
- Fazer commit

Cada objeto JSON corresponde exatamente a uma linha da planilha. Operação de 30 segundos para quem já conhece o formato.

---

## 7. Módulos do frontend

### 7.1 Rotas (hash-based SPA)

| Hash | Módulo | Papéis com acesso de escrita |
|------|--------|------------------------------|
| `#dashboard` | Dashboard do evento | — (só leitura) |
| `#acervo` | Filmes — busca e edição | projecao, programacao, producao |
| `#sessoes` | Sessões — lista + formulário | programacao, producao |
| `#cinemas` | Cinemas — lista + formulário | programacao, producao |
| `#equipe` | Equipe — lista + formulário | producao |
| `#grade` | Grade visual (3 modos) | — (só leitura) |
| `#ficha` | Ficha composta | — (só leitura + impressão) |
| `#config` | Gestão de usuários | admin |
| `#import` | Import/export XLSX | admin (import) · todos (export) |

### 7.2 Dashboard

**Buckets de status** — leitura combinada `status × statusLeg` (ver seção 3.1).

**Alertas:**
- KDM "esperado" sem confirmação de projeção
- Filmes em sessões sem registro no acervo (código órfão)
- Sessão agendada fora do período de uso da sala

**Filtros:** por cinema (dropdown filtrado pelo evento ativo).

### 7.3 Grade — três modos

| Modo | Descrição |
|------|-----------|
| Grid | Visualização sala × horário por dia. Começa na hora da primeira sessão. |
| Tabela | Lista ordenada cronologicamente, imprimível. |
| Por cinema | Todas as sessões de uma sala em todos os dias. |

O painel de grade tem altura fixa (`100vh` menos os elementos acima) e rola internamente nos dois eixos. Cabeçalho de colunas sincronizado com scroll horizontal.

### 7.4 Ficha — filtros em cascata

```
[ Tipo ] → [ filtros ] → [ Gerar automático ou botão ]

Sessão única     →  [ sessão ▼ ]  ← gera automaticamente ao selecionar
Dia × Sala       →  [ dia ▼ ] [ sala ▼ ]
Dia × Complexo   →  [ dia ▼ ] [ complexo ▼ ]
Dia inteiro      →  [ dia ▼ ]
Evento inteiro   →  (sem filtro)
```

Impressão: A4 paisagem, `page-break-inside: avoid` por sessão. Inclui monitores da sala quando disponível.

### 7.5 Formulário de sessão

```
Código *    [CC26MIS01  ]  ← sugerido, 10 chars, caixa alta, validado
Nome        [           ]
Cinema *    [▼          ]  ← dropdown das salas do evento ativo
Dia *       [📅         ]
Hora *      [⏰         ]
Mostra      [▼          ]  ← valores únicos do campo mostra do evento
Evento      [cc2026     ]  ← readonly, automático

Filmes      (até 10, autocomplete por código ou título)
```

---

## 8. Fluxo de escrita — GitHub API

### 8.1 Padrão de operação

Toda escrita segue o mesmo padrão:

```javascript
// 1. Ler arquivo atual + obter SHA
const res = await fetch(
  'https://api.github.com/repos/SEU-USUARIO/festivais/contents/data/filmes.json',
  { headers: { Authorization: `token ${PAT}` } }
);
const { content, sha } = await res.json();
const dados = JSON.parse(atob(content));

// 2. Modificar o array em memória
dados.push(novoFilme); // ou splice, ou find+assign

// 3. Escrever de volta
await fetch(
  'https://api.github.com/repos/SEU-USUARIO/festivais/contents/data/filmes.json',
  {
    method: 'PUT',
    headers: { Authorization: `token ${PAT}`, 'Content-Type': 'application/json' },
    body: JSON.stringify({
      message: `Insere ${novoFilme.codigo} — ${novoFilme.evento} [${email}]`,
      content: btoa(JSON.stringify(dados, null, 2)),
      sha
    })
  }
);
```

### 8.2 Mensagens de commit

Padrão: `[operação] [identificador] — [evento] [usuário]`

Exemplos:
- `Insere PB08 — cc2026 [luiz@...]`
- `Atualiza status PB08 — cc2026 [joao@...]`
- `Insere sessão CC26MIS01 — cc2026 [ana@...]`
- `Importa 12 filmes — cc2026 [luiz@...]`

---

## 9. Fases de desenvolvimento

### Fase 1 — Leitura online
- Configuração do repositório e arquivos JSON
- Autenticação via PAT + leitura de `config.json`
- Módulos ficha, grade e acervo funcionando com dados do repositório
- Seleção de evento por pills
- Sem escrita

### Fase 2 — Dashboard
- Buckets de status
- Alertas de KDM, filmes órfãos, sessões fora do período de sala
- Filtro por cinema

### Fase 3 — Ficha composta
- Filtros em cascata completos
- Modo Dia × Complexo com monitores opcionais
- CSS de impressão multipágina A4 paisagem

### Fase 4 — Escrita
- Edição inline de filmes no Acervo
- Formulário de sessões (criação e edição)
- Formulário de cinemas com chips de datas e autocomplete de monitores
- Formulário de equipe
- Ingestão via pasta DCP (File System Access API)

### Fase 5 — Import/export e Config
- Exportação XLSX
- Importação XLSX com diff
- Módulo Config (gestão de usuários via `config.json`)
- Grade expandida — modo tabela e por cinema refinados

---

## 10. Variáveis de configuração

Três constantes no topo do `index.html`:

```javascript
const CONFIG = {
  GITHUB_USER:  "SEU-USUARIO",
  GITHUB_REPO:  "festivais",
  DATA_BRANCH:  "main"
};
```

O PAT não fica no código — é inserido pelo usuário no primeiro acesso e armazenado em `localStorage`.

---

## 11. Notas sobre segurança

O modelo PAT em frontend tem limitações conhecidas:

- O PAT fica em `localStorage` — acessível para quem tiver acesso físico ao dispositivo
- Não há validação server-side das permissões — a segurança depende da disciplina dos PATs
- Para mitigar: usar PATs com escopo mínimo (`contents` read+write, sem admin do repo)
- O histórico de commits do GitHub serve como auditoria — toda escrita tem autor e timestamp

Para o contexto de uso (equipe pequena, ambiente de festival), esse modelo é suficiente. Uma evolução futura poderia introduzir um proxy serverless (Cloudflare Worker ou Netlify Function) para validação server-side sem adicionar infraestrutura significativa.

---

*Versão 2.0 — arquitetura migrada de Google Sheets + Apps Script para GitHub JSON + REST API. Eliminação de dependências externas ao ecossistema GitHub.*
