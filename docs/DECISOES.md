# g-lab — Registro de decisões

## D-001 — O g-lab vira um módulo (painel) do UYSI Service (2026-11-23)

**Decisão.** O LIS legado de laboratório (PHP/MySQL, pasta local `laboratorio/`) será absorvido pelo **UYSI Service** como mais um módulo/painel, em **Go + PostgreSQL**, com marca **UYSI G-Lab** e cores da família UYSI. Este repositório (`g_lab`) fica como **registro de análise, decisões e plano**; o código do módulo será construído no repositório do UYSI Service.

**Escopo do módulo.**
- API robusta em Go e banco PostgreSQL, integrado ao painel do UYSI Service e ao banco de projetos do usuário.
- Conformidade com a LGPD em todo o ciclo (dado de saúde é sensível, art. 11).
- Análises clínicas com comunicação com analisadores:
  - **Por rede (ASTM / HL7-MLLP):** gateway em Go. Equipamentos citados: C4000 e Yumizen/Horiba Pentra 60 (modelos exatos a confirmar).
  - **Por serial (RS-232):** agente em **C#** (projeto final), rodando na máquina onde o equipamento está conectado, com fila local e envio seguro à API.

**Marca.** Símbolo UYSI + ícone do G-Lab (microscópio do legado) lado a lado. O `logo.png` do legado é de um cliente (SSPM) e não é a marca do produto. Ativos em `assets/brand/`.

## D-002 — Para onde foi e o que será feito lá

**Enviado em 2026-11-23** à sessão **uysi_go_service** (pasta `D:\projetos\UYSI Service`):
- o plano (`docs/PLANO_DE_TRABALHO.md`), com os caminhos locais do legado e dos ativos de marca;
- o resumo do legado, a arquitetura proposta e as fases;
- o aviso de que o dump do legado tem dados reais (importar só o esquema, nunca os `INSERT`; usar dados sintéticos);
- **12 perguntas de encaixe** no UYSI Service.

**O que a sessão uysi_go_service deve fazer:**
1. Ler o plano e validar o encaixe no UYSI Service: multi-tenant (um banco por empresa + central), permissões (catálogo módulo × Ver/Criar/Editar/Excluir), auditoria/lixeira, deploy e hosts.
2. Devolver um **plano revisado por passos**, no estilo da `MEMORIA.md` daquele projeto.
3. Levar ao usuário as decisões de negócio (abaixo), sem decidir por ele.

**Perguntas de encaixe enviadas (resumo):**
1. Tabelas do módulo no banco da empresa (prefixo/esquema `lab`)? Uma empresa tem um ou vários laboratórios (CNES/CNPJ)?
2. Como as portas dos analisadores chegam ao servidor (VPN, túnel, gateway local) e como o gateway identifica a empresa?
3. Autenticação do agente C#: reaproveitar os tokens de integração do UYSI Service ou usar mTLS?
4. Mapear as 25 permissões do perfil legado para o catálogo do UYSI Service; criar módulo "Laboratório" com "liberar/assinar laudo" e "gerenciar equipamentos".
5. Reaproveitar o cadastro de pessoas/pacientes existente? Base legal e consentimento entre módulos.
6. A auditoria e a lixeira atuais cobrem log de leitura de prontuário e "antes/depois"?
7. Onde guardar certificado digital e segredos (hoje ficam no banco, em texto, no legado).
8. Assinatura do laudo: ICP-Brasil A1 por laboratório + TOTP, ou outro modelo.
9. Reaproveitar o painel de chamada existente?
10. Migração: histórico completo ou últimos N anos; ensaio com dados mascarados.
11. CADSUS e SIGTAP entram na primeira entrega?
12. Reaproveitar padrões de tela (lista, shell de menu, tema claro/escuro).

## D-003 — Decisões de negócio que dependem do usuário (em aberto)

- Modelo/fabricante exato do **C4000**, versões e manuais de interface.
- O **Pentra 60** em uso tem Ethernet ou só RS-232?
- Hospedagem (nuvem, servidor próprio ou híbrido).
- **Encarregado de dados (DPO)** e base legal para o uso analítico no banco de projetos.
- Escopo SUS (CADSUS, SIGTAP, faturamento).
- Modelo de assinatura digital do laudo.

## D-004 — Regra de versionamento e dados (LGPD)

Este repositório **não versiona** o legado nem dados de pacientes: `laboratorio/`, dumps SQL, certificados/chaves, `.env`, capturas brutas de equipamentos e uploads estão no `.gitignore`. O legado fica só na máquina local. Nenhuma credencial ou dado pessoal entra em commits, mensagens ou logs; desenvolvimento e testes usam dados sintéticos.
