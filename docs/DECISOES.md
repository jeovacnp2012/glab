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

## D-005 — Plano revisado de encaixe no UYSI Service (2026-11-23)

Retorno da sessão uysi_go_service, gravado no repositório do UYSI Service em `docs/21-plano-glab.md` (somente leitura aqui; nada codado, nada na VPS, nenhum dado real lido).

**Decisões de arquitetura propostas (a aprovar pelo usuário):**
1. O G-Lab é o painel `lab` do UYSI Service (ao lado de `tickets`, `procurement` e `ceaf`); código em `api/` e `web/` do UYSI Service, tabelas `lab_*` no banco da empresa (migrations goose).
2. Uma empresa por laboratório (um banco PostgreSQL por empresa); sem RLS. Vários postos/unidades = Setores.
3. Equipamentos nunca expostos à internet: agente local na rede do laboratório envia por HTTPS com token `uysi_...` (só SHA-256 no banco, revogável); mTLS opcional pelo Nginx. O "gateway Go" vira modo do agente ou listener opcional só em rede privada/VPN.
4. Reaproveitar do UYSI: login, permissões por painel, auditoria imutável, PDF, exportação, segredos cifrados (`UYSI_SECRET_KEY`), rotinas de 6 h, deploy e backup.
5. Novo: parsers ASTM E1381/E1394 e HL7 v2/MLLP, TOTP para liberar laudo, cifra de CPF/CNS/nome da mãe com blind index (HMAC), `lab_access_log` (log de leitura), retenção e limpeza do bruto cifrado.
6. Resultados e laudos versionados (correção = nova versão), sem Lixeira; paciente com exclusão lógica + anonimização.
7. Chave de ingestão: ID de amostra/código de barras gerado pelo sistema (protocolo como alternativa), não só o número do protocolo.
8. Este repositório (`g_lab`) fica com: agente C#, simulador de analisador, capturas (sem dados reais) e documentação.

**Fases revisadas:** 0 preparação · 1 infraestrutura do painel `lab` · 2 cadastros e pacientes · 3 núcleo do LIS manual · 4 instrumentos e ingestão (parsers + simulador, maior risco, antes do agente) · 5 agente local · 6 bidirecional e demais equipamentos · 7 integrações e painéis · 8 migração e go-live (`migrate-lab`, MD5 não migra, ensaio com dados mascarados) · 9 conformidade contínua.

**Decisões de negócio ainda em aberto (do usuário):**
1. Modelos exatos do C4000 e do Pentra 60; Ethernet ou só RS-232; manuais de interface.
2. Uma empresa por laboratório (recomendado) ou vários laboratórios na mesma empresa.
3. Hospedagem: VPS atual, servidor do cliente ou híbrido.
4. Assinatura do laudo: A1 por laboratório (cofre cifrado), A3/token ou serviço de assinatura.
5. Encarregado (DPO) e base legal do uso analítico no banco de projetos.
6. Escopo da migração: histórico completo ou últimos N anos.
7. SUS: CADSUS, SIGTAP e faturamento (BPA).
8. Portal do paciente.
9. **Agente: C# (pedido original) ou Go** — a sessão sugere avaliar Go para reaproveitar os parsers da fase 4.
10. Prazo de retenção do laudo e do bruto de equipamento (jurídico).

**Próximo passo sugerido:** responder ao menos os itens 1, 2, 3 e 9 e aprovar a Fase 1. A proteção do repositório (`.gitignore`) já foi feita (D-004).
