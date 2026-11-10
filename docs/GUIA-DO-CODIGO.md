# AutoFlow — guia arquivo por arquivo

O AutoFlow reúne atendimento pelo WhatsApp, CRM, catálogo, estoque, entregas e agendamentos em um painel multiempresa. O painel usa Flask e templates Jinja. A página pública usa HTML, CSS e JavaScript próprios. As integrações principais são Evolution API para WhatsApp, Groq para IA e SMTP para recuperação de senha.

O inventário cobre os 278 arquivos rastreados no checkout af5f850. Inclui código, configurações, testes, documentação, imagens, recursos 3D e cópias de distribuição. Arquivos locais ignorados, dependências instaladas e configurações privadas ficam fora do escopo. A verificação é por leitura do código e comparação com o inventário Git.

## Como as partes se conectam

| Fluxo | Caminho pelo código |
| --- | --- |
| Acesso ao painel | O Flask cria a aplicação, registra os Blueprints e inicializa extensões. As rotas apontam para os controllers, que tratam a requisição e usam serviços e modelos. O resultado pode ser uma página Jinja ou uma resposta JSON usada pelo JavaScript do painel. |
| Mensagem recebida do WhatsApp | A Evolution chama o webhook. O backend identifica a empresa e a conversa, valida e registra o evento, aplica idempotência e cotas e persiste a intenção de trabalho no outbox. O Celery processa a mensagem e coordena o envio da resposta. |
| Resposta com IA | O worker chama o serviço de IA com contexto da empresa e histórico da conversa. A Groq pode solicitar ferramentas registradas para consultar dados ou executar ações permitidas. A resposta passa pelo fluxo de persistência e envio do WhatsApp. |
| Atendimento humano | O painel permite assumir conversas, enviar mensagens e devolver o atendimento à IA. Estado, responsável e histórico ficam persistidos na conversa e nas mensagens. |
| Persistência e processamento | O MySQL armazena dados de negócio, incluindo o outbox. O Redis serve ao Celery como broker e backend de resultados. O Celery beat agenda novas tentativas de publicação das entradas pendentes. |

## Por onde começar a leitura

- **Entender como o painel inicia** — acompanhe a criação da aplicação em [app/\_\_init\_\_.py](../app/__init__.py) e a configuração das extensões em [app/extensions.py](../app/extensions.py).
- **Acompanhar uma ação do painel** — use o cadastro de produto como exemplo. O adaptador de [rotas de produtos](../app/routes/products.py) liga o [controller](../app/controllers/products_controller.py) ao Flask. O [serviço de criação](../app/services/products/create_product_service.py) coordena produto, variantes e estoque inicial.
- **Acompanhar uma mensagem do WhatsApp** — comece pelo [controller do webhook](../app/controllers/whatsapp_controller.py), siga para o [gateway Evolution](../app/services/whatsapp_service.py), passe pelo [outbox](../app/services/outbox_service.py) e chegue às [tarefas do Celery](../app/tasks/ai_tasks.py).

## Backend Flask

### Módulos centrais de `app/`

- `app/__init__.py` — fábrica Flask: valida o ambiente, configura proxy/logging/extensões/Celery, carrega modelos, registra blueprints, sessão de login, handlers de erro, landing legada, cabeçalhos de segurança, contexto global dos templates e comando `check-config`.
- `app/config.py` — configurações base e subclasses de desenvolvimento, produção e testes. Lê variáveis de ambiente para banco, Redis/Celery, Groq, Evolution API, SMTP, uploads, CORS, cookies e limites; ajusta a URL local do Redis para alinhar senha.
- `app/extensions.py` — instancia as extensões Flask-SQLAlchemy, Migrate, Login, CSRF, CORS e rate limiter; `make_celery` associa Celery à aplicação Flask e ao contexto/configuração dela.
- `app/security.py` — inicializa `MultiFernet` a partir da chave de criptografia configurada e fornece operações para cifrar/decifrar credenciais ou tokens armazenados.
- `app/tenant.py` — resolve a empresa ativa a partir da sessão autenticada, aplica consultas/obtenção por empresa e oferece decorador de autorização por nível `AGENT`, `ADMIN` e `OWNER`.
- `app/validation.py` — schemas Pydantic estritos para cadastro, login, cliente, produto, estoque, agendamento e mensagem; extrai dados de JSON/formulários e retorna erros de validação sem URLs externas.
- `app/controllers/__init__.py` — pacote dos controladores HTTP; não adiciona lógica própria.
- `app/routes/__init__.py` — pacote de blueprints e adaptadores de rotas; não implementa lógica própria.
- `app/tasks/__init__.py` — pacote das tarefas assíncronas; a implementação está em `ai_tasks.py`.

### Controladores HTTP

