# Documentação do Projeto: Portal de Mobilidade Interna (SESAU)

## 1. Apresentação

O **Portal de Mobilidade Interna da SESAU** é uma solução digital desenvolvida para modernizar, simplificar e dar transparência ao processo de movimentação interna dos servidores da Secretaria de Saúde do Recife. 

O sistema substitui fluxos baseados em papel ou e-mails desestruturados por um ambiente web intuitivo, onde o servidor pode:
- Consultar seus dados funcionais.
- Solicitar mobilidade (transferência) para outras unidades.
- Acompanhar o status das suas solicitações.
- Obter comprovantes formais de suas requisições.

A solução foi projetada com foco na experiência do usuário (UX), garantindo acessibilidade em dispositivos móveis e desktops, e integra-se diretamente aos fluxos de automação da prefeitura via n8n.

---

## 2. Público Alvo

O sistema destina-se exclusivamente a:

- **Servidores Ativos da SESAU:** Profissionais lotados na Secretaria de Saúde, de todos os vínculos (Estatutários, CTD, Comissionados), que desejam solicitar mudança de lotação.
- **Equipe de RH / Gestão de Pessoas:** Que receberá os dados padronizados e estruturados para análise, eliminando a necessidade de redigitação.

---

## 3. Guia de Usabilidade

### Acesso ao Portal
O acesso é realizado através de autenticação segura (**Keycloak**), integrado ao servidor de diretório (AD) da Prefeitura. Ao acessar, o servidor deve utilizar suas credenciais de rede para garantir o acesso a uma área logada exclusiva.

