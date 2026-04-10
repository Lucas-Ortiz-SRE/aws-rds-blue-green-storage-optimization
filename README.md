# Otimização de Storage no AWS RDS com Blue/Green Deployment

Estrutura de otimização de custos para AWS RDS: Redução de storage utilizando Blue/Green Deployment para eliminar desperdício e minimizar downtime.

---

## Visão Geral do Projeto

Este projeto documenta a estratégia de redução de storage em bancos de dados Amazon RDS utilizando a funcionalidade Blue/Green Deployment. O objetivo é reduzir custos operacionais eliminando storage provisionado em excesso (por exemplo, reduzir de 4TB para 1TB) com downtime mínimo e zero perda de dados.

**Por que isso é necessário?** O Amazon RDS não permite redução direta de storage após o provisionamento. A única forma de reduzir o tamanho é através de:
- Dump + Restore (com downtime prolongado)
- Blue/Green Deployment (downtime de segundos a minutos)

**Documentação Adicional**:
- [Plano de Rollback](plano-rollback/ROLLBACK.md) - Procedimentos detalhados para reversão em caso de problemas

---

## Motivação e Impacto FinOps

### Por que Blue/Green?

O Blue/Green Deployment foi escolhido por oferecer:

- **Downtime mínimo**: Switchover em segundos (tipicamente 1-2 minutos)
- **Zero data loss**: Replicação síncrona garante integridade
- **Rollback seguro**: Ambiente Blue permanece disponível após switchover
- **Validação prévia**: Possibilidade de testar o ambiente Green antes da promoção

### Impacto Financeiro

**Exemplo de economia:**
- Storage atual: 4TB (4096 GB) em gp3
- Storage necessário: 1TB (1024 GB)
- Redução: 3TB (3072 GB)

**Cálculo mensal (us-east-1):**
```
Storage gp3: $0.08/GB-month
Economia: 3072 GB × $0.08 = $245.76/mês
Economia anual: $2,949.12/ano
```

**Custo da operação:**
- Durante o Blue/Green: ~2-4 horas de storage duplicado (~$0.50-$1.00)
- ROI: Recuperação do investimento em menos de 1 dia

---

## Pré-requisitos

### Configurações Obrigatórias

#### 1. Backups Automáticos

**Por que é necessário:** O Blue/Green Deployment utiliza a infraestrutura de backups automáticos para criar a réplica inicial do ambiente Green. Sem backups habilitados, não é possível criar o deployment.

```bash
# Verificar se backups automáticos estão habilitados
aws rds describe-db-instances \
  --db-instance-identifier <NOME_DA_INSTANCIA_RDS> \
  --query 'DBInstances[0].BackupRetentionPeriod'

# Deve retornar valor > 0 (recomendado: 7-35 dias)
```

#### 2. Binary Logging (MySQL/MariaDB)

**Por que é necessário:** O binary log registra todas as alterações no banco de dados e é essencial para a replicação contínua entre os ambientes Blue e Green. O formato ROW garante que as mudanças sejam replicadas de forma precisa, linha por linha, evitando inconsistências.

```sql
-- Verificar se binary logging está habilitado
SHOW VARIABLES LIKE 'log_bin';
-- Deve retornar: log_bin = ON
```

Parameter Group deve ter:
```
backup_retention_period >= 1
binlog_format = ROW (MySQL/MariaDB)
```

