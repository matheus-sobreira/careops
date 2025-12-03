# Análise de Riscos OWASP Top 10 – CareOps+

Este documento apresenta uma análise preliminar de riscos da aplicação CareOps+ à luz do **OWASP Top 10:2025** (A01–A10), considerando o contexto de uma aplicação de saúde que expõe funcionalidades via API (ex.: `/patients`, `/patient/{id}`, `/prescriptions`, `/echo`, `/sum`, `/debug/config`, portal do paciente, etc.). O foco é conectar vulnerabilidades técnicas a impactos de negócio em saúde (confidencialidade, integridade e disponibilidade de dados clínicos e segurança do paciente).

---

## A01: Broken Access Control (Controles de Acesso Quebrados)

Uma vulnerabilidade típica aqui seria permitir que um paciente autenticado acesse registros de outros pacientes por meio de um endpoint como `GET /patient/{id}` sem validação adequada de autorização (IDOR – Insecure Direct Object Reference). Tecnicamente, isso ocorre quando a API confia apenas no token de sessão, mas não checa se o usuário logado é o titular daquele prontuário, ou quando filtros de escopo/role são mal implementados.

No contexto de saúde, o impacto de negócio é crítico: exposição de diagnósticos, histórico clínico, exames, dados pessoais sensíveis (LGPD – dados sensíveis de saúde), risco de dano à reputação da instituição e possíveis sanções regulatórias. Em um cenário CareOps+, um simples erro de autorização no `GET /patient/{id}` poderia permitir que um atacante enumere prontuários, baixe lotes de dados de pacientes e até use essas informações para fraude, chantagem ou discriminação.

---

## A02: Security Misconfiguration (Misconfiguração de Segurança)

Uma vulnerabilidade comum é manter um endpoint de debug habilitado em produção, como `/debug/config`, retornando variáveis de ambiente, strings de conexão com banco de dados ou chaves de API. Outras formas de misconfiguração incluem deixar CORS excessivamente permissivo, mensagens de erro detalhadas com stack trace ou serviços administrativos expostos sem autenticação forte.

Tecnicamente, misconfigurações ampliam a superfície de ataque e frequentemente funcionam como “atalhos” para exploração de outras falhas: um endpoint de debug pode revelar credenciais reutilizadas, caminhos internos, versões de bibliotecas vulneráveis, etc. Em uma aplicação de saúde, isso pode significar acesso indireto à base de prontuários, à infraestrutura de imagens médicas ou ao serviço de prescrições eletrônicas, com impacto direto na confidencialidade dos dados e possibilidade de paralisação de serviços críticos se o atacante escalar o ataque.

---

## A03: Software Supply Chain Failures (Falhas na Cadeia de Suprimentos de Software)

Nesta categoria entram riscos ligados a dependências de terceiros, bibliotecas e ferramentas usadas na construção e entrega do CareOps+. Um exemplo típico é utilizar uma biblioteca de ORM, framework web ou componente de autenticação com vulnerabilidades conhecidas, ou até confiar em imagens de container de fontes não confiáveis e não verificar integridade/assinaturas.

Tecnicamente, uma falha na cadeia de suprimentos pode permitir que código malicioso entre no sistema mesmo sem “erro” aparente dos desenvolvedores da CareOps+: um pacote comprometido pode incluir backdoors, exfiltração de dados ou bypass de autenticação. Para uma aplicação de saúde, isso significa que a confiança no prontuário eletrônico e nos fluxos de prescrição pode ser comprometida “pela base”, possibilitando alteração silenciosa de dados clínicos ou indisponibilidade do sistema em larga escala, com impacto direto na segurança do paciente e continuidade assistencial.

---

## A04: Cryptographic Failures (Falhas Criptográficas)

Aqui entram cenários como transmissão de dados de pacientes sem HTTPS adequado, uso de algoritmos obsoletos (ex.: SHA-1, RC4), armazenamento de senhas sem hash forte (ex.: em texto puro ou `MD5` simples) e ausência de criptografia em repouso para campos sensíveis, como diagnósticos, laudos e dados pessoais.

Do ponto de vista técnico, falhas criptográficas permitem que um atacante intercepte, leia ou modifique dados em trânsito (ataques man-in-the-middle) ou recupere senhas/segredos a partir de bancos de dados comprometidos. Em uma aplicação de saúde, isso pode expor massa de dados clínicos, credenciais de profissionais, chaves de integração com laboratórios e convênios, causando danos financeiros, perda de confiança, violações de LGPD e, em casos extremos, manipulação de resultados de exames ou prescrições.

