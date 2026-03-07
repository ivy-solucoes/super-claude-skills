# 🧠 Claude Skills — Ivy Soluções

Repositório central de **Agent Skills** para uso com Claude (Claude.ai, Claude Code e API).
Cada skill encapsula o conhecimento e as instruções necessárias para o Claude atuar como especialista em uma das nossas ferramentas e soluções internas.

---

## 📁 Estrutura do Repositório

```
meu-repo-de-skills/
└── skills/
    ├── frappe/          ← Desenvolvimento com Frappe Framework
    │   └── SKILL.md
    ├── erp/             ← Customizações e desenvolvimento ERP
    │   └── SKILL.md
    └── outra-skill/     ← [Descrição da skill]
        └── SKILL.md
```

Cada skill é uma pasta com um arquivo `SKILL.md` contendo:
- **Frontmatter YAML** — nome e descrição (usados pelo Claude para decidir quando ativar a skill)
- **Instruções em Markdown** — o conhecimento e os padrões que o Claude vai seguir

---

## 🚀 Como Instalar

### Claude Code (CLI)

Clone o repositório direto na pasta de skills do Claude Code:

```bash
# Instalar todas as skills de uma vez
git clone https://github.com/sua-empresa/claude-skills ~/.claude/skills/repo-empresa

# Ou copiar uma skill específica
cp -r ./skills/frappe ~/.claude/skills/frappe
```

Após clonar, as skills são detectadas automaticamente. Nenhum restart necessário.

### Claude.ai (Interface Web)

1. Acesse **Settings → Skills**
2. Faça upload da pasta da skill desejada (ex: `skills/frappe/`)
3. A skill estará disponível na próxima conversa

---

## 📦 Skills Disponíveis

| Skill | Descrição | Quando usar |
|---|---|---|
| [`frappe`](./skills/frappe/) | Desenvolvimento com Frappe Framework | DocTypes, Controllers, Hooks, Form Scripts, bench CLI |
| [`erp`](./skills/erp/) | Customizações ERP | Módulos de ERP, customizações, relatórios, workflows |

> Novas skills são adicionadas conforme novas soluções são incorporadas ao stack.

---

## ✍️ Como Criar uma Nova Skill

1. Crie uma pasta em `skills/nome-da-skill/`
2. Crie o arquivo `SKILL.md` com o seguinte template:

```markdown
---
name: nome-da-skill
description: >
  Descreva o que a skill faz e quando o Claude deve ativá-la automaticamente.
  Seja específico: mencione palavras-chave, contextos e casos de uso.
---

# Nome da Skill

Visão geral do que esta skill ensina ao Claude.

## Conceitos Principais
...

## Exemplos de Uso
...
```

3. Abra um Pull Request com a nova skill
4. Após aprovação e merge, a skill estará disponível para o time

### Boas práticas

- A `description` no frontmatter é o principal gatilho de ativação — escreva de forma clara e inclua termos que o usuário provavelmente vai digitar
- Mantenha o `SKILL.md` abaixo de 500 linhas; use arquivos de referência em `references/` para conteúdo mais extenso
- Inclua exemplos de código reais do nosso contexto, não genéricos

---

## 🔄 Mantendo as Skills Atualizadas

```bash
# Atualizar todas as skills instaladas
cd ~/.claude/skills/repo-empresa && git pull
```

Recomendamos configurar um lembrete para revisar as skills a cada nova versão das ferramentas.

---

## 📄 Licença

Uso interno — © Ivy Soluções. Todos os direitos reservados.
