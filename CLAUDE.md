# g-lab — memória do projeto

> Arquivo de memória do projeto. Atualize conforme o projeto avança.

## Visão geral
- **Nome:** g-lab
- **Repositório:** https://github.com/jeovacnp2012/glab.git (branch `main`)
- **Pasta local:** D:\projetos\g_lab
- **Status:** projeto ainda não iniciado.
- **Objetivo:** _a definir_

## Identidade visual
- Marca: **UYSI G-Lab** = símbolo UYSI + ícone G-Lab (microscópio do legado) lado a lado. Ativos em `assets/brand/`.
- Cores/tema: família UYSI (`assets/brand/tokens-cores.css`, mesma do projeto UYSI Service). Só variáveis de cor, nunca cor fixa; tema claro/escuro por `data-tema`.
- `laboratorio/images/logo.png` é logo de cliente (SSPM): não usar como marca.

## Legado e dados sensíveis
- O legado PHP/MySQL está em `laboratorio/` (analisado em `docs/PLANO_DE_TRABALHO.md`). Contém dados reais de pacientes e credenciais: **não commitar** `laboratorio/`, `*.sql`, `*.pem`, `.env`.

## Stack
- Alvo: Go (API + gateway de instrumentos), PostgreSQL, agente C# (serial), LGPD. Ver plano.
_a definir_

## Convenções
- Toda comunicação e texto de UI em português do Brasil (pt_BR).
- Nomes de variáveis, classes e código interno seguem a convenção técnica normal (inglês).

## Decisões
_nenhuma ainda_

## Pendências
- [ ] Definir objetivo e escopo
- [ ] Definir stack