- `app/controllers/http.py` — helpers comuns para converter horários UTC ao fuso da empresa, decidir entre resposta JSON e redirecionamento/flash, ler payloads, coerções, serialização, respostas de sucesso/erro e gravação de auditoria na transação corrente.
- `app/controllers/appointments_controller.py` — endpoints de agenda: lista e disponibilidade, criação/cancelamento/confirmação, serviços, profissionais, horários de funcionamento, bloqueios de agenda e configurações. Serializa dados monetários e horários; chama serviços de domínio e registra notificações/auditoria quando aplicável.
- `app/controllers/auth_controller.py` — cadastro de usuário/empresa/membership inicial, login/logout e fluxos de solicitar, validar e concluir redefinição ou troca de senha; valida destino de redirecionamento para evitar URL externa.
- `app/controllers/conversations_controller.py` — lista e atualiza conversas, fornece detalhe e criação, atribuição a atendente, retorno à IA, resolução, envio de mensagem e gestão de tags. Serializa mensagens/conversas e usa revisionamento para atualizações da lista.
- `app/controllers/customers_controller.py` — listagem e CRUD/arquivamento de clientes, detalhe, pipeline CRM e atualização de etapa, tags e notas; normaliza dados e valores de oportunidade e usa serviços de cliente.
- `app/controllers/dashboard_controller.py` — compõe métricas e dados do painel para a empresa, serve página e endpoint de métricas, e marca notificações como lidas.
- `app/controllers/delivery_controller.py` — mantém regras de entrega por cidade/bairro/faixa de CEP, preço, mínimo e prazo; permite ativar/desativar, configurar retirada e simular cotação para um pedido/endereço.
- `app/controllers/faq_controller.py` — gerencia perguntas frequentes e documentos da base de conhecimento; limita tamanho e valida conteúdo de uploads, extrai texto de DOCX/PDF, remove documentos e expõe busca.
- `app/controllers/inventory_controller.py` — consulta estoque e itens abaixo do mínimo, registra movimentações, altera mínimo e apresenta histórico, mantendo escopo por empresa e passando operações ao serviço de inventário.
- `app/controllers/products_controller.py` — catálogo, CRUD/arquivamento/ativação e variantes de produtos. Também valida PNG/JPEG/WebP e dimensões, limita tamanho/descompressão e quota por empresa, armazena imagens gerenciadas e limpa/substitui arquivos de forma coordenada com o salvamento.
- `app/controllers/settings_controller.py` — leitura e gravação das instruções/configurações da IA e da integração WhatsApp por empresa; permite verificar conexão, enviar teste e desconectar a integração.
- `app/controllers/whatsapp_controller.py` — recebe webhook da Evolution API, valida/autentica origem conforme configuração, encaminha eventos de mensagens/status para processamento e atualiza estado da integração.
- `app/controllers/whatsapp/__init__.py` — pacote do controller específico de WhatsApp; sem implementação própria.
- `app/controllers/whatsapp/whatsapp_qrcode_controller.py` — controller fino que recebe o serviço de geração de QR Code e expõe sua operação como endpoint/blueprint dedicado.

### Adaptadores em `app/routes/`

Estes módulos associam blueprints e handlers já definidos em `app/controllers/` e preservam imports/nomeações usados pelo restante da aplicação. Os módulos de compatibilidade copiam os símbolos públicos do controller correspondente.

- `app/routes/appointments.py` — reexporta `AppointmentsController`, seu blueprint e handlers da agenda, profissionais, serviços e horários.
- `app/routes/auth.py` — reexporta o blueprint e handlers de cadastro, sessão e redefinição de senha.
- `app/routes/conversations.py` — reexporta o blueprint e handlers de listagem, detalhe, atendimento, mensagens e tags.
- `app/routes/customers.py` — adaptador de compatibilidade que publica os símbolos públicos de `customers_controller`.
- `app/routes/dashboard.py` — reexporta blueprint e handlers de painel, métricas e leitura de notificação.
- `app/routes/delivery.py` — adaptador de compatibilidade para os símbolos públicos de `delivery_controller`.
- `app/routes/faq.py` — adaptador de compatibilidade para os símbolos públicos de `faq_controller`.
- `app/routes/helpers.py` — reexporta utilitários HTTP compartilhados de `controllers/http.py` para imports legados.
- `app/routes/inventory.py` — adaptador de compatibilidade para os símbolos públicos de `inventory_controller`.
- `app/routes/products.py` — adaptador de compatibilidade para os símbolos públicos de `products_controller`.
- `app/routes/settings.py` — reexporta o blueprint e handlers das configurações de IA e WhatsApp.
- `app/routes/whatsapp.py` — reexporta blueprint/controller e handler do webhook Evolution.

### Tarefas assíncronas

- `app/tasks/ai_tasks.py` — tarefas Celery para processar mensagens recebidas (concorrência por conversa, pausa/handoff, resposta da IA, persistência e envio via outbox), enviar mensagens WhatsApp com idempotência e proteção contra duplicação em resultado ambíguo, resumir conversas em lotes, enviar email de redefinição com token cifrado, despachar outbox, criar notificações e processar notificações de eventos de agendamento. Aplica retries e atualiza estados de falha/processamento.

### Modelos em `app/models/`

- `app/models/__init__.py` — importa e exporta os mapeadores do domínio para registrar todas as tabelas antes de migrações e disponibilizar uma API de importação única.
- `app/models/base.py` — mixins reutilizáveis: timestamps UTC, chave/relacionamento e consultas por empresa, representação e operações CRUD com commit opcional; inclui serialização seletiva em dicionário.
- `app/models/company.py` — `Company` representa workspace/empresa, plano, status, fuso e configurações; `CompanyMember` vincula usuário, papel e estado de membership. Ambos dão suporte ao tenant e determinam membership ativo.
- `app/models/user.py` — usuário, hash de senha, estado e dados de recuperação; implementa Flask-Login, ID de sessão versionado pelo momento de troca de senha e resolução da empresa/papel ativo pelas memberships.
- `app/models/customer.py` — cadastro multiempresa de clientes com telefone normalizado, contato, notas, estado, etapa CRM, oportunidade, origem e preferências; relaciona conversas, agendamentos e tags e mantém alias legado `company_name`.
- `app/models/conversation.py` — conversa de canal externo ligada a cliente e integração, com atendente, estados de conversa/IA, pedido humano, não lidas, último contato, resumo e memória; relaciona mensagens e tags.
- `app/models/message.py` — mensagens de entrada/saída/sistema, remetente, conteúdo, ids externos/idempotência, estados de entrega e processamento, payload/metadados e timestamps; inclui relações de reply e alias `text` para `content`.
- `app/models/tag.py` — define tags tipadas por empresa e vínculos de associação com clientes ou conversas, com unicidade por escopo e cascata ao remover relações.
- `app/models/product.py` — catálogo e variantes com SKU, preço normal/promocional, atributos e imagem; expõe preço efetivo e relaciona estoque de produto/variante.
- `app/models/inventory.py` — saldo, reservado e mínimo de estoque por produto ou variante; calcula disponível/baixo estoque. `InventoryMovement` registra delta, saldos antes/depois, ator, razão, referência e idempotência.
- `app/models/delivery.py` — regra de preço e elegibilidade de entrega por cidade, bairro e faixa postal, com pedido mínimo, dias, retirada, prioridade e ativação.
- `app/models/appointment.py` — conjunto de agenda: `Service` define serviço/duração/preço; `Professional` relaciona profissionais a serviços; `BusinessHour` define janela semanal; `ScheduleBlock` bloqueia intervalos; `Appointment` liga cliente/serviço/profissional e mantém status, duração, notas, cancelamento e idempotência. A tabela associativa inclui `company_id` para escopo multiempresa.
- `app/models/knowledge.py` — `FAQ` guarda pares pergunta/resposta categorizados e priorizados; `KnowledgeDocument` mantém conteúdo extraído, origem, tipo MIME, checksum, status e metadados para consulta da base de conhecimento.
- `app/models/settings.py` — `AISettings` armazena personalidade, instruções, modelo, limites e controles de resposta da IA por empresa; `WhatsAppIntegration` armazena instância, status e último erro/webhook da conexão.
- `app/models/notification.py` — notificações multiempresa direcionadas opcionalmente a um usuário, com tipo, texto, link, dados, chave de idempotência e estado de leitura.
- `app/models/audit.py` — trilha append-only com ator, ação, tipo/id da entidade, alterações/contexto JSON, IP, agente do navegador e horário; usada para registrar operações administrativas/de domínio.
- `app/models/outbox.py` — outbox transacional das tarefas Celery permitidas, com payload e chave idempotente única, estado pendente/despachado, agenda de disponibilidade, tentativas, erro e timestamps; vincula opcionalmente a empresa.

