# Ativar backup via Supabase Storage — AURA One

O bucket `aura-private-backups` foi criado como **privado**, limite por objeto de **50 MiB**, MIME `application/octet-stream`. Não há políticas de acesso para `storage.objects`; portanto, clientes comuns não devem conseguir listar/ler/gravar os arquivos. A chave privilegiada de servidor contorna RLS: proteja-a rigorosamente.

O workflow `.github/workflows/backup-supabase-storage.yml` agenda um `pg_dump` PostgreSQL 17 diário às **12:30 UTC**, criptografa com a **chave pública age** e envia o arquivo criptografado para o bucket.

## Configuração necessária no GitHub

Em **Settings > Secrets and variables > Actions**:

**Secrets:**
- `AURA_BACKUP_DATABASE_URL`: conexão PostgreSQL com TLS e permissão de exportação.
- `AURA_BACKUP_SUPABASE_SERVICE_KEY`: chave de servidor do Supabase, **nunca** a publishable/anon key; tratar como segredo de altíssimo privilégio.

**Variables:**
- `AURA_BACKUP_SUPABASE_URL`: `https://blmwvwutkzyugvxvmyfx.supabase.co`.
- `AURA_BACKUP_AGE_RECIPIENT`: chave pública age, prefixo `age1`.

Gere o par de chaves age **em ambiente seguro fora do GitHub**. Guarde a chave privada em cofre protegido e com cópia de recuperação. **Não cole credenciais nem a chave privada neste chat.** Restrinja quem pode editar workflows, branches e segredos, pois código executado pelo GitHub Actions pode acessar a chave de servidor.

## Validação

1. Configure os quatro valores acima.
2. Em **Actions > AURA One - Backup criptografado Supabase Storage**, execute **Run workflow**.
3. Verifique sucesso do job e a presença do objeto `.dump.age` no bucket privado.
4. Baixe-o com acesso administrativo autorizado, descriptografe somente em ambiente isolado e faça `pg_restore` em um banco de ensaio. Confira RLS, dados, gatilhos, funções e isolamento entre clínicas.
5. Configure alertas de falha e retenção/exclusão de objetos antigos. Até essa etapa, **backup e restauração não estão homologados**.

## Limitações e riscos

- **Mesmo projeto:** exclusão, suspensão ou perda do projeto pode afetar simultaneamente banco e bucket; mantenha uma segunda cópia externa para recuperação de desastre.
- **Não cobre tudo:** Auth, arquivos existentes no Storage, configurações e segredos requerem proteção própria.
- **Limite de 50 MiB por arquivo** nesta configuração. Se ultrapassar, o job falha e não haverá backup daquele dia.
- **Custo/quotas:** armazenamento, transferência e execução têm limites do plano gratuito; monitore consumo.
- **Segredos:** nunca colocar `service_role`, string de conexão ou chave privada age em frontend, código versionado ou logs.
- **Restauração:** nunca sobrescrever produção sem aprovação explícita e ensaio.
