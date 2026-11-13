# AURA One — Continuidade, backup e recuperação (v0.33)

> Procedimento operacional. **Não é evidência de que um backup já exista ou de que a restauração foi testada.** Antes de receber dados reais de pacientes, executar e registrar um teste completo em ambiente isolado.

## Regras de proteção

- Nunca enviar backups, dados de pacientes, chaves TOTP, códigos de recuperação, URLs de conexão com senha, tokens ou arquivos `.env` para este repositório, issues, chats ou logs.
- Guardar backups criptografados em armazenamento privado com acesso mínimo; definir retenção e exclusão segura de acordo com a política de privacidade/LGPD.
- Um `pg_dump` do PostgreSQL **não equivale a um backup completo do Supabase Auth, Storage e configurações externas**. Planejar cada componente separadamente.
- Não restaurar diretamente no banco de produção sem aprovação e janela de manutenção. Restaurações sobrescrevem ou duplicam dados se executadas incorretamente.

## 1. Backup manual do PostgreSQL (sem Supabase Pro)

**Pré-requisitos:** PostgreSQL client `pg_dump`/`pg_restore` compatível com a versão do servidor; acesso autorizado à conexão do banco; computador confiável com espaço suficiente.

1. No painel do Supabase, em **Project Settings > Database**, obter a string de conexão apropriada (preferir conexão direta IPv6 se suportada ou pooler de sessão). **Não colar a string em arquivos versionados ou em mensagens.**
2. No terminal local, fornecer a conexão por variável de ambiente protegida ou gerenciador de segredos, sem gravar senha no histórico do shell.
3. Executar:

```bash
# DB_URL deve estar configurada apenas no ambiente seguro da máquina.
# O arquivo produzido contém dados pessoais e NÃO deve ir para o GitHub.
pg_dump --dbname="$DB_URL" --format=custom --no-owner --no-acl --file="aura-one-$(date +%Y%m%d).dump"
```

4. Criptografar imediatamente o arquivo usando ferramenta corporativa aprovada; enviar **somente o arquivo criptografado** a um cofre privado com controle de acesso. Apagar com segurança a cópia não criptografada conforme a política adotada.
5. Registrar data, tamanho, hash SHA-256 do arquivo criptografado, responsável e local do cofre **sem incluir credenciais nem dados de pacientes**.

**Atenção:** um dump padrão cobre objetos e dados acessíveis ao usuário da conexão. Extensões, papéis gerenciados, permissões, objetos do esquema `auth`, Storage e configurações da plataforma podem exigir estratégias distintas. Inspecionar o conteúdo e os erros do dump antes de considerar o backup válido.

## 2. Ensaio de restauração

1. Criar um projeto Supabase **separado e isolado**, sem integrações de envio ou automações externas habilitadas. Não usar dados reais sem base legal e controles equivalentes.
2. Usar a cópia criptografada, descriptografada apenas em ambiente protegido.
3. Restaurar em banco de ensaio, após avaliar objetos/roles existentes e extensões do projeto de destino:

```bash
# Exemplo ilustrativo: somente no banco isolado e preparado para restauração.
pg_restore --dbname="$RESTORE_DB_URL" --no-owner --no-acl --verbose "aura-one-YYYYMMDD.dump"
```

4. Conferir erros, tabelas, RLS, políticas, funções, gatilhos, agendamentos, permissões e contagens agregadas, sem expor dados pessoais.
5. Verificar login/MFA com contas de teste e isolamento entre duas clínicas. **Não usar WhatsApp real nem executar disparos.**
6. Documentar tempo de recuperação (RTO), perda máxima tolerável (RPO), resultado e pendências. Um arquivo de backup sem restauração ensaiada **não é recuperação validada**.

## 3. Recuperação de acesso MFA

- Antes de ativar MFA para todos os administradores, manter **duas contas administrativas individuais**, ambas protegidas com MFA, sem compartilhamento de senha ou autenticador.
- Se alguém perder o autenticador, verificar identidade por processo interno documentado, com aprovação de outro administrador autorizado e registro de auditoria.
- A recuperação ou remoção de um fator MFA pela administração do Supabase só deve ocorrer por meio de mecanismos oficiais e permissões apropriadas. Não criar função pública, endpoint com `service_role` no navegador, nem bypass de MFA.
- Após recuperação, invalidar sessões antigas quando possível, cadastrar novo fator e revisar os logs.
- Se não houver administrador alternativo ou processo de recuperação oficial testado, **o risco de bloqueio administrativo permanece aberto**.

## 4. Verificações diárias e semanais

- **Diariamente:** consultar o painel **Monitoramento operacional · AURA Connect**; conferir se o job está ativo, a última execução e eventuais erros.
- **Semanalmente:** confirmar que backups privados foram produzidos, criptografados e podem ser lidos; verificar espaço, retenção e permissões.
- **Mensalmente:** ensaiar uma restauração isolada e documentar os resultados.
- **Antes do piloto real:** testar expiração de sessão, MFA, RLS entre clínicas, exclusão/consentimento, backup e restauração.

## 5. Pendências conhecidas

- Backup automático e armazenamento privado: **não configurados**.
- Restauração isolada: **não testada**.
- Recuperação administrativa de MFA: **procedimento definido, mas não ensaiado**.
- Cron AURA Connect: agendado às 11:00 UTC; considerar confirmado apenas após registro de execução real.
- Backup de Supabase Auth, Storage e configurações externas: **avaliar separadamente**.
