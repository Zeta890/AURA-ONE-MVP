# Ativação do backup automático — AURA One

O workflow `.github/workflows/backup-database.yml` está agendado para **12:30 UTC diariamente**, mas **não produz backups até configurar as credenciais e o destino**. Em execução agendada sem configuração, registra aviso e encerra sem criar backup; execução manual sem configuração falha.

## Pré-requisitos

1. Um bucket S3 **privado**, com bloqueio de acesso público, criptografia em repouso, retenção, versionamento e política de acesso mínimo. Configure lifecycle/retention conforme LGPD e política aprovada.
2. Um usuário/role AWS dedicado, autorizado somente a `s3:PutObject` no prefixo `aura-one/*` do bucket de backup (e, se necessário, permissões específicas para políticas de criptografia). Não conceder leitura pública.
3. Um par de chaves **age** criado e guardado fora do GitHub. A chave **privada** deve ficar em cofre seguro, com cópia de recuperação separada. Só a chave pública `age1...` será cadastrada como variável.
4. Uma string de conexão PostgreSQL com permissão de leitura suficiente para exportar os objetos necessários. Evitar credenciais de superusuário quando possível; não armazenar em código.

## Configurar GitHub

Em **Settings > Secrets and variables > Actions** do repositório, criar os seguintes **Repository secrets**:

- `AURA_BACKUP_DATABASE_URL`: URL de conexão do PostgreSQL (com TLS, quando suportado).
- `AURA_BACKUP_AWS_ACCESS_KEY_ID`
- `AURA_BACKUP_AWS_SECRET_ACCESS_KEY`
- `AURA_BACKUP_AWS_REGION`
- `AURA_BACKUP_BUCKET`

Criar uma **Repository variable**:

- `AURA_BACKUP_AGE_RECIPIENT`: chave **pública** age (começa por `age1`).

**Nunca enviar esses valores no chat, em issues ou em commits.** Não colocar a chave privada age em GitHub Actions.

## Primeira execução e homologação

1. Em **Actions > AURA One - Backup criptografado > Run workflow**, iniciar manualmente.
2. Verificar se a execução terminou com sucesso e se os dois arquivos `.dump.age` e `.sha256` estão no bucket privado.
3. Conferir o SHA-256 após baixar o objeto criptografado para ambiente de ensaio.
4. Descriptografar **apenas em ambiente isolado** com a chave privada age e restaurar no banco de teste; validar tabelas, RLS, triggers, funções e isolamento de clínicas.
5. Registrar data, responsável, tamanho, RTO/RPO e resultado. **Somente depois marcar backup e restauração como validados.**

## Limitações

- Este workflow usa `pg_dump` do PostgreSQL 17. **Não cobre automaticamente** objetos do Supabase Storage, configurações Auth externas, chaves, segredos ou outros serviços.
- O dump pode conter dados pessoais. Ele passa pelo pipeline diretamente para a criptografia age, sem arquivo intermediário desprotegido; o runner ainda processa esses dados em memória.
- Uma falha no agendamento ou nas credenciais pode impedir o backup; configurar alertas de falha e verificar o bucket regularmente.
- A retenção, custos e permissões do S3 são responsabilidade da configuração da conta de armazenamento.
- O primeiro backup **não foi executado** neste commit; as credenciais não estão configuradas por esta integração.
