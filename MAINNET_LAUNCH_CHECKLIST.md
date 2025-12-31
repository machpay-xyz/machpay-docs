# 🚀 MachPay Mainnet Launch Checklist

**Version:** 1.0.0  
**Last Updated:** December 2024  
**Status:** Pre-Production

---

## ⚠️ CRITICAL DISCLAIMER

> **REAL MONEY IS AT STAKE.**  
> This checklist must be completed by authorized personnel before any mainnet deployment.  
> Do NOT proceed unless you have explicit approval from the security team.

---

## 📋 Pre-Launch Checklist

### 1. Key Management & Security 🔐

| Status | Item | Responsible | Notes |
|--------|------|-------------|-------|
| ☐ | **Admin keypair generated offline** | Security Lead | Use air-gapped machine |
| ☐ | **Admin keypair backed up to cold storage** | Security Lead | 2+ physical locations |
| ☐ | **Vendor keypairs rotated** | DevOps | Generate fresh for production |
| ☐ | **Old test/devnet keypairs invalidated** | DevOps | Remove from all systems |
| ☐ | **JWT secrets rotated** | Backend Lead | 48+ character random string |
| ☐ | **API keys rotated for all services** | DevOps | Unique per service |
| ☐ | **Secrets stored in vault (HashiCorp/AWS SM)** | DevOps | NOT in git or env files |
| ☐ | **2FA enabled on all RPC provider accounts** | DevOps | Helius, Triton, etc. |
| ☐ | **2FA enabled on cloud provider accounts** | DevOps | AWS, GCP, etc. |

### 2. Infrastructure & Network 🌐

| Status | Item | Responsible | Notes |
|--------|------|-------------|-------|
| ☐ | **Relayer IP whitelisted in firewall** | DevOps | Only relayer → gateway traffic |
| ☐ | **Database IP whitelisted** | DevOps | No public access |
| ☐ | **SSL/TLS certificates installed** | DevOps | Valid certs, not self-signed |
| ☐ | **HTTPS enforced on all endpoints** | DevOps | HTTP → HTTPS redirect |
| ☐ | **DDoS protection enabled** | DevOps | Cloudflare, AWS Shield, etc. |
| ☐ | **Rate limiting configured** | DevOps | See config checklist |
| ☐ | **Load balancer health checks configured** | DevOps | /health endpoints |
| ☐ | **Auto-scaling policies defined** | DevOps | Based on CPU/memory thresholds |

### 3. Database 💾

| Status | Item | Responsible | Notes |
|--------|------|-------------|-------|
| ☐ | **Database backed up** | DBA | Full backup before launch |
| ☐ | **Automated backup schedule configured** | DBA | Daily minimum |
| ☐ | **Point-in-time recovery enabled** | DBA | 7+ day retention |
| ☐ | **Database credentials rotated** | DBA | Unique production credentials |
| ☐ | **Connection pooling configured** | Backend Lead | pg_bouncer or similar |
| ☐ | **SSL/TLS for DB connections** | DBA | sslmode=require |
| ☐ | **Read replicas deployed** | DBA | For read-heavy loads |
| ☐ | **Migration scripts tested** | Backend Lead | On production-like data |

### 4. Monitoring & Alerting 📊

| Status | Item | Responsible | Notes |
|--------|------|-------------|-------|
| ☐ | **Prometheus/Grafana deployed** | DevOps | Or equivalent |
| ☐ | **Log aggregation configured** | DevOps | ELK, Loki, etc. |
| ☐ | **Error tracking enabled** | DevOps | Sentry, Rollbar, etc. |
| ☐ | **PagerDuty/OpsGenie configured** | DevOps | 24/7 on-call rotation |
| ☐ | **Alert thresholds defined** | DevOps | See monitoring checklist |
| ☐ | **Dashboard for key metrics** | DevOps | Revenue, latency, errors |
| ☐ | **Uptime monitoring** | DevOps | External pingdom/statuspage |

### 5. Solana On-Chain 🔗

| Status | Item | Responsible | Notes |
|--------|------|-------------|-------|
| ☐ | **Program deployed to mainnet** | Blockchain Lead | Final audit complete |
| ☐ | **Program upgrade authority secured** | Blockchain Lead | Multisig or frozen |
| ☐ | **GlobalBond initialized** | Blockchain Lead | With mainnet USDC mint |
| ☐ | **Admin authority in multisig** | Security Lead | 2/3 or 3/5 minimum |
| ☐ | **Sufficient SOL for fees** | DevOps | Admin, relayer accounts |
| ☐ | **RPC provider is production-grade** | DevOps | NOT public endpoints |
| ☐ | **RPC rate limits understood** | DevOps | Stay under provider limits |