## Serviços da aplicação (`app/services`)

### Compatibilidade e registro de ferramentas

- `app/services/__init__.py` — expõe sob demanda as antigas classes-fachada (`*Service`) para compatibilidade; não anuncia esses nomes como API nova.
- `app/services/ai_tools.py` — define os esquemas e manipuladores de ferramentas que a IA pode executar, incluindo catálogo, estoque, entrega, conhecimento, agenda, cliente e transferência para atendimento humano; constrói o registro padrão e aplica restrições ao contexto da conversa.
- `app/services/tool_registry.py` — implementa contexto, especificações e registro de ferramentas, conversão dos esquemas para o formato Groq e validação/execução controlada das entradas.

### IA

- `app/services/ai/__init__.py` — reexporta os dois casos de uso de IA disponíveis neste pacote.
- `app/services/ai/answer_conversation_service.py` — caso de uso fino que encaminha a geração de resposta ao gateway Groq legado, aceitando histórico/cursor e lote de mensagens recebidas.
- `app/services/ai/summarize_conversation_service.py` — caso de uso fino para resumir conversa ou processar um lote incremental via gateway Groq legado.
- `app/services/groq_service.py` — gateway de IA: faz requisições ao provedor Groq, executa ciclos de ferramentas, monta o prompt da conversa e gera respostas e resumos.

### Agendamentos

- `app/services/appointment_service.py` — implementação das operações de agendamento: normaliza horários e fuso, calcula disponibilidade respeitando expediente, bloqueios e reservas, cria agendamentos e cancela-os; `AppointmentService` é alias de compatibilidade.
- `app/services/appointments/__init__.py` — reexporta os casos de uso de criação, consulta, disponibilidade e cancelamento.
- `app/services/appointments/cancel_appointment_service.py` — caso de uso que encaminha o cancelamento (com motivo e opção transacional) à fachada legada de agendamentos.
- `app/services/appointments/create_appointment_service.py` — caso de uso que encaminha a criação transacional, incluindo cliente, serviço, profissional e chaves de idempotência, à fachada legada.
- `app/services/appointments/get_appointment_service.py` — caso de uso que busca um agendamento pelo identificador dentro da empresa.
- `app/services/appointments/get_available_appointments_service.py` — caso de uso que consulta horários disponíveis para serviço, dia e, opcionalmente, profissional.

### Autenticação e e-mail

- `app/services/auth/__init__.py` — reexporta o caso de uso de envio de e-mail para redefinição de senha.
- `app/services/auth/send_password_reset_email_service.py` — caso de uso fino que delega o envio da redefinição de senha à fachada legada de e-mail.
- `app/services/email_service.py` — gateway SMTP para verificar configuração e enviar e-mails transacionais de redefinição de senha.

### Catálogo e produtos

- `app/services/catalog_service.py` — consultas somente de leitura ao catálogo: pesquisa, busca por ID/SKU e serialização de produtos, variantes e disponibilidade; `CatalogService` mantém compatibilidade.
- `app/services/catalog/__init__.py` — reexporta os casos de uso de busca e pesquisa de produto.
- `app/services/catalog/get_product_service.py` — caso de uso fino que busca um produto por ID ou SKU por meio das consultas legadas de catálogo.
- `app/services/catalog/search_products_service.py` — caso de uso fino que pesquisa produtos por texto/categoria com limite e filtro de ativos, delegando à fachada legada.
- `app/services/products/__init__.py` — reexporta os casos de uso de criação, edição, listagem e arquivamento de produtos e variantes.
- `app/services/products/archive_product_service.py` — arquiva o produto e suas variantes ativas sem apagar registros, com suporte a commit ou flush.
- `app/services/products/archive_product_variant_service.py` — desativa uma variante pertencente à empresa, preservando seu histórico.
- `app/services/products/create_product_service.py` — cria produto, variantes e respectivos registros/quantidades iniciais de estoque como uma operação coordenada, validando dados e controlando a transação.
- `app/services/products/create_product_variant_service.py` — cria uma variante para produto da empresa e inicializa seu registro e saldo de estoque.
- `app/services/products/list_products_service.py` — monta consulta tenant-safe de produtos com filtros opcionais por texto, categoria e estado ativo.
- `app/services/products/update_product_service.py` — atualiza campos permitidos do produto da empresa, limita textos e valida o nome antes de persistir.
- `app/services/products/update_product_variant_service.py` — atualiza campos permitidos de uma variante da empresa, normalizando textos e persistindo alterações.

### Conversas e clientes