**URLs de Acesso:**
- **Portal do Servidor:** [https://n8n.homolog.app.emprel.gov.br/webhook/portal-sesau-servidores](https://n8n.homolog.app.emprel.gov.br/webhook/portal-sesau-servidores)
- **Formulário de Solicitação:** [https://n8n.homolog.app.emprel.gov.br/webhook/f188b0f4-f4c9-455b-8dd9-e800c44b46e2](https://n8n.homolog.app.emprel.gov.br/webhook/f188b0f4-f4c9-455b-8dd9-e800c44b46e2)

### Fluxo de Solicitação
1.  **Início:** No painel principal, clique em "Nova Solicitação".
2.  **Identificação:** Seus dados funcionais (Nome, Matrícula, Cargo) são carregados automaticamente. Verifique se estão corretos.
3.  **Contatos e Endereço:** Atualize seu endereço, telefone (WhatsApp) e e-mail. Estes dados são cruciais para a comunicação sobre o resultado do pleito.
4.  **Lotação Atual:** Informe onde você trabalha atualmente, seu turno e regime.
5.  **Preferências:**
    - Indique para onde deseja ir.
    - Você pode adicionar múltiplas opções de unidades desejadas.
    - Se houver permuta (troca casada com outro servidor), informe a matrícula do colega.
6.  **Justificativa:** Selecione o motivo da solicitação e anexe documentos comprobatórios, se necessário.
7.  **Conclusão:** Revise os dados e envie. Um comprovante PDF será gerado automaticamente.

### Acompanhamento
No painel inicial, o cartão "Minhas Solicitações" exibe o status atual do processo (ex: "Em análise", "Aprovado", "Aguardando Vaga").

### Dicas de Uso
- **Navegadores:** Compatível com Chrome, Firefox, Edge e Safari (versões recentes).
- **Mobile:** O sistema é responsivo e pode ser acessado via celular ou tablet.

---

## 4. Dados Técnicos

### Arquitetura Frontend
- **Linguagens:** HTML5, CSS3, JavaScript (ES6+).
- **Framework CSS:** Bootstrap 5.3 (para layout responsivo e componentes).
- **Bibliotecas:** 
    - `Choices.js`: Para caixas de seleção (selects) avançadas e pesquisáveis.
    - Google Fonts (Inter, Public Sans): Para tipografia institucional.
- **Estilização:** CSS customizado com variáveis (CSS Variables) para fácil manutenção de temas e cores institucionais (Azul SESAU).

### Fluxo de Processamento (n8n)
O processamento das solicitações é orquestrado por um fluxo no **n8n** que executa as seguintes etapas:

1.  **Recepção e Validação:**
    - O webhook `mobilidade-html` recebe os dados do formulário.
    - Scripts JavaScript (Nodes `Code`) formatam datas (`DD/MM/YYYY` -> `YYYY-MM-DD`) e padronizam campos de texto.

2.  **Persistência e Atualização Cadastral (PostgreSQL):**
    - **Mestre:** Atualiza os dados de contato (email, telefone) e endereço na tabela mestre (`mobilidade_segtes_relacao_nominal`) para garantir que a base esteja sempre corrente.
    - **Histórico:** Remove solicitações anteriores não processadas da mesma matrícula para evitar duplicidade.
    - **Registro:** Insere a nova solicitação na tabela `mobilidade_dados_recebidos`.

3.  **Cálculo de Ranking:**
    - O sistema executa uma *stored procedure* ou query complexa (`ranking_mobilidade`) que classifica os candidatos baseando-se em:
        - Prioridade informada pelo servidor (1ª, 2ª, 3ª opção).
        - Critérios de desempate: Data de solicitação, Data de submissão, Data de admissão e Data de nascimento.
    - O resultado é salvo na tabela `ranking_mobilidade`.

4.  **Integração com SE Suite (SOAP):**
    - O fluxo consome o *Web Service* do SE Suite (`.../ws/wf_ws.php`).
    - **Ação:** `newChildEntityRecord` ou `editEntityRecord`.
    - **Objetivo:** Criar ou atualizar automaticamente o processo administrativo de mobilidade no sistema de gestão de processos (BPM/SE Suite), garantindo que a solicitação tramite oficialmente.

5.  **Notificação:**
    - O servidor recebe um e-mail automático (via **Mailgun** ou **Gmail API**) contendo:
        - Protocolo do processo (`wfid`).
        - Resumo das opções de lotação escolhidas.
        - Classificação (Ranking) preliminar.

### Estrutura de Bando de Dados (PostgreSQL)
O sistema consulta/persiste dados nas seguintes tabelas:
- `mobilidade_segtes_relacao_nominal`: Base de dados dos servidores (dados pessoais, funcionais).
- `unidades_sesau`: Catálogo de unidades de saúde, distritos, turnos e regimes.
- `ranking_mobilidade`: Histórico de solicitações e classificação.

### Integração (Backend)
- **Comunicação:** Webhooks via API REST.
- **Motor de Processamento/Hospedagem:** n8n (Workflow Automation).
- **Autenticação:** Keycloak (AD da Prefeitura).
- **Endpoints (Webhooks):**
    - `portal-sesau-servidores`: Renderização do Portal ([link](https://n8n.homolog.app.emprel.gov.br/webhook/portal-sesau-servidores)).
    - `f188b0f4-f4c9-455b-8dd9-e800c44b46e2`: Renderização do Formulário ([link](https://n8n.homolog.app.emprel.gov.br/webhook/f188b0f4-f4c9-455b-8dd9-e800c44b46e2)).
    - `mobilidade-html`: Recebimento/Persistência das solicitações.
    - `unidades`, `turnos`, `regimes`, `distritos`: Endpoints auxiliares para carregamento dinâmico de selects.
- **Integração Externa (SE Suite):**
    - Utiliza o endpoint SOAP `https://sesuite.recife.pe.gov.br/apigateway/se/ws/wf_ws.php` para criação automática de processos de workflow (Action: `newWorkflow`) após a submissão.
- **Segurança:** Tráfego criptografado (HTTPS) e validação de dados no client-side e server-side.

### Estrutura de Arquivos Principais
- `portal.html`: Dashboard principal do servidor.
- `formulario.html`: Interface de preenchimento da solicitação.
- `assets/`: Imagens, ícones e scripts auxiliares.

---

## 5. Biblioteca de Dados (Dicionário de Dados)

Abaixo estão listados os principais campos capturados e processados pelo sistema:

### Dados do Servidor (Leitura/Identificação)
| Campo | Descrição | Tipo | Origem |
|-------|-----------|------|--------|
| `matricula` | Matrícula funcional do servidor | Numérico | Sistema RH / n8n |
| `nome` | Nome completo | Texto | Sistema RH / n8n |
| `cpf` | CPF do servidor | Texto | Sistema RH / n8n |
| `cargo` | Cargo ocupado | Texto | Sistema RH / n8n |
| `vinculo` | Tipo de vínculo (Estatutário, etc.) | Texto | Sistema RH / n8n |

### Dados de Contato (Atualizáveis)
| Campo | Descrição | Obrigatório | Validação |
|-------|-----------|-------------|-----------|
| `whatsapp` | Número de celular com DDD | Sim | Numérico |
| `email` | E-mail pessoal ou corporativo | Sim | Formato de e-mail |
| `cep` | Código de Endereçamento Postal | Sim | 8 dígitos |

### Dados da Solicitação
| Campo | Descrição | Notas |
|-------|-----------|-------|
| `unidade_atual` | Onde o servidor está lotado hoje | Seleção de lista fechada |
| `unidades_destino` | Lista de unidades desejadas | Múltipla escolha (Array) |
| `motivo` | Justificativa do pedido | Texto livre ou Seleção |
| `permuta_matricula` | Matrícula do servidor para troca | Opcional |

---

## 6. Termo de Aceite de Entrega de  Projeto

**PROJETO:** Portal de Mobilidade Interna SESAU  
**CLIENTE:** Secretaria de Saúde do Recife  
**DATA DE ENTREGA:** 31/01/2026

Declaro, para os devidos fins, que a solução de software descrita nesta documentação foi entregue, testada e homologada, estando em conformidade com os requisitos funcionais solicitados.

**Itens Entregues:**
- [x] Código Fonte Completo (Frontend HTML/CSS/JS).
- [x] Integrações configuradas (Webhooks n8n).
- [x] Documentação Técnica e de Usuário.

**Considerações:**
O sistema encontra-se em condições de operação (produção). A manutenção evolutiva ou corretiva futura será objeto de novas tratativas, se necessário.

<br>
<br>

__________________________________________________________________
**Responsável Técnico (Desenvolvimento)**

<br>
<br>

__________________________________________________________________
**Responsável pelo Recebimento (Cliente)**
Secretaria de Saúde do Recife

<br>
Local: Recife/PE, ______ de __________________________ de 2026.