#### 3. Permissões IAM Necessárias

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "rds:CreateBlueGreenDeployment",
        "rds:DeleteBlueGreenDeployment",
        "rds:DescribeBlueGreenDeployments",
        "rds:SwitchoverBlueGreenDeployment",
        "rds:DescribeDBInstances",
        "rds:DescribeDBClusters",
        "rds:ModifyDBInstance",
        "rds:CreateDBInstanceReadReplica"
      ],
      "Resource": "*"
    }
  ]
}
```

### Limitações e Restrições

- Não suporta instâncias com Read Replicas cross-region
- Não suporta instâncias em Domain (Microsoft AD)
- Não suporta RDS Proxy durante o deployment
- Não suporta instâncias que usam AWS Secrets Manager para gerenciar senhas
- Storage do Green deve ser maior ou igual ao espaço usado no Blue
- Máximo de 5 Blue/Green deployments simultâneos por região

### Atenção Especial: Instâncias Burstable (Família T)

**IMPORTANTE**: Se sua instância RDS utiliza a família T, considere migrar temporariamente para uma instância de outra família (M, R ou C) antes de iniciar o Blue/Green Deployment.

**Por que isso é necessário?**

Instâncias da família T utilizam um modelo de créditos de CPU. Durante o Blue/Green Deployment:
- A replicação inicial consome CPU e memória intensivamente
- A sincronização contínua mantém uso elevado de recursos
- Os créditos de CPU podem se esgotar rapidamente
- Quando os créditos acabam, a performance cai drasticamente
- Você pode ter custos adicionais com créditos excedentes

**Recomendação:**

1. **Antes do Blue/Green**: Modifique a instância para m6g+, r6g+ ou c6g+ (ou equivalente)
2. **Execute o Blue/Green Deployment**: Com a instância M/R/C, o processo será mais rápido e estável
3. **Após validação**: Volte para a instância T se necessário (após deletar o ambiente Blue)

**Custo adicional**: A diferença de custo entre T e M/R/C por algumas horas é mínima comparada ao risco de problemas durante o deployment.

---

## Etapa 1: Avaliação de Recursos e Planejamento

### 1.1 Criar Snapshot de Segurança

**Recomendação**: Antes de iniciar qualquer operação, crie um snapshot manual do RDS atual para fins de rollback em caso de necessidade.

```bash
# Criar snapshot manual
aws rds create-db-snapshot \
  --db-instance-identifier <NOME_DA_INSTANCIA_RDS> \
  --db-snapshot-identifier backup-before-bluegreen-$(date +%Y%m%d-%H%M%S)

# Aguardar snapshot completar
aws rds wait db-snapshot-completed \
  --db-snapshot-identifier backup-before-bluegreen-$(date +%Y%m%d-%H%M%S)
