# g-lab — Plano de trabalho

> Migração do LIS legado (PHP/MySQL) para **Go + PostgreSQL**, com integração a analisadores
> (rede + serial), agente **C#** e conformidade **LGPD**.
> Versão 0.1 — gerado a partir da análise de `laboratorio/` (legado). Itens marcados **[CONFIRMAR]** dependem de decisão ou de documentação do fabricante.

---

## 1. Diagnóstico do legado

### 1.1 O que existe
| Área | Situação |
|---|---|
| Aplicação | PHP puro, ~95 páginas em `paginas/` (sem framework, sem camada de serviço), sessão PHP, SQL direto nas páginas |
| Banco | MySQL, 34 tabelas `tb_*`, FKs presentes, charset misto (utf8mb3 e utf8mb4) |
| Integração com equipamentos | `integracao/server.js` (Node): servidor TCP **ASTM** (porta 4000) e **HL7/MLLP** (porta 5000), gerenciado por PM2 e controlado via `controle_server.php` |
| Laudo | `laudo/` e `validacao/` (emissão e validação por código), `assinador/`, assinatura digital (ICP-Brasil) e liberação com TOTP |
| Integrações externas | CADSUS (cliente SOAP/PDQ), SIGTAP (importação de procedimentos SUS) |
| Extras | Painel de chamada (TV), agenda, etiquetas/código de barras, laboratórios de apoio e contratos, galeria |
| Peso | 3.146 arquivos, a maioria é `vendor`/`node_modules`; o código próprio é pequeno |

### 1.2 Domínio (entidades principais)
`paciente` → `protocolo` (pedido, numeração `XXX+MÊS+ANO`) → `lista_exame_protocolo` (exame no pedido; flags `lancado`/`liberado`/`impresso`/`terceirizado`) → `item_exame_protocolo` (parâmetros/resultados, com `codigo_integrador` = chave de mapeamento com o analisador).
Cadastros: laboratório, bancada, material, método, exame, item de exame, referência (sexo/idade), etiqueta, perfil de usuário (25 permissões), posto de coleta, unidade e profissional solicitante, laboratório de apoio, contrato.
Assinatura: `assinatura_resultado`, `liberacao_resultado_totp`, `usuario_totp`.

### 1.3 Fluxo atual do analisador (a ser preservado e melhorado)
1. Equipamento envia ASTM (ENQ/STX/ETX/checksum/EOT) ou HL7 (MLLP) por TCP.
2. O servidor extrai **nº do protocolo** (registro `O`/`OBR`), **código do exame** (`R`/`OBX-3`) e resultado/unidade.
3. Casa por `protocolo.numero` + `item_exame_protocolo.codigo_integrador`; só atualiza se o exame **ainda não foi liberado**; marca `lancado=1`.
4. Grava o bruto em disco (`recebidos/`), sem política de retenção.
5. **Só recebe resultados.** Não há envio de *worklist* (consulta bidirecional) ao equipamento.

### 1.4 Problemas encontrados (viram requisitos)
**Segurança / LGPD (críticos)**
1. Credenciais do banco **fixas no código** (`funcoes/conexao.php`) e usuário de banco com acesso amplo.
2. Senhas com **MD5 legado** ainda aceito no login (há `password_hash` para novos, migração incompleta).
3. `tb_laboratorio` guarda **certificado `.pem` e `senha_pem`** (chave privada e senha) no banco, em texto.
4. `laboratorio.sql` contém **dados reais** (pacientes, CPF, CNS, nome da mãe, endereço, resultados). Está dentro do repositório git do projeto — **não pode ser commitado** (ver §7, ação imediata).
5. Dados pessoais sem criptografia em repouso e sem controle de acesso por registro; `tb_historico` é auditoria fraca (só tabela/chave/operação, sem antes/depois nem IP).
6. Arquivos brutos ASTM/HL7 (com identificadores de pacientes) gravados em pasta local sem retenção.
7. Páginas montam consulta SQL com entrada do usuário em vários pontos (revisar todas; migrar para consultas parametrizadas).
8. Porta ASTM/HL7 aberta em `0.0.0.0` sem TLS nem lista de IPs permitidos.

**Funcionais / arquitetura**
9. Sem multi-tenant real (`tb_laboratorio_id` espalhado; sem isolamento garantido).
10. Parser ASTM tolerante demais: ignora resultados sem protocolo, não trata *multi-frame* (ETB), não trata resultados repetidos/corrigidos, não guarda flags (H/L/A), não guarda status do equipamento.
11. Casamento por `numero` inteiro (≤ 2^31) — frágil; o ideal é **ID de amostra/código de barras** gerado pelo LIS.
12. Código do analisador amarrado ao item de exame (`codigo_integrador`) sem distinguir **equipamento** (o mesmo exame pode ter códigos diferentes por máquina).