- `app/services/conversation_service.py` — implementa operações de conversa e mensagens: obtenção/criação, registro de entrada/saída, atribuição, transferência entre IA e humano, leitura, histórico para Groq, resumo, etiquetas e status de entrega; `ConversationService` é alias compatível.
- `app/services/conversations/__init__.py` — reexporta os casos de uso individuais de conversa, mensagens e histórico.
- `app/services/conversations/add_conversation_tag_service.py` — adiciona uma etiqueta a uma conversa via fachada legada, preservando a opção de controlar a transação.
- `app/services/conversations/build_groq_conversation_history_range_service.py` — obtém um intervalo delimitado do histórico da conversa formatado para Groq, usando cursores e limite.
- `app/services/conversations/build_groq_conversation_history_service.py` — obtém o histórico recente da conversa no formato aceito pela Groq.
- `app/services/conversations/claim_conversation_service.py` — atribui a conversa a um atendente específico por meio da operação legada.
- `app/services/conversations/get_conversation_service.py` — busca uma conversa da empresa, opcionalmente bloqueando o registro para atualização.
- `app/services/conversations/get_or_create_conversation_service.py` — obtém ou abre a conversa do cliente para canal/integração externa informados.
- `app/services/conversations/list_recent_messages_service.py` — lista mensagens recentes da conversa, opcionalmente limitadas por um cursor.
- `app/services/conversations/mark_conversation_read_service.py` — marca a conversa como lida (zera pendências) pela operação legada.
- `app/services/conversations/record_inbound_message_service.py` — registra uma mensagem recebida, incluindo ID externo, tipo e payload, por meio da fachada legada.
- `app/services/conversations/record_outbound_message_service.py` — registra mensagem enviada e seus metadados de remetente, resposta e idempotência.
- `app/services/conversations/return_conversation_to_ai_service.py` — devolve à IA o controle de uma conversa anteriormente transferida, opcionalmente identificando o usuário.
- `app/services/conversations/set_conversation_summary_service.py` — salva o resumo e o cursor de mensagens processadas, aceitando cursor esperado para controle concorrente.
- `app/services/conversations/transfer_conversation_to_human_service.py` — transfere o atendimento para uma pessoa e registra motivo por meio da operação legada.
- `app/services/conversations/update_message_status_service.py` — encaminha a atualização do status de entrega de uma mensagem, informado pelo provedor externo.
- `app/services/customer_service.py` — implementa operações de CRM para clientes: normalização de telefone, busca/listagem, criação/atualização, upsert de WhatsApp e etiquetas; `CustomerService` é alias compatível.
- `app/services/customers/__init__.py` — reexporta os casos de uso de consulta, cadastro, atualização, arquivamento, etiquetas e sincronização de clientes do WhatsApp.
- `app/services/customers/add_customer_tag_service.py` — adiciona etiqueta identificada pelo nome a um cliente da empresa, por meio da fachada legada.
- `app/services/customers/archive_customer_service.py` — arquiva cliente da empresa sem removê-lo, usando a operação legada.
- `app/services/customers/create_customer_service.py` — cria cliente com dados de contato e CRM via fachada legada.
- `app/services/customers/find_customer_by_phone_service.py` — localiza cliente da empresa pelo telefone normalizado via fachada legada.
- `app/services/customers/get_customer_service.py` — busca cliente por identificador dentro da empresa.
- `app/services/customers/list_customers_service.py` — lista clientes da empresa com filtros de pesquisa, etapa de CRM e status.
- `app/services/customers/update_customer_service.py` — aplica alterações de perfil/CRM ao cliente da empresa via fachada legada.
- `app/services/customers/upsert_customer_from_whatsapp_service.py` — localiza ou cria/atualiza cliente a partir do telefone e nome recebidos do WhatsApp.

### Entrega

- `app/services/delivery_service.py` — calcula opções de entrega configuradas para uma empresa a partir de cidade, bairro, CEP e valor do pedido, incluindo regras de retirada e normalização de CEP.
- `app/services/delivery/__init__.py` — reexporta o caso de uso de consulta de opções de entrega.
- `app/services/delivery/get_delivery_options_service.py` — caso de uso fino que encaminha os critérios do pedido ao calculador de entrega legado.

### Estoque

- `app/services/inventory_service.py` — implementa operações de estoque com isolamento por empresa, bloqueios, validação de saldo reservado, idempotência e registro de movimentações/auditoria; também permite localizar estoque e consultar itens baixos; `InventoryService` é alias compatível.
- `app/services/inventory/__init__.py` — reexporta os casos de uso de consulta, criação, ajuste/reserva e histórico de estoque.
- `app/services/inventory/adjust_stock_service.py` — caso de uso fino que encaminha ajuste de quantidade e metadados auditáveis à operação legada.
- `app/services/inventory/find_inventory_for_catalog_service.py` — encontra registros de estoque para o catálogo usando filtros por produto, variante, SKU, cor ou tamanho.
- `app/services/inventory/get_inventory_service.py` — busca registro de estoque da empresa, com opção de bloqueio para atualização.
- `app/services/inventory/get_or_create_inventory_service.py` — obtém ou cria o registro de estoque para exatamente um produto ou variante.
- `app/services/inventory/list_inventory_history_service.py` — retorna o estoque e suas movimentações recentes, limitando a consulta a 250 registros e escopando-a à empresa.
- `app/services/inventory/list_low_stock_inventory_service.py` — caso de uso fino que lista registros cujo estoque disponível atingiu o limite baixo configurado.
- `app/services/inventory/reserve_stock_service.py` — caso de uso fino que reserva quantidade disponível com suporte a chave de idempotência.
- `app/services/inventory/set_stock_service.py` — define a quantidade total alvo delegando à operação de ajuste legada.
- `app/services/inventory/update_minimum_inventory_service.py` — atualiza o limite de estoque baixo, rejeita valores negativos e bloqueia o registro durante a alteração.

### Conhecimento

- `app/services/knowledge_service.py` — pesquisa FAQs aprovadas e documentos da base de conhecimento da empresa, com geração de trechos; `KnowledgeService` preserva compatibilidade.
- `app/services/knowledge/__init__.py` — reexporta os casos de uso de pesquisa em FAQ e documentos.
- `app/services/knowledge/search_faq_service.py` — caso de uso fino para pesquisar FAQs aprovadas da empresa via fachada legada.
- `app/services/knowledge/search_knowledge_service.py` — caso de uso fino para pesquisar documentos aprovados e configurar tamanho do trecho retornado.

### Notificações e auditoria

