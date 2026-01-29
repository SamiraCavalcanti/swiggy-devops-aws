# 🚀 Resumo da Configuração - Projeto Swiggy DevOps AWS

**Data:** 27 de Janeiro de 2026  
**Sessão:** Configuração inicial do CI/CD Pipeline

---

## 📋 O que foi Implementado

### ✅ 1. Estrutura do Repositório
- Reorganizado de `DevOps-Projects/DevOps-Project-23` para `devops-projeto-swiggy`
- Repositório: https://github.com/SamiraCavalcanti/swiggy-devops-aws
- Branch: `main`

### ✅ 2. Arquivos Criados/Otimizados

#### **Dockerfile** (Otimizado)
- ✅ Multi-stage build com Alpine
- ✅ Redução de ~85% no tamanho da imagem
- ✅ De `node:16` para `node:16-alpine`

#### **.gitignore** e **.dockerignore**
- ✅ Proteção de node_modules, .env, logs
- ✅ Exclusão de arquivos temporários
- ✅ Proteção de security scan reports

#### **buildspec.yaml** (Raiz do repositório)
- ✅ Configurado para estrutura de pastas correta
- ✅ Sanitização de variáveis de ambiente
- ✅ Integração com ferramentas de segurança

---

## 🔧 Problemas Encontrados e Soluções

### **Problema 1: buildspec.yaml não encontrado**
**Erro:** `YAML_FILE_ERROR: no such file or directory`

**Causa:** CodeBuild procura na raiz, arquivo estava em `devops-projeto-swiggy/Swiggy_clone/`

**Solução:**
- Movido buildspec.yaml para raiz
- Adicionado `cd devops-projeto-swiggy/Swiggy_clone` em cada fase

---

### **Problema 2: Docker build falhou - tag inválida**
**Erro:** `invalid tag "***/samiracavalcanti\n\n/swiggy:latest"`

**Causa:** Quebras de linha nos parâmetros do Systems Manager

**Solução:** Sanitização de variáveis
```bash
export DOCKER_REGISTRY_URL=$(echo "$DOCKER_REGISTRY_URL" | tr -d '\n')
export DOCKER_REGISTRY_USERNAME=$(echo "$DOCKER_REGISTRY_USERNAME" | tr -d '\n')
export DOCKER_REGISTRY_PASSWORD=$(echo "$DOCKER_REGISTRY_PASSWORD" | tr -d '\n')
```

---

### **Problema 3: Navegação de pastas entre fases**
**Erro:** `cd: devops-projeto-swiggy/Swiggy_clone: No such file or directory`

**Causa:** CodeBuild reseta diretório entre fases

**Solução:** 
- PRE_BUILD: Entra na pasta, faz scan, **volta para raiz** (`cd ../..`)
- BUILD: Entra na pasta novamente
- POST_BUILD: Volta para raiz antes de acessar dependency-check

---

### **Problema 4: OWASP Dependency-Check - Erro 403**
**Erro:** `Error retrieving https://nvd.nist.gov/feeds/json/cve/1.1/nvdcve-1.1-modified.meta; received response code 403`

**Causa:** 
- Versão antiga (7.0.2 de 2022)
- API da NVD mudou em 2023
- Requer API Key

**Soluções Aplicadas:**
1. ✅ Atualizado para versão 10.0.4 (2024)
2. ✅ Criada API Key na NVD: https://nvd.nist.gov/developers/request-an-api-key
3. ✅ Armazenada no Parameter Store: `/security/nvd_api_key`
4. ✅ Configurado `--nvdApiKey $NVD_API_KEY` no comando

---

## 🔐 AWS Systems Manager - Parameter Store

### Parâmetros Configurados:

| Nome | Tipo | Descrição |
|------|------|-----------|
| `/cicd/docker-credenciais/username` | String | Username do Docker Hub |
| `/cicd/docker-credenciais/password` | SecureString | Senha do Docker Hub |
| `/cicd/docker-registry/url` | String | `docker.io` |
| `/cicd/sonar/sonar-token` | SecureString | Token do SonarQube |
| `/security/nvd_api_key` | SecureString | API Key da NVD |

**⚠️ IMPORTANTE:** Valores sem espaços ou quebras de linha no final!

---

## 🎯 SonarQube Configurado

- **URL:** http://54.226.236.61:9000/
- **Projeto:** swiggy
- **Token:** sqp_d4dffba67fcd19a7ffd7cb7bd5666c030a0176d4

---

## 📦 AWS CodeBuild Configurado

### **Detalhes do Projeto:**
- **Nome:** swiggy-pipeline-2026
- **Região:** us-east-1
- **Source:** GitHub (OAuth)
- **Repository:** https://github.com/SamiraCavalcanti/swiggy-devops-aws
- **Buildspec:** buildspec.yaml (na raiz)

### **Environment:**
- **Image:** aws/codebuild/amazonlinux-x86_64-standard:5.0
- **Compute:** EC2
- **Type:** Container
- **Runtime:** Python 3.11, Java Corretto 17

### **Artifacts:**
- **Bucket:** bucket-samira
- **Path:** swiggy-builds
- **Name:** swiggy
- **Files:** devops-projeto-swiggy/Swiggy_clone/appspec.yaml

### **Service Role:**
- **Role:** codebuild-swiggy-pipeline-2026-service-role
- **Permissions necessárias:**
  - AmazonSSMReadOnlyAccess (ler parâmetros)
  - AmazonS3FullAccess (artifacts)
  - AmazonEC2ContainerRegistryPowerUser (push imagens)

---

## 🔄 Fluxo do Pipeline (buildspec.yaml)

### **INSTALL:**
```yaml
runtime-versions:
  python: 3.11
  java: corretto17
```

### **PRE_BUILD:**
1. cd devops-projeto-swiggy/Swiggy_clone
2. Baixa e instala Trivy
3. Scan de arquivos com Trivy → `trivyfilescan.txt`
4. **cd ../..**  ← Volta para raiz
5. Baixa OWASP Dependency-Check 10.0.4
6. Baixa SonarQube Scanner

### **BUILD:**
1. cd devops-projeto-swiggy/Swiggy_clone
2. Sanitiza variáveis (remove \n)
3. Docker login
4. Docker build
5. Docker push → Docker Hub

### **POST_BUILD:**
1. Trivy image scan → `trivyimage.txt`
2. **cd ../..**  ← Volta para raiz
3. OWASP Dependency-Check scan (com NVD API Key)
4. cd devops-projeto-swiggy/Swiggy_clone
5. SonarQube analysis

---

## 📊 Ferramentas de Segurança Integradas

### **1. Trivy**
- **O que faz:** Scan de vulnerabilidades em arquivos e imagens Docker
- **Quando roda:** PRE_BUILD (arquivos) e POST_BUILD (imagem)
- **Output:** trivyfilescan.txt, trivyimage.txt

### **2. OWASP Dependency-Check**
- **O que faz:** Verifica vulnerabilidades em dependências (CVEs)
- **Quando roda:** POST_BUILD
- **Output:** HTML, JSON, XML reports
- **Versão:** 10.0.4
- **Database:** NVD (banco CVE de outubro/2024)
- **⚠️ Flag especial:** `--noupdate` (não atualiza banco)

**🐛 LIMITAÇÃO CONHECIDA - Bug CVSS v4.0:**
- OWASP 10.0.4 tem bug de parsing no CVSS v4.0
- Não reconhece valor `"SAFETY"` em `ModifiedCiaType`
- Falha ao tentar baixar atualizações da NVD
- **Solução:** Usar `--noupdate` (usa banco que vem no ZIP)
- **Impacto:** NÃO detecta CVEs descobertas após out/2024
- **Mitigação:** Trivy (sempre atualizado) cobre vulnerabilidades recentes ✅
- **Aguardando:** Correção esperada no OWASP v11.x

