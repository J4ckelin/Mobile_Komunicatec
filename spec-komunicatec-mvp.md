# Komunicatec — SPEC do MVP (SPA estática para GitHub Pages)

> Documento único para Spec-Driven Development. Decisões arquiteturais já estão tomadas;
> não reintroduzir funcionalidades marcadas como **FORA DO MVP** (seção 9).

---

## 1. Contexto e objetivo

SPA em **HTML + CSS + JavaScript puro** (sem frameworks, sem pacotes, sem build, sem backend),
hospedada no **GitHub Pages**, para centralizar a comunicação acadêmica da Fatec Franco da Rocha:
comunicados oficiais, eventos e vagas de estágio/emprego.

**Modo demonstração:** por ser 100% estático, não há servidor nem banco compartilhado.
Todos os dados de usuários, solicitações e publicações criadas no app vivem **apenas no
localStorage do navegador**. Publicações oficiais vêm de um arquivo JSON seed embutido no
repositório. Isso é uma limitação conhecida e declarada, não um bug.

---

## 2. Stack e restrições técnicas

- HTML5 semântico, CSS3, JavaScript (ES6+, vanilla). Proibido: npm, bundlers, frameworks, CDNs externos.
- Roteamento por **hash** (`#/feed`, `#/login`, `#/cadastro`, `#/publicacao/:id`,
  `#/esqueci-senha`, `#/painel`), para funcionar no GitHub Pages sem configuração de servidor.
- Dados em **localStorage**, com chaves prefixadas `komunicatec:`.
- Seed de publicações embutido como **módulo JS** (`js/data/seed.js`), não via `fetch` de JSON
  (evita problemas de CORS/`file://` e simplifica o deploy).
- Todas as rotas de arquivos relativas (funciona em subdiretório de `*.github.io`).
- Persistir sessão: ao recarregar, usuário logado permanece logado.

### Estrutura sugerida de arquivos

```
/
├── index.html
├── css/styles.css
└── js/
    ├── app.js            # bootstrap + roteador hash
    ├── data/seed.js      # publicações oficiais + usuários seed (export const)
    ├── services/
    │   ├── storage.js    # leitura/escrita localStorage com prefixo
    │   ├── auth.js       # login, cadastro, sessão, recuperação de senha
    │   └── posts.js      # feed: merge seed + locais, expiração, busca, filtros
    └── ui/               # renderização por tela (uma função por rota)
```

---

## 3. Decisões-chave (não reabrir)

| # | Decisão |
|---|---|
| D1 | **Sem backend.** Aprovação de comunicadores, publicações e usuários existem apenas no navegador local (modo demonstração). |
| D2 | **Geolocalização registra, não bloqueia.** Se o usuário negar a permissão, o cadastro/solicitação **prossegue normalmente**, gravando `localizacao: null`, e exibe aviso "vínculo com o campus não confirmado — procure a coordenação". Nunca impedir o fluxo por causa de geolocalização. |
| D3 | **Sem CPF.** Cadastro pede apenas: nome, e-mail institucional, curso, senha. |
| D4 | **Sem biometria / CredentialsContainer.** Autenticação é login simulado por e-mail + senha. |
| D5 | **Sem edição de publicações.** Comunicador/Admin podem apenas criar e excluir. |
| D6 | **E-mail de verificação 100% simulado:** o código (6 dígitos) aparece em um painel "Simulação de e-mail" na própria tela. |
| D7 | **Sem sincronização entre dispositivos** — limitação declarada do modo demonstração. |
| D8 | Segurança é simulada: senhas e tokens ficam em localStorage em texto simples. A SPEC declara isso explicitamente; não implementar hash/criptografia. |

---

## 4. Modelo de dados

### 4.1 Publicação (objeto único para seed e locais)

```js
{
  id: "pub-001",              // string única
  titulo: "...",              // string, obrigatório
  descricao: "...",           // string (texto longo), obrigatório
  categoria: "Comunicado",    // "Comunicado" | "Evento" | "Vaga"
  urgente: false,             // boolean; se true, exibe selo vermelho
  autor: "Coordenação",       // string (nome do perfil publicador)
  origem: "seed",             // "seed" | "local"
  dataPublicacao: "2026-10-15", // ISO date
  dataExpiracao: null         // ISO date ou null
}
```

**Expiração (RF11/RN04):** se `dataExpiracao` não for nula e for anterior à data atual,
a publicação **não aparece no feed** de discentes, mas permanece armazenada e é listada
no painel admin/comunicador com o selo "Expirada".

### 4.2 Usuário