- `app/services/notification_service.py` — cria notificações para membros ativos ou para toda a empresa com suporte a idempotência, e grava registros de auditoria; `NotificationService` é alias compatível.
- `app/services/notifications/__init__.py` — reexporta casos de uso de notificação e auditoria.
- `app/services/notifications/create_audit_log_service.py` — caso de uso fino que registra ação, entidade, ator, alterações, contexto e dados de requisição na auditoria.
- `app/services/notifications/create_notification_service.py` — caso de uso fino que cria notificação por empresa/usuário com dados, link e chave de idempotência.

### Caixa de saída

- `app/services/outbox_service.py` — persiste intenções de tarefas Celery permitidas, valida payloads e idempotência, despacha entradas e recupera pendências com retentativa; `OutboxService` é alias de compatibilidade.
- `app/services/outbox/__init__.py` — reexporta casos de uso para enfileirar e despachar entradas da caixa de saída.
- `app/services/outbox/dispatch_outbox_best_effort_service.py` — tenta despachar imediatamente uma entrada, tratando falhas como recuperáveis pelo processamento periódico.
- `app/services/outbox/dispatch_outbox_entry_service.py` — solicita o despacho de uma entrada específica da caixa de saída.
- `app/services/outbox/dispatch_pending_outbox_service.py` — despacha um lote limitado de itens pendentes e retorna contagens do lote.
- `app/services/outbox/enqueue_outbox_task_service.py` — registra intenção de tarefa autorizada com payload, escopo opcional da empresa e chave obrigatória de idempotência.

### Cotas e isolamento multiempresa

- `app/services/quota_service.py` — aplica cotas transacionais para mensagens de entrada por empresa/hora e remetente/minuto, deduplica mensagens e reserva capacidade sob bloqueio; `QuotaService` é alias compatível.
- `app/services/quota/__init__.py` — reexporta o caso de uso de aplicação da cota de entrada.
- `app/services/quota/enforce_inbound_quota_service.py` — caso de uso fino para verificar duplicidade e aplicar a política de cota a uma mensagem recebida.
- `app/services/tenancy.py` — centraliza resolução da empresa do usuário, autorização por papel, busca de registros por tenant e validação de relações entre empresas.
- `app/services/exceptions.py` — define erros de domínio tipados e seus códigos/status HTTP, incluindo validação, não encontrado, conflito, limite, autorização, tenant e serviço externo.

### WhatsApp

- `app/services/whatsapp_service.py` — gateway de integração com Evolution API: valida webhook/eventos, processa conexão e mensagens recebidas em transação, envia texto e administra QR code, instância, webhook e desconexão; `WhatsAppService` é alias compatível.
- `app/services/whatsapp/__init__.py` — reexporta os casos de uso de conexão, QR code, desconexão, recebimento e envio de mensagens.
- `app/services/whatsapp/check_whatsapp_connection_service.py` — caso de uso injetável que consulta a conexão do WhatsApp por meio do gateway Evolution.
- `app/services/whatsapp/disconnect_whatsapp_service.py` — caso de uso injetável que desconecta a integração da empresa via gateway.
- `app/services/whatsapp/generate_whatsapp_qrcode_service.py` — valida e coordena a geração/retorno do QR code da integração, delegando chamadas externas ao gateway.
- `app/services/whatsapp/process_whatsapp_payload_service.py` — caso de uso injetável para validar/processar payload de webhook e eventos permitidos do WhatsApp.
- `app/services/whatsapp/send_whatsapp_text_service.py` — caso de uso injetável para enviar texto pela integração conectada, com suporte a resposta citada.

## Interface, página pública e recursos visuais

### Templates e arquivos estáticos do Flask

`frontend/README.md` resume o padrão server-rendered e identifica templates, CSS e JS estáticos do app. Também registra que o QR Code de WhatsApp é requisitado por `fetch` e não aparece no HTML inicial.

`app/templates/base.html` é a moldura compartilhada do painel autenticado: navegação, cabeçalho, token CSRF, tema, slots de conteúdo e inclusão do CSS/JS globais.

`app/templates/components/ui.html` fornece macros Jinja para estado vazio, paginação e estrutura de modal, reutilizados pelas páginas do painel.

`app/templates/dashboard/index.html` apresenta a visão geral do painel e seus indicadores.

`app/templates/appointments/index.html` lista e administra agendamentos. `app/templates/appointments/professionals.html` gerencia profissionais associados à agenda.

`app/templates/conversations/index.html` é a interface de conversas; `app/templates/conversations/detail.html` estende essa interface para a conversa individual.

`app/templates/customers/index.html` lista clientes; `app/templates/customers/detail.html` mostra o detalhe individual; `app/templates/customers/crm.html` apresenta a visão de CRM/relacionamento.

`app/templates/delivery/index.html` organiza entregas. `app/templates/faq/index.html` administra perguntas frequentes. `app/templates/inventory/index.html` exibe e controla estoque. `app/templates/products/index.html` administra produtos.

`app/templates/settings/ai.html` concentra configurações de IA. `app/templates/settings/whatsapp.html` configura a integração de WhatsApp e a obtenção controlada do QR Code.

`app/templates/auth/base_auth.html` é a base visual das páginas de autenticação. `app/templates/auth/login.html` é o login Flask; `app/templates/auth/register.html` é o cadastro; `app/templates/auth/forgot_password.html` inicia recuperação de senha; `app/templates/auth/reset_password.html` define nova senha com token. `app/templates/auth/change_password.html` é a troca de senha dentro do painel e estende a base principal.

`app/templates/auth/forgot.html` estende `forgot_password.html`, e `app/templates/auth/reset.html` estende `reset_password.html`; são aliases/templates muito pequenos, não implementações independentes.

`app/templates/errors/base_error.html` é a base das páginas de erro. `app/templates/errors/400.html`, `app/templates/errors/403.html`, `app/templates/errors/404.html`, `app/templates/errors/429.html` e `app/templates/errors/500.html` especializam, respectivamente, requisição inválida, acesso não autorizado, página inexistente, excesso de requisições e falha interna.