### 6. Application Configuration ⚙️

| Status | Item | Responsible | Notes |
|--------|------|-------------|-------|
| ☐ | **DEV_MODE=false on all services** | DevOps | CRITICAL |
| ☐ | **LOG_LEVEL=info (not debug)** | DevOps | Reduce log volume |
| ☐ | **METRICS_ENABLED=true** | DevOps | For observability |
| ☐ | **Correct USDC mint address** | DevOps | EPjFW... (mainnet) |
| ☐ | **Correct program ID** | DevOps | Mainnet deployed ID |
| ☐ | **Gateway pricing configured** | Product | Review fee structure |
| ☐ | **Settlement intervals configured** | DevOps | 60s default |

---

## 🔴 Critical Alert Thresholds

Configure alerts for these scenarios:

| Metric | Warning | Critical |
|--------|---------|----------|
| Gateway P99 Latency | > 500ms | > 2s |
| Error Rate (5xx) | > 1% | > 5% |
| Payment Rejection Rate | > 10% | > 25% |
| Settlement Queue Depth | > 100 | > 500 |
| GlobalBond Vault Balance | < $10,000 | < $1,000 |
| Database Connection Pool | > 80% | > 95% |
| Memory Usage | > 80% | > 95% |
| Disk Usage | > 70% | > 90% |

---

## 🔒 Security Hardening

### Firewall Rules

```bash
# Allow only from known IPs
ufw default deny incoming
ufw default allow outgoing
ufw allow from $RELAYER_IP to any port 8080
ufw allow from $LB_IP to any port 8081
ufw allow from $INTERNAL_NET to any port 5432
```

### Rate Limiting

| Endpoint | Limit | Window |
|----------|-------|--------|
| `/v1/pay/*` | 100 req/s | 1s |
| `/v1/link/*` | 10 req/min | 1min |
| `/v1/auth/*` | 5 req/min | 1min |

---

## 📞 Emergency Contacts

| Role | Name | Phone | Email |
|------|------|-------|-------|
| Security Lead | | | |
| DevOps Lead | | | |
| Backend Lead | | | |
| On-Call Primary | | | |
| On-Call Secondary | | | |

---

## 🚦 Go/No-Go Decision

### Final Sign-Offs Required

| Role | Name | Signature | Date |
|------|------|-----------|------|
| Engineering Lead | | | |
| Security Lead | | | |
| DevOps Lead | | | |
| Product Lead | | | |

---

## 📝 Post-Launch Checklist

### Immediately After Launch

| Status | Item | Responsible |
|--------|------|-------------|
| ☐ | Verify health endpoints responding | DevOps |
| ☐ | Confirm metrics flowing to dashboards | DevOps |
| ☐ | Execute smoke test transaction | QA |
| ☐ | Verify settlement execution | Backend Lead |
| ☐ | Confirm logs aggregating | DevOps |

### Within 24 Hours

| Status | Item | Responsible |
|--------|------|-------------|
| ☐ | Review error rates | Backend Lead |
| ☐ | Check database performance | DBA |
| ☐ | Verify backup completion | DBA |
| ☐ | Review security logs | Security Lead |
| ☐ | Update status page | DevOps |

### Within 7 Days

| Status | Item | Responsible |
|--------|------|-------------|
| ☐ | Full security scan | Security Lead |
| ☐ | Performance baseline established | DevOps |
| ☐ | Incident runbook validated | DevOps |
| ☐ | DR drill scheduled | DevOps |

---

## 🔙 Rollback Plan

### If Critical Issues Occur:

1. **Stop new traffic** - Update load balancer to maintenance mode
2. **Notify users** - Update status page
3. **Assess damage** - Check for financial losses
4. **Rollback application** - Deploy previous version
5. **Rollback database** - If migrations caused issues
6. **Post-mortem** - Document and learn

### Rollback Commands

```bash
# Stop new deployments
kubectl rollout pause deployment/gateway
kubectl rollout pause deployment/relayer

# Rollback to previous version
kubectl rollout undo deployment/gateway
kubectl rollout undo deployment/relayer

# Verify rollback
kubectl rollout status deployment/gateway
```

---

## 📚 Reference Documents

- [MachPay Architecture Overview](./architecture/overview.md)
- [Security Best Practices](./security/best-practices.md)
- [Incident Response Playbook](./operations/incident-response.md)
- [Disaster Recovery Plan](./operations/disaster-recovery.md)

---

**Remember:** *Security is not a feature, it's a foundation. Every item on this checklist exists because of a past incident in the industry.*