```js
{
  id: "usr-001",
  nome: "Maria Silva",
  email: "maria@fatec.sp.gov.br",   // obrigatório terminar em @fatec.sp.gov.br
  curso: "Análise e Desenvolvimento de Sistemas",
  senha: "demo123",                 // texto simples — modo demonstração (D8)
  perfil: "discente",               // "discente" | "egresso" | "comunicador" | "admin"
  status: "ativo",                  // "pendente" | "ativo" | "recusado" (só para comunicador)
  localizacao: { lat: -23.32, lng: -46.72 } | null,  // D2
  criadoEm: "2026-10-20"
}
```

### 4.3 Solicitação de comunicador

Usa o mesmo objeto de usuário com `perfil: "comunicador"` e `status: "pendente"`,
mais campo `justificativa: "..."`. Aprovação = mudar `status` para `"ativo"` (só admin, RN03).

### 4.4 Chaves do localStorage (prefixo `komunicatec:`)

| Chave | Conteúdo |
|---|---|
| `session` | `{ token, email, perfil }` do usuário logado, ou ausente |
| `users` | array de usuários cadastrados neste navegador (inclui comunicadores pendentes) |
| `posts` | array de publicações criadas neste navegador (`origem: "local"`) |
| `favorites` | array de ids de publicações favoritadas |
| `filters` | última busca/categoria selecionada no feed |

---

## 5. Usuários seed (modo demonstração)

| E-mail | Senha | Perfil | Status |
|---|---|---|---|
| admin@fatec.sp.gov.br | admin123 | admin | ativo |
| comunicador@fatec.sp.gov.br | com123 | comunicador | ativo |
| aluno@fatec.sp.gov.br | aluno123 | discente | ativo |

- Login deve aceitar tanto os seed quanto os cadastrados neste navegador (buscar em ambos).
- Cursos disponíveis no cadastro (select): "Análise e Desenvolvimento de Sistemas",
  "Gestão Empresarial", "Logística", "Outro".

---

## 6. Requisitos funcionais do MVP

### RF01 — Cadastro de Discente/Egresso
Formulário **de página única** (sem wizard): nome, e-mail institucional, curso (select),
perfil (select: discente/egresso), senha.
**Critérios de aceite:**
- [ ] E-mail fora do domínio `@fatec.sp.gov.br` é rejeitado com mensagem inline.
- [ ] E-mail já cadastrado (local ou seed) é rejeitado.
- [ ] Ao submeter, solicita geolocalização: se concedida, grava coords; se negada, grava `null` e exibe aviso (D2) — **em ambos os casos o cadastro conclui**.
- [ ] Após cadastro, exige verificação simulada de e-mail (RF-código) antes do primeiro acesso.
- [ ] Campos preservam os dados preenchidos quando há erro de validação.

### RF02 — Solicitação de perfil Comunicador
Mesmo formulário de RF01, mais campo "justificativa" e perfil fixo `comunicador`.
**Critérios de aceite:**
- [ ] Solicitante fica com `status: "pendente"` e **não consegue publicar** até aprovação.
- [ ] Solicitação pendente aparece no painel do admin logado neste navegador (RN02, modo demo).

### RF03 — Aprovar/recusar Comunicador (somente admin, RN02/RN03)
No painel admin, lista de solicitações pendentes com botões Aprovar / Recusar.
**Critérios de aceite:**
- [ ] Aprovar muda `status` para `"ativo"` e o usuário passa a logar como comunicador.
- [ ] Recusar muda `status` para `"recusado"` e o login é bloqueado com mensagem.

### RF05 — Login
E-mail + senha, validando contra seed + usuários locais.
**Critérios de aceite:**
- [ ] Credenciais seed funcionam sem nenhum cadastro prévio.
- [ ] Usuário pendente/recusado recebe mensagem específica, sem entrar.
- [ ] Sucesso cria `session` em localStorage e redireciona ao feed.
- [ ] Recarregar a página mantém a sessão; botão "Sair" limpa `session`.

### RF10 — Recuperação de senha (simulada)
Informar e-mail → gerar código de 6 dígitos → exibir no painel "Simulação de e-mail"
(D6) → usuário digita o código → define nova senha.
**Critérios de aceite:**
- [ ] Código incorreto exibe erro e preserva o formulário.
- [ ] Nova senha passa a valer no próximo login.
- [ ] E-mail inexistente exibe mensagem genérica (não revelar existência da conta).

### RF07 — Publicar (somente comunicador ativo ou admin)
Painel com formulário: título, descrição, categoria, urgente (checkbox), data de expiração (opcional).
**Critérios de aceite:**
- [ ] Publicação criada aparece no topo do feed imediatamente (`origem: "local"`).
- [ ] Comunicador/Admin podem excluir suas publicações locais (com confirmação).
- [ ] Discente/visitante não acessa o painel (tela de login ou "acesso negado").