`app/static/css/app.css` contém estilos compartilhados da aplicação Flask. `app/static/js/app.js` contém as interações comuns do painel e dos formulários: chamadas autenticadas, menus laterais, modais, busca, validação e componentes de tabela/formulário. São distintos do CSS e do JS do site estático.

### Site estático, rotas legadas e dependências

`index.html` é a landing page em português, com âncoras de recursos/demo/contato, canvas de introdução e demonstração de telefone. Importa Three.js 0.169 via import map/CDN e carrega `js/intro-scene.js` e `js/phone-scroll.js`. No app Flask, `/` entrega esse arquivo para visitante anônimo e direciona usuário autenticado ao painel.

`login.html` é uma tela de login estática legada com a marca e links de retorno à landing. Não é o template Flask de autenticação: `/login.html` redireciona para a rota Flask de login no `app/__init__.py`; o Express atende o fluxo estático/API legado.

`dashboard.html` é o painel demonstrativo estático legado, com múltiplas telas/seções no mesmo documento. O Express protege `/dashboard.html` com sessão; o alias Flask `/dashboard.html` redireciona para o painel server-rendered em `app/templates/dashboard/index.html`.

`dashbord.html` (grafia com erro) é um HTML mínimo em inglês, apenas com um título/texto “Dashboard”. Não há referências a ele no código consultado; não é o alvo da rota `/dashboard.html`.

`script.js` implementa interações do site legado e do painel estático: navegação por hash, formulários e chamadas a `/api/...`, incluindo login/cadastro, leitura de sessão, logout e componentes da demonstração. Os links de redirecionamento para `dashboard.html`/`login.html` pertencem a esse fluxo.

`style.css` é a folha extensa de estilos da landing, autenticação e painel legado. `server.js` é a aplicação Express correspondente: expõe API, valida sessão para `dashboard.html`, publica arquivos da raiz via `express.static` e usa `index.html` como fallback. Isso difere da aplicação Flask descrita acima.

`package.json` define o pacote privado `autoflow-site`, os comandos `start`/`dev` (ambos `node server.js`) e `create-user`, e dependências `express`, `mysql2`, `bcryptjs` e `dotenv`. `package-lock.json` fixa a árvore de dependências npm (lockfile v3) e acompanha alterações no manifesto. O comando `create-user` aponta para `scripts/create-user.js`, ausente no checkout.

`vercel.json` configura funções Node para `server.js`, publicação estática de `index.html`, CSS, JS, `js/**` e `assets/**`, roteia API e páginas antigas ao servidor e aplica fallback de filesystem/raiz. Esse arquivo descreve a publicação da versão estática com Express.

### Animação e cena 3D

`js/intro-scene.js` monta a cena 3D de abertura no canvas `#intro-canvas`. `js/phone-scene.js` cria a cena Three.js do aparelho, carrega o GLTF e atualiza a pose com progresso de rolagem. `js/phone-screen-texture.js` desenha em canvas a conversa/WhatsApp aplicada à tela do aparelho. `js/phone-scroll.js` conecta a seção `#demo` e seus passos ao progresso do scroll e à cena do telefone, respeitando movimento reduzido.

`intro-scene.js` e `phone-scene.js` na raiz têm conteúdo byte a byte idêntico às cópias `js/intro-scene.js` e `js/phone-scene.js` (SHA-256 conferido). A página ativa carrega os módulos sob `js/`; as cópias da raiz não aparecem nos scripts do HTML. Além disso, o `phone-scene.js` da raiz importa `./phone-screen-texture.js`, que não existe na raiz, então essa cópia não é autocontida para execução nesse local.

### Imagens e modelo de telefone

`assets/autoflow-mark-transparent.png` é o logo transparente referenciado pelo HTML público. `assets/ai-automation-core-dashboard.png` é a imagem de ilustração do fluxo de IA/automação usada na landing. `assets/autoflow-mark-source.png` e `assets/autoflow-mark-chroma.png` são variantes-fonte/cromática da marca; não encontrei referência direta a elas nos HTML/JS examinados.

- `assets/iphone-16/1b338ec19f15ad72904b (1).gltf` — descreve a cena, as malhas e os materiais do iPhone carregado por js/phone-scene.js. O carregador remove referências a imagens e texturas KTX do JSON antes de interpretar a cena.
- `iphone-16-inspect/model/1b338ec19f15ad72904b (1).gltf` — cópia idêntica da descrição glTF, na pasta de inspeção.
- `iphone-16-inspect/source/1b338ec19f15ad72904b (1).zip` — arquivo compactado de origem associado à cópia de inspeção. O conteúdo interno do ZIP ficou fora desta leitura.
- `assets/iphone-16/58110435e09ab71a2a8e.bin` — arquivo binário que acompanha o modelo. O glTF rastreado não aponta para este nome na seção buffers.
- `iphone-16-inspect/model/58110435e09ab71a2a8e.bin` — cópia idêntica do binário homônimo em assets/iphone-16, mantida com o modelo de inspeção.
- `assets/iphone-16/6996ea824c04c36ad72c.bin` — buffer binário referenciado pela seção buffers do glTF. Contém os dados usados pelas malhas do modelo.
- `iphone-16-inspect/model/6996ea824c04c36ad72c.bin` — cópia idêntica do binário homônimo em assets/iphone-16, mantida com o modelo de inspeção.
- `assets/iphone-16/70c122eddaa2a99f9c90.bin` — arquivo binário que acompanha o modelo. O glTF rastreado não aponta para este nome na seção buffers.
- `iphone-16-inspect/model/70c122eddaa2a99f9c90.bin` — cópia idêntica do binário homônimo em assets/iphone-16, mantida com o modelo de inspeção.
- `assets/iphone-16/af59ff3f676b9b756a02.bin` — arquivo binário que acompanha o modelo. O glTF rastreado não aponta para este nome na seção buffers.
- `iphone-16-inspect/model/af59ff3f676b9b756a02.bin` — cópia idêntica do binário homônimo em assets/iphone-16, mantida com o modelo de inspeção.
- `assets/iphone-16/fe1bc97f6fe23cdadab2.bin` — arquivo binário que acompanha o modelo. O glTF rastreado não aponta para este nome na seção buffers.
- `iphone-16-inspect/model/fe1bc97f6fe23cdadab2.bin` — cópia idêntica do binário homônimo em assets/iphone-16, mantida com o modelo de inspeção.