---

## 2. Arquitetura-alvo

```
 Analisadores em rede ──TCP (ASTM / HL7-MLLP)──►  gateway-go  ─┐
                                                               ├─► fila (NATS/Redis/PG) ─► API Go ─► PostgreSQL
 Analisadores seriais ─RS-232─► Agente C# (PC local) ─HTTPS/mTLS┘                           │
                                                                                            ├─► Painel(is) web
                                                                                            └─► Banco de projetos (visões/ETL anonimizado)
```

- **API Go** (monólito modular, `chi` ou `echo`, `pgx`, `sqlc`, migrações com `goose`): casos de uso de LIS (pacientes, pedidos, coleta, resultados, liberação, laudo).
- **Gateway de instrumentos (Go)**: servidores TCP ASTM/E1381-E1394 e HL7/MLLP, um listener por equipamento cadastrado, com *driver* por modelo.
- **Agente C#** (.NET, Windows Service): lê porta serial, fala ASTM/ENQ-ACK na ponta serial, guarda **fila local persistente (SQLite cifrado)** e reenvia à API em HTTPS com mTLS; funciona offline.
- **PostgreSQL**: um banco, esquemas `core`, `lab`, `instrumento`, `audit`, `lgpd`; Row-Level Security por laboratório; `pgcrypto` para colunas sensíveis.
- **Observabilidade**: logs estruturados **sem dados pessoais**, métricas (Prometheus), tracing (OpenTelemetry).
- **Painel**: SPA (ou server-driven) consumindo a API REST/OpenAPI; integração com seu “painel” e com o banco de projetos por visões somente leitura/anonimizadas.

Decisão arquitetural central: **o equipamento nunca fala com o banco**. Tudo passa por uma camada de ingestão que valida, registra o bruto (cifrado), e só então aplica regras de negócio.

---

## 2.1 Identidade visual — **UYSI G-Lab**

Segue a família UYSI (mesma do projeto `UYSI Service`), com a marca composta **UYSI + G-Lab**.

- **Cores e tema:** reutilizar `assets/brand/tokens-cores.css` (cópia de `UYSI Service/referencia/uysiarbo/tokens-cores.css`): escala `--slate-*`, `--brand-*` (marca `--brand-600` `#155a9e`), estados `--ok`/`--perigo`/`--aviso`, fonte Inter, raio `0.75rem`, tema escuro por `:root[data-tema='escuro']` (preferência em `localStorage` com `try/catch`, chave `uysi.tema`). **Regra: a interface usa só variáveis de cor, nunca cor fixa.**
- **Marca:** símbolo UYSI (`assets/brand/uysi-symbol.png`) **ao lado** do ícone do G-Lab (microscópio do legado, `assets/brand/glab-icone.webp`), com o nome “UYSI G-Lab”. Uso: login, cabeçalho do menu, laudo/PDF, favicon e instalador do agente C#.
- **Não reaproveitar** `laboratorio/images/logo.png` (é a logo do cliente SSPM, não da marca do produto); o legado usa o `icone.webp` como logomarca do sistema.
- **Pendências de design:** versão vetorial (SVG) do microscópio e do símbolo; versão monocromática para impressão (laudo, etiqueta); lockup horizontal e vertical; favicon 32/192/512; versão para fundo escuro.

---

## 3. Modelo de dados (PostgreSQL) — pontos de projeto

- Chaves `uuid`/`bigint identity`; `timestamptz` sempre; `text` + `CHECK` em vez de `varchar(245)`.
- Tabelas novas essenciais:
  - `instrumento` (modelo, protocolo, modo `rede|serial`, host/porta ou agente, IPs permitidos, ativo).
  - `instrumento_exame_map` (instrumento × item de exame × **código do equipamento**, unidade, fator de conversão).
  - `amostra` (código de barras/ID único, tipo de material, coletada em, local).
  - `mensagem_instrumento` (bruto cifrado, hash, direção RX/TX, instrumento, recebido_em, status de processamento, retenção até).
  - `resultado` (valor, unidade, flag, **origem**: manual|instrumento, histórico de versões; correção gera nova versão, nunca sobrescreve).
  - `worklist` (pedido para o equipamento, status).
  - `agente` (id, certificado, versão, último *heartbeat*).
- **LGPD no esquema**: `paciente` com CPF/CNS/nome da mãe cifrados e *blind index* (HMAC) para busca; `consentimento`, `finalidade_tratamento`, `solicitacao_titular`, `incidente`, `log_acesso` (quem viu qual paciente), `audit_log` (antes/depois).
- Migração: ETL MySQL → PG com mapeamento de IDs, reconciliação por contagem/hash, **ensaio em ambiente com dados mascarados**.