### RF11 — Expiração de publicação
**Critérios de aceite:**
- [ ] Publicação com `dataExpiracao` vencida some do feed de discentes, sem ser apagada.
- [ ] Publicação expirada aparece no painel com selo "Expirada".

### Feed (RF08 + RF09) — tela principal `#/feed`
Lista todas as publicações não expiradas (seed + locais), ordenadas por `dataPublicacao` desc.
**Critérios de aceite:**
- [ ] Publicações `urgente: true` exibem selo vermelho e vêm primeiro, com destaque visual.
- [ ] Busca por texto filtra por correspondência em título e descrição (case-insensitive).
- [ ] Filtro por categoria (Comunicado / Evento / Vaga / Todas) combina com a busca.
- [ ] Última busca/categoria ficam salvas em `filters` e são restauradas ao recarregar.
- [ ] Estado vazio: "Nenhuma publicação encontrada".
- [ ] Card abre tela de detalhe `#/publicacao/:id` com conteúdo completo.
- [ ] Feed acessível sem login (visitante); favoritar exige login.

### Favoritos
**Critérios de aceite:**
- [ ] Usuário logado pode favoritar/desfavoritar no card ou detalhe; estado persiste em localStorage.
- [ ] Filtro "Salvos" exibe apenas favoritos (de não expiradas).

### Painel admin (modo demo)
**Critérios de aceite:**
- [ ] Lista usuários cadastrados neste navegador (RN05 adaptada: apenas o que existe localmente — D1).
- [ ] Lista solicitações pendentes (RF03).
- [ ] Lista publicações expiradas (RF11).

---

## 7. Regras de negócio do MVP

- **RN01:** Funcionalidades restritas (publicar, painel, favoritar) exigem login com perfil adequado.
- **RN02:** Comunicador só publica com `status: "ativo"` (aprovado por admin — local/demo).
- **RN03:** Só admin aprova solicitações. (Elevar comunicador a admin está FORA DO MVP.)
- **RN04:** Expiradas são ocultadas do feed e preservadas (RF11).
- **RN05:** Lista de usuários restrita a perfis comunicador/admin (no modo demo, lista apenas usuários locais).
- **RN06 (nova):** E-mail institucional obrigatório: domínio `@fatec.sp.gov.br`.

---

## 8. Requisitos não funcionais aplicáveis

- **RNF01** Compatibilidade: Chrome, Firefox, Edge, Safari (versões dos últimos 2 anos), mobile e desktop.
- **RNF02** Design: minimalista; paleta `#1F496B` (primária), `#D9D9D9` (neutros), `#FFFFFF` (fundo),
  `#E21518` (somente urgência/erros); fonte **Poppins** (fallback system-ui — sem CDN externo;
  declarar via `font-family` apenas, não carregar a fonte).
- **RNF04** Facilidade: navegação por no máximo 2 toques entre feed e qualquer tela principal.
- **RNF05** Responsividade: mobile-first; uso confortável em 360px e em 1280px.
- **RNF07** Tratamento de entrada: todo texto vindo de usuário é escapado antes de entrar no DOM
  (função utilitária `escapeHTML`), sem exceções.
- **RNF09** Acessibilidade básica: `lang="pt-BR"`, label em todo input, contraste mínimo AA,
  navegação por teclado nos botões principais.
- Sanitização de dados locais: ao ler do localStorage, ignorar registros malformados sem quebrar o app.

---

## 9. FORA DO MVP (não implementar, não reintroduzir)

- Backend/API, banco de dados, qualquer chamada de rede.
- Biometria / WebAuthn / CredentialsContainer.
- CPF.
- Edição de publicações (criar e excluir apenas).
- Elevar comunicador a admin (RF04 da spec original).
- Envio real de e-mail/SMS/push.
- Histórico escolar, notas, faltas, matrícula (nunca — o app não substitui o SIGA).
- Inscrição/candidatura a vagas, ou qualquer gestão de processo seletivo (mural de divulgação apenas).
- Sincronização entre dispositivos, PWA/offline, notificações.
- RNF03 (24/7), RNF06 (5s) e RNF08 (escalabilidade) da spec original — não aplicáveis a estático.

---

## 10. Definition of Done

- [ ] Todos os critérios de aceite da seção 6 passando.
- [ ] App funciona abrindo `index.html` via GitHub Pages sem erros de console.
- [ ] Layout utilizável em 360px e 1280px sem scroll horizontal.
- [ ] Nenhum dado de usuário inserido sem passar por `escapeHTML`.
- [ ] Fluxos completos testáveis com os 3 usuários seed.