## Inicialização, configuração e infraestrutura

| Arquivo | Função |
| --- | --- |
| `run.py` | Cria a aplicação Flask pela factory, expõe `run:app` para o Gunicorn e inicia o servidor Flask na porta 5000 quando executado diretamente. |
| `config.py` | Reexporta as classes e o mapa de ambientes de `app/config.py` para manter imports antigos a partir da raiz. |
| `celery_worker.py` | Cria o contexto Flask para o processo Celery, expõe a instância `celery_worker.celery` e carrega as tarefas. É a entrada usada pelo worker e pelo beat. |
| `healthcheck.py` | Consulta `/health` pelo loopback, com o cabeçalho HTTP Host derivado de `APP_URL`. Assim, a checagem usa o endereço interno e apresenta ao Flask o host público configurado. Falhas são propagadas ao processo que executa a checagem. |
| `requirements.txt` | Define as faixas de versões das dependências Python. Inclui Flask, SQLAlchemy, Alembic via Flask-Migrate, MySQL via PyMySQL, Celery, Redis, utilitários de validação, parsers de documentos, Gunicorn e ferramentas de teste. |
| `.env.example` | Modelo de variáveis para a aplicação, com opções de banco, Redis, Groq, Evolution, SMTP, sessão, URLs, cotas, uploads e outbox. |
| `.env.evolution.example` | Modelo de configuração próprio da Evolution, com URL, autenticação e conexões para seu PostgreSQL e Redis. |
| `.gitignore` | Exclui do versionamento arquivos de ambiente locais, ambientes virtuais, caches, logs, dados de execução e artefatos de trabalho. |
| `.dockerignore` | Reduz o contexto de build excluindo arquivos privados, caches, testes, dados locais e outros itens dispensáveis para montar a imagem. |
| `.gitattributes` | Define final de linha LF para scripts shell, que são executados no Linux. |
| `Dockerfile` | Monta a imagem Python 3.12 slim, instala dependências, copia a aplicação e a landing, prepara o diretório de uploads e inicia o Gunicorn como usuário sem privilégios. Declara o probe em `healthcheck.py`. |
| `docker-compose.yml` | Define os serviços principais, MySQL, Redis, web, worker e beat. O web aplica migrações antes do Gunicorn. Volumes preservam banco, Redis e imagens enviadas ao painel. |
| `docker-compose.evolution.yml` | Define a Evolution API e seu PostgreSQL e Redis dedicados. Pode complementar a composição principal e fornece a instância intermediária entre o AutoFlow e o WhatsApp. |
| `docker-compose.vm.yml` | Sobrepõe a configuração base para a VM. Reduz concorrência e memória, amplia o tempo de inicialização do probe, faz worker e beat aguardarem o web saudável e publica o tráfego pelo Caddy. |
| `Caddyfile` | Configura o domínio da VM, compressão e proxy reverso para o serviço interno `web:5000`. |
| `.github/workflows/deploy-vm.yml` | No checkout atual, executa o Pytest em Ubuntu com Python 3.12 em pushes para `main` ou por disparo manual. O nome do arquivo é histórico e o conteúdo atual contém apenas o job de testes. |
| `scripts/deploy-vm.sh` | Valida os arquivos de ambiente e a composição, constrói uma imagem para web, worker e beat, sobe os serviços e checa Evolution e web. Registra diagnóstico em falhas e tenta restaurar a imagem anterior quando disponível. |
| `scripts/receive-deploy-vm.sh` | Recebe um pacote de revisão pela entrada padrão do comando SSH, bloqueia implantações concorrentes, extrai para staging e sincroniza o código preservando configurações locais. Atualiza a cópia instalada do receptor e registra o log da implantação por revisão. |

## Banco e migrações

| Arquivo | Função |
| --- | --- |
| `database.sql` | DDL histórico de um esquema simples, com as tabelas `usuarios` e `sessoes_login`. O provisionamento do schema do SaaS no Compose passa pelo Alembic. |
| `migrations/README` | Nota introdutória sobre a configuração do diretório de migrações. |
| `migrations/alembic.ini` | Configura o Alembic e seus loggers, handlers e formatação de mensagens. |
| `migrations/env.py` | Integra o Alembic ao engine e aos metadados SQLAlchemy da aplicação Flask. Define a execução online com conexão ao banco e offline para geração de SQL. |
| `migrations/script.py.mako` | Template usado pelo Alembic para gerar um novo arquivo de revisão com metadados e funções `upgrade()` e `downgrade()`. |
| `migrations/versions/6973a8314ce3_initial_autoflow_schema.py` | Cria o schema inicial multiempresa, incluindo entidades, relações, índices e restrições para atendimento, clientes, catálogo, estoque, agenda, auditoria e outros domínios. O downgrade remove a estrutura em ordem de dependência. |
| `migrations/versions/b73d10e6a842_evolution_api_integration.py` | Migra a tabela de integrações do WhatsApp dos identificadores Meta para instâncias Evolution. Copia dados para os novos campos, garante nome de instância único e atualiza as colunas e os índices. |

## Testes

Os arquivos abaixo descrevem a cobertura escrita na suíte. As fixtures compartilham uma aplicação de teste e permitem simular dependências externas.