---

## 4. Integração com analisadores

> Modelos citados: **C4000** e **Yumizen/Horiba Pentra 60** **[CONFIRMAR modelo exato e fabricante do “C4000”, versão de firmware e manual de interface (LIS Interface Specification)]**. Nada abaixo substitui o manual do fabricante: o plano é validar cada *driver* com **capturas reais**.

| Equipamento | Meio | Protocolo provável | Observação |
|---|---|---|---|
| Pentra 60 | Rede (se houver módulo Ethernet) ou serial RS-232 | ASTM (README do legado já cita ASTM) | No legado, a própria documentação avisa que modelos antigos podem ser **só RS-232** → cai no agente C# |
| C4000 | **[CONFIRMAR]** | ASTM e/ou HL7 | Definir se é bidirecional (worklist) |
| Outros em rede | TCP | ASTM / HL7 v2 (MLLP) | Driver genérico + perfil por modelo |

**Capacidades do gateway**
1. ASTM E1381 (camada de enlace: ENQ/ACK/NAK/EOT, quadros, checksum, ETB multi-quadro, timeouts, retransmissão) e E1394 (registros H, P, O, R, C, Q, L).
2. HL7 v2.x sobre MLLP (ORU^R01, ORM^O01, ACK com AA/AE/AR).
3. **Bidirecional**: consulta de *worklist* (registro Q ASTM / QRY HL7) e envio de pedidos.
4. Mapeamento por equipamento (`instrumento_exame_map`), conversão de unidade, tratamento de flags e de resultados qualitativos.
5. Idempotência (hash da mensagem + ID do equipamento), reprocessamento e **fila de mensagens com falha** (DLQ) visível no painel.
6. Segurança: lista de IPs permitidos, limites de taxa, TLS onde o equipamento permitir (ou túnel/VPN/stunnel), isolamento por VLAN.
7. Simulador de equipamento (testes automatizados) e reprodução de capturas reais.

**Agente C# (serial)**
- Windows Service (.NET 8), `System.IO.Ports`, configuração de porta/baud/paridade por perfil.
- Implementa o mesmo enlace ASTM e envia à API já **parseado + bruto**.
- Fila local persistente cifrada, reenvio com *backoff*, *heartbeat*, versão e **auto-atualização assinada**.
- Autenticação: certificado de cliente (mTLS) emitido por laboratório/agente; revogável.
- Sem acesso a banco; sem armazenar dados além da fila (purga após confirmação da API).

---

## 5. LGPD — requisitos (resumo executável)

Dados de saúde são **dados pessoais sensíveis** (LGPD art. 5º, II e art. 11).

| Tema | O que implementar |
|---|---|
| Base legal | Tutela da saúde / procedimento realizado por profissional de saúde (art. 11, II, “f”) e cumprimento de obrigação legal; **[CONFIRMAR com jurídico/DPO]** para pesquisa e uso analítico |
| Governança | Encarregado (DPO), **RIPD/DPIA** do sistema, registro das operações de tratamento (ROPA), política de privacidade |
| Minimização e finalidade | Campos mínimos por finalidade; painel de chamada exibe apenas nome social/iniciais, nunca resultado |
| Controle de acesso | RBAC por perfil (migrar as 25 permissões), RLS por laboratório, MFA para liberar laudo, “quebra de vidro” justificada |
| Criptografia | TLS 1.2+ em tudo; **cifra em repouso** (disco + colunas sensíveis); chaves em KMS/Vault, rotação; segredo e certificado A1 **fora do banco** |
| Auditoria | Log imutável de acesso e alteração (quem, quando, o quê, IP), retenção definida |
| Direitos do titular | Acesso, correção, anonimização, portabilidade, eliminação quando cabível, informação sobre compartilhamento — fluxo e prazos no sistema |
| Retenção | Tabela de temporalidade (prontuário/laudo: prazo legal; **[CONFIRMAR]** prazo aplicável ao laboratório) e descarte seguro do bruto de equipamento |
| Compartilhamento | Laboratórios de apoio, CADSUS/SUS, portal do paciente: contrato, finalidade e registro |
| Uso analítico | Banco de projetos/painel analítico recebe **somente dados anonimizados ou pseudonimizados** (visões dedicadas, k-anonimato quando houver publicação) |
| Incidentes | Plano de resposta, registro e comunicação à ANPD e aos titulares |
| Ambientes | **Nunca** usar dados reais em dev/homologação (usar dados sintéticos/mascarados) |

