# Plano de Rollback - Blue/Green Deployment RDS

Este documento descreve os procedimentos de rollback para o processo de otimização de storage usando Blue/Green Deployment no Amazon RDS.

---

## Visão Geral

O Blue/Green Deployment oferece múltiplas camadas de segurança para rollback, permitindo reverter a operação em diferentes estágios do processo. O ambiente Blue (original) permanece intacto até que você decida deletá-lo, garantindo um caminho seguro de retorno.

---

## Cenários de Rollback

### Cenário 1: Rollback ANTES do Switchover

**Situação**: Você criou o Blue/Green Deployment mas ainda não executou o switchover.

**Impacto**: Zero impacto na produção. O ambiente Blue continua operando normalmente.

**Procedimento**:

```bash
# 1. Deletar o Blue/Green Deployment sem afetar o ambiente Blue
aws rds delete-blue-green-deployment \
  --blue-green-deployment-identifier <NOME_DO_DEPLOYMENT> \
  --no-delete-target

# Exemplo:
aws rds delete-blue-green-deployment \
  --blue-green-deployment-identifier prod-storage-optimization \
  --no-delete-target
```

**Resultado**: O ambiente Green é deletado e o Blue continua operando normalmente.

**Tempo estimado**: 5-10 minutos

---

### Cenário 2: Rollback IMEDIATAMENTE APÓS o Switchover

**Situação**: O switchover foi concluído mas você identificou problemas críticos nas primeiras horas.

**Impacto**: Downtime de 1-2 minutos durante o switchover reverso.

**Pré-requisitos**:
- Ambiente Blue ainda existe (não foi deletado)
- Blue/Green Deployment ainda está ativo
- Tempo decorrido < 24 horas após switchover

**Procedimento**:

```bash
# 1. Verificar se o Blue/Green Deployment ainda existe
aws rds describe-blue-green-deployments \
  --blue-green-deployment-identifier <NOME_DO_DEPLOYMENT>

# 2. Executar switchover reverso (volta para o Blue)
aws rds switchover-blue-green-deployment \
  --blue-green-deployment-identifier <NOME_DO_DEPLOYMENT> \
  --switchover-timeout 300

# Exemplo:
aws rds switchover-blue-green-deployment \
  --blue-green-deployment-identifier prod-storage-optimization \
  --switchover-timeout 300
```

**Via Console**:
1. RDS Console → Blue/Green Deployments
2. Selecione o deployment
3. Actions → Switch over
4. Configure timeout: 300 segundos
5. Confirme

**Resultado**: O endpoint volta a apontar para o ambiente Blue (original).

**Tempo estimado**: 1-2 minutos de downtime

---

### Cenário 3: Rollback APÓS Deletar o Blue/Green Deployment

**Situação**: Você deletou o Blue/Green Deployment mas manteve o ambiente Blue.

**Impacto**: Downtime de 5-10 minutos para reconfiguração de DNS/endpoint.

**Pré-requisitos**:
- Ambiente Blue ainda existe (não foi deletado com --delete-target)
- Você tem acesso ao endpoint do Blue

**Procedimento**:

```bash
# 1. Identificar o endpoint do ambiente Blue
aws rds describe-db-instances \
  --db-instance-identifier <ID_DA_INSTANCIA_BLUE> \
  --query 'DBInstances[0].Endpoint.Address'

# Exemplo:
aws rds describe-db-instances \
  --db-instance-identifier prod-database-blue \
  --query 'DBInstances[0].Endpoint.Address'

# 2. Atualizar aplicações para apontar para o endpoint Blue
# (Requer mudança de configuração nas aplicações)

# 3. Opcional: Renomear a instância Blue para o nome original
aws rds modify-db-instance \
  --db-instance-identifier <ID_DA_INSTANCIA_BLUE> \
  --new-db-instance-identifier <NOME_ORIGINAL> \
  --apply-immediately

# 4. Deletar a instância Green (antiga produção)
aws rds delete-db-instance \
  --db-instance-identifier <ID_DA_INSTANCIA_GREEN> \
  --skip-final-snapshot
```

**Resultado**: Aplicações voltam a usar o ambiente Blue original.

**Tempo estimado**: 5-10 minutos

---

### Cenário 4: Rollback via Snapshot (Último Recurso)