| Arquivo | O que verifica |
| --- | --- |
| `pytest.ini` | Limita a descoberta ao diretório `tests`, exige marcadores registrados e converte avisos em erros, com exceção dos de depreciação. |
| `tests/__init__.py` | Marca `tests` como pacote Python. |
| `tests/conftest.py` | Monta a aplicação de teste, inicializa e limpa a base, fornece cliente HTTP e runner de CLI e oferece factories de empresas, usuários e helpers de login. |
| `tests/test_ai_handoff.py` | Verifica a transferência da IA para o atendimento humano, incluindo concorrência com a tomada da conversa, validade da confirmação pendente e rollback em falha do outbox. |
| `tests/test_ai_summary.py` | Valida o cursor dos resumos, divisão de mensagens grandes, espera por mensagens pendentes, retry após truncamento e descarte de resultado calculado sobre cursor desatualizado. |
| `tests/test_ai_tools.py` | Verifica o registro explícito de ferramentas, o escopo por empresa, o tratamento de argumentos inválidos e o contrato das ferramentas de estoque e agenda. |
| `tests/test_appointments.py` | Exercita disponibilidade, fuso horário da empresa, regras de horários, bloqueios, conflitos, idempotência, cancelamento e ordenação dos horários disponíveis. |
| `tests/test_auth.py` | Cobre cadastro, hash de senha, login com normalização de e-mail, proteção de páginas, logout e sigilo da existência da conta na recuperação de senha. |
| `tests/test_controller_structure.py` | Valida que os Blueprints apontam para controllers, que módulos de rota se limitam ao registro e que a aplicação usa um único pacote de backend. |
| `tests/test_conversations.py` | Verifica histórico para a IA, tomada e devolução do atendimento, persistência antes do envio assíncrono e atualização da lista ao mudar outra conversa. |
| `tests/test_customer_inventory_use_case_services.py` | Exercita diretamente os serviços de arquivamento de cliente, histórico de estoque e alteração do mínimo, com isolamento, ordenação e controle transacional. |
| `tests/test_customers.py` | Verifica normalização e duplicidade de telefones, movimentação pelas etapas do CRM e persistência e renderização dos dados de oportunidade. |
| `tests/test_groq_service.py` | Exercita a montagem da requisição Groq, personalização pelo nome do contato, contexto de agenda e execução do ciclo de chamadas de ferramentas. |
| `tests/test_healthcheck.py` | Confere URL de loopback, host configurado, fallback local, propagação de falhas HTTP e integração com a validação de hosts do Flask. |
| `tests/test_inventory.py` | Verifica histórico de movimentação e saldo, bloqueio de saldo negativo e deduplicação de movimentações. |
| `tests/test_knowledge_search.py` | Verifica a geração de trechos próximos ao termo encontrado no conteúdo, com uso do início do texto quando a correspondência ocorre só no título. |
| `tests/test_landing.py` | Confere a landing e seus assets via Flask, os links de autenticação, a política CSP do import map, os redirects legados e o destino da raiz autenticada. |
| `tests/test_model_crud.py` | Verifica a presença do contrato CRUD nos modelos e o comportamento dos aliases de persistência, atualização, remoção e flush sem commit. |
| `tests/test_multitenancy.py` | Cobre filtros por empresa em modelos e endpoints e a resolução da empresa pelo contexto autenticado. |
| `tests/test_outbox.py` | Simula falha do broker e recuperação pelo beat. Também verifica o registro de tarefas, a validação de payload e a idempotência do outbox. |
| `tests/test_product_use_case_services.py` | Exercita diretamente os casos de uso do catálogo e verifica o isolamento da listagem entre empresas. |
| `tests/test_products.py` | Cobre criação de produto com estoque inicial, atualização, arquivamento, permissões e exclusão de itens inativos das consultas feitas pela IA. |
| `tests/test_professionals.py` | Cobre cadastro, edição e remoção ou arquivamento de profissionais, vínculos com vários serviços, duplicatas, permissões e isolamento por empresa. |
| `tests/test_security_regressions.py` | Verifica invalidação de sessão, isolamento de segredos de integração, permissões de atendimento e rejeição de ambiente de execução desconhecido. |
| `tests/test_service_structure.py` | Confere a organização dos casos de uso em módulos, o contrato `execute()` e o papel dos módulos de compatibilidade. |
| `tests/test_tasks.py` | Verifica processamento do histórico e envio pelo WhatsApp, extração de ID da Evolution, transições de estado e reconciliação de envios incertos. |
| `tests/test_upload_security.py` | Cobre validação e limites de documentos e imagens, cotas por empresa, limpeza de arquivos temporários e publicação de imagem associada ao commit. |
| `tests/test_webhook_quota.py` | Confere cotas por empresa e remetente, tratamento de duplicatas e aceitação da primeira mensagem autenticada de instância pendente. |
| `tests/test_whatsapp_qrcode.py` | Verifica escopo e permissão da tela de QR Code, criação de instância, sincronização de webhook e normalização da imagem do código. |
| `tests/test_whatsapp_webhook.py` | Cobre autenticação de eventos, localização da instância, atualização de conexão, persistência e fila idempotentes e filtros de mensagens de grupo ou enviadas pela própria conta. |

## Documentação e pacote histórico de implantação

| Arquivo | Função |
| --- | --- |
| `README.md` | Documenta arquitetura, instalação, variáveis, integrações, migrações, execução, testes e deploy. A nota inicial sobre a landing externa está desatualizada em relação ao código e aos testes da landing integrada. |
| `docs/ARQUITETURA.md` | Define responsabilidades de rotas, controllers, serviços e modelos, com convenções para CRUD, isolamento por empresa e transações. |
| `LEIA-ME.txt` | Relato histórico da correção do healthcheck em uma VM, com diagnóstico, revisão-base, instruções para aplicar o patch e limites da validação feita naquela ocasião. |
| `AutoFlow-correcao-deploy.patch` | Diff distribuível com mudanças no Dockerfile, na checagem HTTP, no diagnóstico do deploy e nos testes associados. |
| `AutoFlow-correcao-deploy/AutoFlow-correcao-deploy.patch` | Cópia do patch incluída dentro do pacote de correção. |
| `AutoFlow-correcao-deploy/Dockerfile` | Dockerfile da revisão do pacote histórico, com a checagem HTTP baseada em script Python. |
| `AutoFlow-correcao-deploy/LEIA-ME.txt` | Instruções e contexto que acompanham o pacote de correção. |
| `AutoFlow-correcao-deploy/healthcheck.py` | Versão da checagem de saúde entregue no pacote, com loopback e Host derivado da URL configurada. |
| `AutoFlow-correcao-deploy/scripts/deploy-vm.sh` | Versão do script de deploy entregue com a correção, incluindo coleta do estado do contêiner web em falhas. |
| `AutoFlow-correcao-deploy/tests/test_healthcheck.py` | Versão dos testes da checagem HTTP incluída no pacote para validação da correção. |