---

## 6. Plano em fases

| Fase | Entregas | Critério de pronto |
|---|---|---|
| **0. Preparação** (1–2 sem.) | Proteger repositório (§7), decisões abertas (§8), levantar manuais dos equipamentos, ambiente de dev com dados sintéticos | Checklist §8 respondido; `.gitignore` e política de dados em vigor |
| **1. Fundação** (2–3 sem.) | Repositório Go (monólito modular), CI, migrações, autenticação (Argon2id/JWT ou sessão), RBAC, auditoria, observabilidade, Docker Compose (PG + API) | Login, perfil e auditoria funcionando com testes |
| **2. Núcleo LIS** (4–6 sem.) | Cadastros, paciente, protocolo, coleta, lançamento manual, liberação com 2º fator, laudo em PDF, validação por código | Fluxo completo manual equivalente ao legado |
| **3. Gateway de instrumentos** (3–5 sem.) | ASTM + HL7/MLLP, mapeamento por equipamento, bruto cifrado, idempotência, DLQ, simulador, 1º equipamento em rede (homologação com capturas reais) | Resultados do equipamento chegam ao exame correto, 0 perda em teste de falha de rede/banco |
| **4. Agente C#** (3–4 sem.) | Serviço Windows, serial/ASTM, fila local, mTLS, atualização assinada, instalador | Pentra 60 (ou serial) operando offline e sincronizando ao voltar a rede |
| **5. Bidirecional e demais equipamentos** (3–4 sem.) | Worklist, C4000, drivers adicionais | Pedido aparece no equipamento sem digitação |
| **6. Integrações e painel** (3–4 sem.) | CADSUS, SIGTAP, laboratórios de apoio, painel de chamada, integração com seu painel e banco de projetos (visões anonimizadas) | Painéis consumindo a API/visões |
| **7. Migração e go-live** (2–3 sem.) | ETL MySQL→PG, reconciliação, ensaio, treinamento, *cutover* e plano de reversão | Paralelo validado; legado em somente leitura |
| **8. Conformidade contínua** | RIPD final, testes de intrusão, treinamento, rotina de direitos do titular | Relatório de conformidade assinado pelo DPO |

Estimativas são grosseiras (uma pessoa); ajustar após as decisões do §8.

---

## 7. Ação imediata de segurança

O legado está **dentro da pasta do repositório git** `g_lab` (que tem remoto no GitHub) e contém dados reais de pacientes (`laboratorio.sql`), credenciais (`funcoes/conexao.php`) e certificado/chaves no banco. **Não faça `git add .` antes de proteger a pasta.** Recomendado:
1. ~~Adicionar `.gitignore` cobrindo `laboratorio/`, `*.sql`, `*.pem`, `.env`, `recebidos/`, `uploads/`.~~ **Feito** (ver `.gitignore` e `docs/DECISOES.md`, D-004).
2. Se for versionar o legado, fazê-lo em repositório **privado e separado**, sem o dump e sem credenciais.
3. Tratar as credenciais do MySQL do legado como expostas (trocar quando o legado for desativado ou se já tiverem circulado).
4. Se o repositório `glab` for público, considerar torná-lo privado.

---

## 8. Decisões em aberto

1. Modelo exato do **C4000** (fabricante), versões e manuais de interface; equipamentos adicionais previstos.
2. O Pentra 60 em uso tem Ethernet ou só RS-232?
3. Multi-laboratório (vários CNPJ/CNES) ou instalação única por cliente? (define RLS e deploy)
4. Hospedagem: nuvem, servidor próprio ou híbrido? (define KMS, backup, rede dos equipamentos)
5. Painel: web novo, ou o “painel” existente que vai consumir a API? Qual stack de front?
6. Assinatura digital: manter certificado A1 por laboratório (precisa de cofre) ou A3/token/serviço de assinatura?
7. Portal do paciente (resultado pela web) entra no escopo?
8. Quem é o **DPO/encarregado** e qual base legal será registrada para o uso no banco de projetos?
9. Migração: histórico completo de resultados ou apenas cadastros e últimos N anos?
10. SUS: manter CADSUS/SIGTAP e faturamento (BPA) no escopo?

---

## 9. Próximos passos sugeridos

1. Você responde (ou prioriza) o §8.
2. Eu crio o `.gitignore` e a estrutura inicial do repositório (`/api`, `/gateway`, `/agent`, `/db`, `/docs`).
3. Desenhar o DDL PostgreSQL (§3) e o contrato OpenAPI do núcleo LIS.
4. Construir o simulador de analisador + parser ASTM em Go (maior risco técnico, atacar primeiro).