**Situação**: O ambiente Blue foi deletado e você precisa reverter completamente.

**Impacto**: Downtime de 30-60 minutos. Perda de dados desde o snapshot.

**Pré-requisitos**:
- Snapshot manual criado antes do Blue/Green (conforme Etapa 1.1 do README)
- Ou snapshot automático recente disponível

**Procedimento**:

```bash
# 1. Listar snapshots disponíveis
aws rds describe-db-snapshots \
  --db-instance-identifier <NOME_DA_INSTANCIA_ORIGINAL> \
  --query 'DBSnapshots[*].[DBSnapshotIdentifier,SnapshotCreateTime]' \
  --output table

# 2. Restaurar snapshot para nova instância
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier <NOME_NOVA_INSTANCIA> \
  --db-snapshot-identifier <ID_DO_SNAPSHOT> \
  --db-instance-class <CLASSE_DA_INSTANCIA> \
  --storage-type gp3 \
  --allocated-storage <TAMANHO_ORIGINAL_GB> \
  --iops 3000

# Exemplo:
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier prod-database-restored \
  --db-snapshot-identifier backup-before-bluegreen-20240115-143000 \
  --db-instance-class db.m6g.xlarge \
  --storage-type gp3 \
  --allocated-storage 4096 \
  --iops 3000

# 3. Aguardar restauração completar
aws rds wait db-instance-available \
  --db-instance-identifier <NOME_NOVA_INSTANCIA>

# 4. Atualizar aplicações para novo endpoint
# 5. Validar dados e funcionalidade
# 6. Deletar instância problemática após validação
```

**Resultado**: Nova instância criada a partir do snapshot com configuração original.

**Tempo estimado**: 30-60 minutos

**ATENÇÃO**: Você perderá todas as transações realizadas após a criação do snapshot.

---

## Checklist de Validação Pós-Rollback

Após executar qualquer procedimento de rollback, valide:

**Conectividade**:
```bash
# PostgreSQL
psql -h <ENDPOINT_RDS> -U <USUARIO> -d <DATABASE> -c "SELECT version();"

# MySQL
mysql -h <ENDPOINT_RDS> -u <USUARIO> -p -e "SELECT VERSION();"
```

**Storage**:
```bash
aws rds describe-db-instances \
  --db-instance-identifier <NOME_DA_INSTANCIA> \
  --query 'DBInstances[0].[AllocatedStorage,StorageType,Iops]'
```

**Métricas CloudWatch**:
- CPU Utilization
- Database Connections
- Read/Write Latency
- FreeStorageSpace

**Testes de Aplicação**:
- Executar testes de integração
- Validar funcionalidades críticas
- Verificar logs de erro
- Monitorar performance

---

## Matriz de Decisão de Rollback

| Situação | Cenário Recomendado | Downtime | Perda de Dados |
|----------|---------------------|----------|----------------|
| Problema detectado antes do switchover | Cenário 1 | Zero | Não |
| Problema nas primeiras 24h após switchover | Cenário 2 | 1-2 min | Não |
| Blue/Green deletado mas Blue existe | Cenário 3 | 5-10 min | Não |
| Blue foi deletado completamente | Cenário 4 | 30-60 min | Sim* |

*Perda de dados desde o último snapshot

---

## Prevenção de Necessidade de Rollback

**Boas Práticas**:

1. **Sempre crie snapshot manual antes de iniciar** (Etapa 1.1)
2. **Mantenha o ambiente Blue por 24-48 horas** após switchover
3. **Execute testes completos no ambiente Green** antes do switchover
4. **Monitore métricas ativamente** nas primeiras horas pós-switchover
5. **Documente o endpoint do Blue** antes de deletar
6. **Configure alertas do CloudWatch** para detecção rápida de problemas
7. **Tenha equipe de plantão disponível** durante e após o switchover

**Evite**:
- Deletar o Blue/Green Deployment imediatamente após switchover
- Deletar o ambiente Blue sem validação completa
- Executar switchover sem snapshot de segurança
- Ignorar alertas ou métricas anormais

**Documentação de Suporte**:
- [AWS RDS Blue/Green Deployments](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/blue-green-deployments.html)
- [AWS Support](https://console.aws.amazon.com/support/)