No CareOps+, um exemplo seria o backend aceitar conexões HTTP simples para o portal do paciente e para a API `/patients`, permitindo que alguém em uma mesma rede capture pacotes e leia resultados de exames e diagnósticos sem qualquer esforço de quebra de criptografia.

---

## A05: Injection (Injeção)

Injeção SQL em um endpoint de busca de pacientes (por exemplo, `GET /patients?name=...`) é um exemplo clássico: parâmetros não sanitizados são concatenados diretamente na consulta ao banco, permitindo que um atacante liste todos os pacientes, altere ou apague registros. Outras variantes incluem injeção de comando em scripts de integração, injeção LDAP, NoSQL ou até injeção de expressão em templates.

Tecnicamente, a injeção resulta em execução de comandos arbitrários pelo banco de dados ou outro interpretador, rompendo completamente o modelo de segurança planejado. Na área de saúde, isso significa que um atacante pode apagar prontuários, alterar medicamentos registrados, inserir resultados de exame falsos ou indisponibilizar a base de dados via `DROP TABLE`, afetando diretamente a continuidade do cuidado e potencialmente colocando vidas em risco.

No contexto CareOps+, um atacante poderia usar um parâmetro mal validado em `/patients` para vazar toda a tabela `patients` e `prescriptions`, ou então modificar silenciosamente a prescrição de um paciente de um medicamento para outro, sem que o médico ou o paciente percebam de imediato.

---

## A06: Insecure Design (Design Inseguro)

Insecure Design foca na ausência de requisitos de segurança e de uma arquitetura pensada para mitigar ameaças, independentemente de bugs específicos de implementação. Exemplos incluem não prever limitação de tentativas de login, não definir claramente quais papéis podem acessar dados de pacientes, ou desenhar fluxos críticos (como assinatura de prescrições) sem etapas de verificação ou aprovação adequada.

Tecnicamente, isso se traduz em sistemas que “funcionam” do ponto de vista funcional, mas embutem decisões inseguras desde a concepção. Em uma aplicação de saúde, um design que não separa claramente perfis (médico, enfermeiro, administrativo, paciente) pode permitir que qualquer colaborador acesse dados que não deveria, ou realize ações (como encerrar internações ou alterar diagnósticos) sem trilhas de auditoria e revisões.

Na CareOps+, por exemplo, um desenho de API que expõe `/patients` e `/prescriptions` apenas com autenticação simples, sem escopos ou vínculo forte entre usuário e paciente, já nasce inseguro, mesmo que não haja nenhum “bug” óbvio no código.

---

## A07: Authentication Failures (Falhas de Autenticação)

Falhas de autenticação incluem senhas fracas sem política mínima, ausência de MFA para perfis sensíveis (como médicos e administradores), tokens de sessão sem expiração adequada, armazenamento inseguro de credenciais ou fluxos de login suscetíveis a brute force sem qualquer detecção.

Tecnicamente, isso facilita que um atacante assuma identidades legítimas — por exemplo, um médico ou administrador — e passe a operar “por dentro” do sistema, o que é especialmente perigoso em ambientes de saúde. Com credenciais comprometidas, é possível acessar a API de pacientes, registrar prescrições, alterar laudos ou exportar dados massivamente.

No CareOps+, um portal do paciente ou painel clínico sem MFA, combinado com senhas de baixa complexidade e ausência de travas de bloqueio após tentativas mal-sucedidas sucessivas, representaria um risco alto de comprometimento de contas e vazamento de dados clínicos.

---

## A08: Software or Data Integrity Failures (Falhas de Integridade de Software ou Dados)

Este risco aparece quando não há mecanismos para garantir a integridade de software (atualizações, imagens de container, artefatos de build) ou dos dados (logs clínicos, prescrições, registros de atendimento). Exemplos incluem pipeline de deploy sem assinaturas/verificação de integridade, backups que podem ser alterados sem detecção e ausência de checksums ou trilhas de auditoria confiáveis.

Tecnicamente, um atacante pode introduzir código malicioso no caminho de build/deploy (supply chain), ou alterar dados armazenados sem deixar rastros conf