```

### 1.2 Verificar Métricas no CloudWatch

```bash
# Verificar uso de storage nos últimos 7 dias
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name FreeStorageSpace \
  --dimensions Name=DBInstanceIdentifier,Value=<NOME_DA_INSTANCIA_RDS> \
  --start-time $(date -u -d '7 days ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 86400 \
  --statistics Average
```

### 1.3 Calcular Storage Mínimo Necessário

**Fórmula:**
```
Storage Mínimo = (Espaço Usado × 1.2) + Buffer de Crescimento
```

**Explicação do cálculo:**

1. **Espaço Usado**: Valor obtido das métricas do CloudWatch (Storage alocado - FreeStorageSpace)
2. **Margem de 20% (1.2)**: Adiciona segurança para:
   - Fragmentação do sistema de arquivos
   - Arquivos temporários durante operações
   - Logs de transações (WAL, binlog)
   - Overhead do sistema operacional
3. **Buffer de Crescimento**: Espaço adicional para crescimento futuro (recomendado: 100-200 GB)

**Exemplo (cenário 4TB → 1TB):**
- Storage alocado atual: 4096 GB (4TB)
- FreeStorageSpace (CloudWatch): 3246 GB
- **Espaço usado**: 4096 - 3246 = 850 GB
- Margem de segurança (20%): 850 × 0.2 = 170 GB
- Buffer de crescimento: 100 GB
- **Cálculo final**: 850 + 170 + 100 = 1120 GB
- **Storage recomendado**: 1200 GB (arredondado para cima)

**IMPORTANTE**: Nunca provisione storage muito próximo ao limite usado. Sempre mantenha pelo menos 15-20% de espaço livre.

### 1.4 Verificar Tipo de Volume e IOPS

```bash
aws rds describe-db-instances \
  --db-instance-identifier <NOME_DA_INSTANCIA_RDS> \
  --query 'DBInstances[0].[StorageType,AllocatedStorage,Iops]'
```

**Limites de IOPS por tipo:**
- **gp3**: 3,000-16,000 IOPS (independente do tamanho)
- **gp2**: 3 IOPS/GB (mínimo 100, máximo 16,000)
- **io1/io2**: 1,000-256,000 IOPS (50:1 ratio)

**Atenção**: Ao reduzir storage em gp2, você também reduz IOPS proporcionalmente.

### 1.5 Verificar Extensões e Compatibilidade

```sql
-- PostgreSQL: Listar extensões instaladas
SELECT extname, extversion FROM pg_extension;

-- MySQL: Verificar plugins
SELECT plugin_name, plugin_status FROM information_schema.plugins;
```

---

## Etapa 2: Provisionamento do Ambiente Green

### 2.1 Criar Blue/Green Deployment via Console

1. Acesse RDS Console → Databases
2. Selecione a instância Blue (produção)
3. Actions → Create Blue/Green Deployment

**[INSERIR PRINT: Tela inicial do RDS Console com a instância selecionada]**

4. Configure:
   - **Blue/Green deployment identifier**: `prod-storage-optimization`
   - **DB engine version**: (manter a mesma ou atualizar)
   - **Allocated storage**: `1200` (novo tamanho reduzido - cenário 4TB → 1TB)
   - **Storage type**: `gp3` (manter ou alterar)
   - **Provisioned IOPS**: (se aplicável)

**[INSERIR PRINT: Formulário de criação do Blue/Green Deployment com campos preenchidos]**

### 2.2 Criar via AWS CLI

```bash
aws rds create-blue-green-deployment \
  --blue-green-deployment-name <NOME_DO_DEPLOYMENT> \
  --source-arn <ARN_DA_INSTANCIA_RDS_BLUE> \
  --target-db-instance-class <CLASSE_DA_INSTANCIA> \
  --target-allocated-storage <TAMANHO_EM_GB> \
  --target-storage-type <TIPO_STORAGE> \
  --target-iops <IOPS> \
  --tags Key=Environment,Value=Production Key=Purpose,Value=StorageOptimization
```

**Exemplo prático (cenário 4TB → 1TB):**
```bash
aws rds create-blue-green-deployment \
  --blue-green-deployment-name prod-storage-optimization \
  --source-arn arn:aws:rds:us-east-1:123456789012:db:prod-database \
  --target-db-instance-class db.r6g.xlarge \
  --target-allocated-storage 1200 \
  --target-storage-type gp3 \
  --target-iops 3000 \
  --tags Key=Environment,Value=Production Key=Purpose,Value=StorageOptimization
```

**Parâmetros:**
- `--source-arn`: ARN completo da instância RDS Blue (produção)
- `--target-db-instance-class`: Classe da instância (ex: db.r6g.xlarge)
- `--target-allocated-storage`: Novo tamanho em GB (1200 GB no exemplo)
- `--target-storage-type`: Tipo de storage (gp3, gp2, io1, io2)
- `--target-iops`: IOPS provisionado (apenas para gp3, io1, io2)

### 2.3 Verificar Status da Criação

```bash
# Monitorar progresso
aws rds describe-blue-green-deployments \
  --blue-green-deployment-identifier <NOME_DO_DEPLOYMENT>

# Exemplo:
aws rds describe-blue-green-deployments \
  --blue-green-deployment-identifier prod-storage-optimization

# Status esperados:
# - PROVISIONING: Criando recursos
# - AVAILABLE: Pronto para switchover
# - SWITCHOVER_IN_PROGRESS: Durante a troca
# - SWITCHOVER_COMPLETED: Finalizado
```

**Tempo estimado**: 15-45 minutos dependendo do tamanho do banco.

**[INSERIR PRINT: Status do deployment como PROVISIONING ou AVAILABLE]**

---

## Etapa 3: Monitoramento da Sincronização de Dados

### 3.1 Verificar Replication Lag

```bash
# CloudWatch Metric
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name ReplicaLag \
  --dimensions Name=DBInstanceIdentifier,Value=<ID_INSTANCIA_GREEN> \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 60 \
  --statistics Average,Maximum
```

### 3.2 Verificar Integridade no Banco

**PostgreSQL:**
```sql
-- No ambiente Green (read-only durante sync)
SELECT 
  pg_is_in_recovery() as is_replica,
  pg_last_wal_receive_lsn() as receive_lsn,
  pg_last_wal_replay_lsn() as replay_lsn,
  pg_wal_lsn_diff(pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn()) as lag_bytes;
```

**MySQL/MariaDB:**
```sql
SHOW REPLICA STATUS\G
-- Verificar:
-- Seconds_Behind_Master: deve ser 0 ou próximo
-- Replica_IO_Running: Yes
-- Replica_SQL_Running: Yes
```

### 3.3 Critérios para Switchover

**Pronto para switchover quando:**
- Replication Lag < 1 segundo
- Status = AVAILABLE
- Nenhum erro de replicação
- Validação de integridade concluída

**[INSERIR PRINT: CloudWatch mostrando ReplicaLag próximo de zero]**

---

## Etapa 4: Fase de Switchover (Cutover)

### 4.1 Preparação para Switchover

**Checklist pré-switchover:**
- Notificar stakeholders sobre janela de manutenção
- Backup manual recente disponível
- Replication lag < 1 segundo
- Plano de rollback documentado (consulte [Plano de Rollback](plano-rollback/ROLLBACK.md))
- Monitoramento ativo (CloudWatch, APM)
- Equipe de plantão disponível

### 4.2 Executar Switchover

**Via Console:**
1. RDS Console → Blue/Green Deployments
2. Selecione o deployment
3. Actions → Switch over

**[INSERIR PRINT: Tela de Blue/Green Deployments com botão Switch over]**

4. Configure timeout: `300` segundos (recomendado)
5. Confirme a operação

**[INSERIR PRINT: Modal de confirmação do switchover com timeout configurado]**

**Via CLI:**
```bash
aws rds switchover-blue-green-deployment \
  --blue-green-deployment-identifier <NOME_DO_DEPLOYMENT> \
  --switchover-timeout <TIMEOUT_EM_SEGUNDOS>

# Exemplo:
aws rds switchover-blue-green-deployment \
  --blue-green-deployment-identifier prod-storage-optimization \
  --switchover-timeout 300
```

**Parâmetros:**
- `--blue-green-deployment-identifier`: Nome do deployment criado na Etapa 2
- `--switchover-timeout`: Tempo máximo em segundos (recomendado: 300)

### 4.3 O que Acontece Durante o Switchover

**Sequência de eventos:**

1. Início do switchover
2. Blue entra em read-only mode
3. Replicação final (catch-up)
4. DNS swap (endpoint aponta para Green)
5. Green promovido para read-write
6. Conexões antigas são encerradas
7. Switchover completo

**Comportamento das conexões:**
- Conexões ativas no Blue são **terminadas** após o timeout
- Aplicações devem ter **retry logic** implementado
- Connection pooling reconecta automaticamente ao novo endpoint

### 4.4 Monitorar o Switchover

```bash
# Terminal 1: Monitorar status
watch -n 2 'aws rds describe-blue-green-deployments \
  --blue-green-deployment-identifier <NOME_DO_DEPLOYMENT> \
  --query "BlueGreenDeployments[0].Status"'

# Terminal 2: Monitorar conexões
watch -n 1 'aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name DatabaseConnections \
  --dimensions Name=DBInstanceIdentifier,Value=<NOME_DA_INSTANCIA_RDS> \
  --start-time $(date -u -d "5 minutes ago" +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 60 \
  --statistics Average'
```

**[INSERIR PRINT: Dashboard mostrando métricas durante o switchover]**

---

## Etapa 5: Pós-Deployment e Limpeza

### 5.1 Validação Pós-Switchover

**Verificar conectividade:**
```bash
# Testar conexão
psql -h <ENDPOINT_RDS> -U <USUARIO> -d <DATABASE> -c "SELECT version();"
# ou
mysql -h <ENDPOINT_RDS> -u <USUARIO> -p -e "SELECT VERSION();"
```

**Verificar storage:**
```bash
aws rds describe-db-instances \
  --db-instance-identifier <NOME_DA_INSTANCIA_RDS> \
  --query 'DBInstances[0].[AllocatedStorage,StorageType,Iops]'
```

**Verificar métricas:**
- CPU Utilization
- Database Connections
- Read/Write Latency
- IOPS

**[INSERIR PRINT: Detalhes da instância mostrando novo storage alocado]**

### 5.2 Período de Observação

**Recomendação**: Manter ambiente Blue por 24-48 horas antes de deletar.

**Por que manter o Blue?** O ambiente Blue serve como caminho de rollback rápido caso problemas sejam identificados. Consulte o [Plano de Rollback](plano-rollback/ROLLBACK.md) para procedimentos detalhados de reversão.

**Monitorar:**
- Performance da aplicação
- Erros de conexão
- Crescimento de storage
- Alertas do CloudWatch

### 5.3 Deletar Blue/Green Deployment

**ATENÇÃO**: Esta ação deleta o ambiente Blue (antigo).

**Via Console:**
1. RDS Console → Blue/Green Deployments
2. Selecione o deployment
3. Actions → Delete
4. Escolha: **Delete the blue database instances**
5. Confirme

**Via CLI:**
```bash
aws rds delete-blue-green-deployment \
  --blue-green-deployment-identifier <NOME_DO_DEPLOYMENT> \
  --delete-target

# Exemplo:
aws rds delete-blue-green-deployment \
  --blue-green-deployment-identifier prod-storage-optimization \
  --delete-target
```

### 5.4 Snapshot Final (Opcional)

```bash
# Criar snapshot do ambiente Blue antes de deletar
aws rds create-db-snapshot \
  --db-instance-identifier <ID_DA_INSTANCIA_BLUE> \
  --db-snapshot-identifier final-snapshot-before-deletion-$(date +%Y%m%d)

# Exemplo:
aws rds create-db-snapshot \
  --db-instance-identifier prod-database-blue \
  --db-snapshot-identifier final-snapshot-before-deletion-$(date +%Y%m%d)
```

**[INSERIR PRINT: Confirmação de deleção do Blue/Green Deployment]**

---

## Resolução de Problemas e Lições Aprendidas

### Problema 1: Replication Lag Alto

**Sintoma**: Lag permanece > 10 segundos

**Causas:**
- Carga de escrita muito alta no Blue
- Instância Green subdimensionada
- Network throughput insuficiente

**Solução:**
```bash
# 1. Reduzir carga de escrita temporariamente
# 2. Aguardar horário de menor carga
```

### Problema 2: Switchover Timeout

**Sintoma**: Switchover falha por timeout

**Causas:**
- Transações longas em execução
- Conexões não fechando corretamente
- Timeout muito curto

**Solução:**
```sql
-- PostgreSQL: Identificar transações longas
SELECT pid, usename, state, query_start, query
FROM pg_stat_activity
WHERE state != 'idle'
AND query_start < NOW() - INTERVAL '5 minutes';

-- Terminar transações problemáticas
SELECT pg_terminate_backend(pid);
```

### Problema 3: Storage Insuficiente no Green

**Sintoma**: Erro ao criar Blue/Green - "storage too small"

**Causa**: Storage do Green < espaço usado no Blue

**Solução:**
```bash
# 1. Verificar espaço real usado
# 2. Adicionar margem de 20%
# 3. Recriar deployment com storage maior
```
### Lições Aprendidas

**Boas Práticas:**

1. **Sempre teste em ambiente não-produtivo primeiro**
2. **Execute durante janela de baixa carga**
3. **Mantenha Blue por 24-48h antes de deletar** (permite rollback rápido)
4. **Configure alertas do CloudWatch antes do switchover**
5. **Documente o processo e tempos observados**
6. **Implemente retry logic nas aplicações**
7. **Monitore crescimento de storage pós-redução**
8. **Tenha o [Plano de Rollback](plano-rollback/ROLLBACK.md) acessível durante a operação**

**Evite:**

1. Switchover durante horário de pico
2. Reduzir storage muito próximo ao limite usado
3. Deletar Blue imediatamente após switchover
4. Ignorar replication lag alto
5. Não testar conectividade pós-switchover

---

## Calculadora de Otimização de Custos

**Cenário de Exemplo: Redução de 4TB para 1TB**

**Configuração para simulação:**

**Antes (Ambiente Atual):**
- Região: US East (N. Virginia)
- Engine: MySQL
- Instance Type: db.m6g.xlarge
- Storage Type: General Purpose SSD (gp3)
- Allocated Storage: 4096 GB (4TB)
- IOPS: 3000
- **Total Monthly Cost: $692.96**

**Depois (Ambiente Otimizado):**
- Região: US East (N. Virginia)
- Engine: MySQL
- Instance Type: db.m6g.xlarge (mesmo)
- Storage Type: General Purpose SSD (gp3)
- Allocated Storage: 1024 GB (1TB)
- IOPS: 3000
- **Total Monthly Cost: $339.68**

**Economia Estimada:**
- **Economia mensal: $353.28**
- **Economia anual: $4,239.36**
- **Economia em 3 anos: $12,718.08**

**Acesse a [Calculadora AWS](https://calculator.aws/#/estimate?id=f6c63688740d4c1cdcf7bfb036d5b7654d22c7c4) para simular seu cenário**
---

## Referências

- [AWS RDS Blue/Green Deployments](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/blue-green-deployments.html)
- [Blue/Green Best Practices](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/blue-green-deployments-best-practices.html)
- [RDS Storage Types](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_Storage.html)
- [RDS Pricing Calculator](https://calculator.aws)
- [AWS Well-Architected Framework - Cost Optimization](https://docs.aws.amazon.com/wellarchitected/latest/cost-optimization-pillar/welcome.html)