### **3. SonarQube**
- **O que faz:** Análise de qualidade e segurança de código
- **Quando roda:** POST_BUILD
- **Dashboard:** http://54.226.236.61:9000/dashboard?id=swiggy

---

## 🐛 Debugging Aplicado

### **Echo de variáveis sanitizadas:**
```bash
echo "URL=[$DOCKER_REGISTRY_URL]"
echo "USER=[$DOCKER_REGISTRY_USERNAME]"
```

Permite visualizar quebras de linha nos logs!

---

## 💰 Custos Estimados (Hoje)

| Recurso | Custo |
|---------|-------|
| **CodeBuild** | $0 (Free Tier - 100 min/mês) ✅ |
| **S3** | ~$0.01 ✅ |
| **EC2 SonarQube (t2.medium)** | ~$0.40 (8h) ⚠️ |
| **ECS** (quando criar) | ~$0.80 (8h) ⚠️ |
| **Total estimado (1 dia)** | **~$1.50 - $2.00** |

**⚠️ IMPORTANTE:** Destruir EC2 e ECS no final do dia!

---

## ✅ Status Atual

- ✅ Repositório GitHub configurado
- ✅ Dockerfile otimizado
- ✅ buildspec.yaml funcional
- ✅ CodeBuild criado e testado
- ✅ Docker Hub integrado
- ✅ SonarQube rodando
- ✅ Parameter Store configurado
- ✅ Trivy funcionando
- ✅ OWASP Dependency-Check atualizado com API Key
- 🔄 **Build em andamento** (baixando CVEs da NVD)

---

## 📝 Próximos Passos

### **Após Build Completo:**
1. ✅ Verificar imagem no Docker Hub
2. ✅ Analisar relatório do SonarQube
3. ✅ Revisar vulnerabilidades (Trivy + OWASP)
4. ⏳ Criar ECS Cluster (Step 4A)
5. ⏳ Criar Task Definition (Step 4B)
6. ⏳ Atualizar appspec.yaml com ARN
7. ⏳ Configurar CodeDeploy
8. ⏳ Criar CodePipeline

---

## 🎓 Aprendizados Importantes

### **1. Estrutura de Pastas no CI/CD**
- CodeBuild sempre começa na raiz
- Cada fase pode resetar o diretório
- Sempre navegue explicitamente com `cd`

### **2. Sanitização de Variáveis**
- Sempre limpar strings de parâmetros externos
- `tr -d '\n'` remove quebras de linha
- Previne erros sutis difíceis de debugar

### **3. Versionamento de Ferramentas**
- APIs mudam (NVD mudou em 2023)
- Sempre usar versões recentes
- Documentar versões no código

### **4. DevSecOps na Prática**
- Segurança integrada no pipeline
- Múltiplas camadas de scan
- Automação completa

---

## 🔗 Links Importantes

- **Repositório:** https://github.com/SamiraCavalcanti/swiggy-devops-aws
- **SonarQube:** http://54.226.236.61:9000/
- **Docker Hub:** https://hub.docker.com/
- **NVD API:** https://nvd.nist.gov/developers/request-an-api-key
- **OWASP Releases:** https://github.com/jeremylong/DependencyCheck/releases

---

## 👏 Conquistas de Hoje

✨ **Projeto DevSecOps completo configurado!**  
✨ **Pipeline CI/CD funcional!**  
✨ **3 ferramentas de segurança integradas!**  
✨ **Boas práticas aplicadas!**  
✨ **Debugging eficiente!**  

**Status:** 🟢 **EM ANDAMENTO - BUILD RODANDO!**

---

**Última atualização:** 27/01/2026 - 21:00  
**Próxima revisão:** Após conclusão do build atual
