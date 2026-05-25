# PR Review Static - Modo Análise Estática

Este documento descreve como usar o PR Review Static sem dependência de LLM (Large Language Models), usando SonarCloud como fonte de findings.

## Visão Geral

A versão com análise estática detecta problemas de código através do SonarCloud, sem custo de API próprio e sem dependência de serviços externos como OpenAI, Anthropic ou Google.

### O que funciona

✅ **Análise de Segurança**
- Detecção de SQL injection
- Detecção de XSS (Cross-Site Scripting)
- Identificação de credenciais hardcoded
- Desserialização insegura

✅ **Análise de Bugs**
- Referências potenciais a variáveis nulas/indefinidas
- Variáveis declaradas mas não utilizadas
- Funções assíncronas sem return
- Confusão entre atribuição e comparação
- Muitos parâmetros em função

✅ **Boas Práticas**
- Números mágicos (constantes sem nome)
- console.log/console.error em código de produção
- TODO/FIXME/HACK comments
- Blocos catch vazios

✅ **Métricas**
- Score de qualidade (0-100)
- Esforço estimado de revisão (1-5)
- Detecção de testes relevantes
- Resumo de preocupações de segurança

### O que não funciona como LLM

❌ Análise semântica complexa (requer entendimento contextual)
❌ Sugestões de refatoração inteligentes
❌ Detecção de bugs lógicos sofisticados
❌ Respostas a comentários interativos
❌ Geração de títulos e descrições otimizadas

## Configuração

### Opção 1: GitHub Actions (Recomendado)

Crie ou atualize `.github/workflows/pr-review-sonarcloud.yml`:

```yaml
name: PR Review Static - SonarCloud Analysis

permissions:
  contents: read
  pull-requests: write
  issues: write

on:
  pull_request_target:
    types: [opened, synchronize]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5

      - name: Run SonarCloud scan
        uses: SonarSource/sonarqube-scan-action@v6
        env:
          SONAR_TOKEN: ${{ secrets.SONARCLOUD_TOKEN }}
          SONAR_HOST_URL: https://sonarcloud.io
        with:
          args: >-
            -Dsonar.organization=${{ secrets.SONARCLOUD_ORGANIZATION }}
            -Dsonar.projectKey=${{ secrets.SONARCLOUD_PROJECT_KEY }}
            -Dsonar.pullrequest.key=${{ github.event.pull_request.number }}
            -Dsonar.pullrequest.branch=${{ github.head_ref }}
            -Dsonar.pullrequest.base=${{ github.base_ref }}

      - name: Wait for SonarCloud quality gate
        uses: SonarSource/sonarqube-quality-gate-action@v1
        env:
          SONAR_TOKEN: ${{ secrets.SONARCLOUD_TOKEN }}
          SONAR_HOST_URL: https://sonarcloud.io

      - uses: thallesasv/pullRequestSonarCloudCodeReview@main
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SONARCLOUD_TOKEN: ${{ secrets.SONARCLOUD_TOKEN }}
          SONARCLOUD_ORGANIZATION: ${{ secrets.SONARCLOUD_ORGANIZATION }}
          SONARCLOUD_PROJECT_KEY: ${{ secrets.SONARCLOUD_PROJECT_KEY }}
          SONARCLOUD_URL: https://sonarcloud.io
```

**Configuração recomendada:**
- `GITHUB_TOKEN` para autenticação local

### Opção 2: Execução Local

```bash
# Instalar dependências
npm install

# Executar a revisão com análise estática padrão
npm run review

# Ou com Node.js direto
node dist/cli.js
```

### Variáveis de Ambiente

| Variável | Valor Padrão | Descrição |
|----------|-------------|-----------|
| `GITHUB_TOKEN` | - | Token do GitHub (obrigatório) |
| `DRY_RUN` | `"false"` | Use `"true"` para modo simulação |

## Suporte de Linguagens

### 🎯 Cobertura por Linguagem

A análise estática funciona com múltiplas linguagens:

| Linguagem | Suporte | Detecções |
|-----------|---------|----------|
| **JavaScript/TypeScript** | ✅ Excelente | Todos os padrões otimizados |
| **Python** | ✅ Bom | SQL injection, secrets, eval, console |
| **Java** | ✅ Bom | SQL injection, secrets, eval, catch vazio |
| **C#** | ✅ Bom | SQL injection, secrets, eval, console |
| **Go** | ✅ Bom | SQL injection, secrets, eval |
| **PHP** | ✅ Bom | SQL injection, secrets, eval, XSS |
| **Ruby** | ✅ Bom | SQL injection, secrets, eval |
| **Outras** | ⚠️ Básico | Apenas padrões genéricos |

### Detalhes de Suporte

#### ✅ Excelente (JavaScript/TypeScript)

Otimizado para JS/TS com detecção de:
- `const`/`var`/`let` declarations
- `async/await` patterns
- Arrow functions (`=>`)
- DOM operations (`innerHTML`, `dangerouslySetInnerHTML`)
- Browser APIs (`document.write`, `console.log`)
- Null/undefined reference checks

**Exemplo:**
```typescript
// ✅ Detecta todos esses problemas
const unused = fetchData()         // variável não usada
async function test() { }          // sem return
element.innerHTML = user.data      // XSS
```

#### ✅ Bom (Python, Java, C#, Go, PHP, Ruby)

Detecção de padrões genéricos:
- **SQL Injection** - `query(` com template strings/concatenação
- **Hardcoded Secrets** - `password`, `token`, `api_key` com valores
- **Unsafe Deserialization** - `eval`, `pickle.load`, `yaml.load`, `JSON.parse`
- **Console/Debug Statements** - `console.log`, `print`, `println`
- **Empty Exception Handlers** - `except:` ou `catch` vazios
- **Magic Numbers** - números sem constante

**Exemplos:**
```python
# ✅ Python - Detecta
query = f"SELECT * FROM users WHERE id = {user_id}"  # SQL injection
API_KEY = "sk-1234567890"                             # hardcoded secret
eval(user_input)                                       # unsafe eval
```

```java
// ✅ Java - Detecta
String query = "SELECT * FROM users WHERE id = " + userId;  // SQL injection
String password = "admin123";                               // hardcoded
eval(userCode);                                             // eval
```

#### ⚠️ Básico (Outras Linguagens)

Detecção apenas de padrões extremamente genéricos que funcionam em qualquer linguagem:
- Hardcoded credentials (qualquer linguagem com `password`/`token`)
- `eval` ou `execute` patterns
- Números muito grandes sem contexto

### Estender para Sua Linguagem

Se sua linguagem não tem suporte total, você pode adicionar análises específicas editando `src/static-analysis.ts`:

```typescript
// Adicionar após as funções existentes

function analyzePythonPatterns(hunk: Hunk, filename: string): AIComment[] {
  const comments: AIComment[] = [];
  
  // Detecção específica para Python
  if (/except\s*:\s*pass/m.test(hunk.diff)) {
    comments.push({
      file: filename,
      start_line: 10,
      end_line: 10,
      highlighted_code: "except: pass",
      header: "Bloco except vazio detectado",
      content: "Except blocks sem tratamento ocultam erros. Adicione logging ou re-lance.",
      label: "best practice",
      critical: true,
    });
  }
  
  return comments;
}

function analyzeJavaPatterns(hunk: Hunk, filename: string): AIComment[] {
  const comments: AIComment[] = [];
  
  // Detecção específica para Java
  if (/catch\s*\(\w+\s+\w+\)\s*\{\s*\}/m.test(hunk.diff)) {
    comments.push({
      file: filename,
      start_line: 10,
      end_line: 10,
      highlighted_code: "catch(Exception e) {}",
      header: "Bloco catch vazio",
      content: "Catch blocks sem tratamento ocultam exceções.",
      label: "best practice",
      critical: true,
    });
  }
  
  return comments;
}

// No performStaticAnalysis(), antes de retornar:
const language = detectLanguage(file.filename);
if (language === 'python') {
  const pythonIssues = analyzePythonPatterns(hunk, file.filename);
  allComments.push(...pythonIssues);
} else if (language === 'java') {
  const javaIssues = analyzeJavaPatterns(hunk, file.filename);
  allComments.push(...javaIssues);
}
```

### Detectar Linguagem do Arquivo

Usamos a extensão do arquivo para identificar a linguagem:

```typescript
function detectLanguage(filename: string): string {
  if (/\.(ts|tsx|js|jsx)$/.test(filename)) return "typescript";
  if (/\.py$/.test(filename)) return "python";
  if (/\.java$/.test(filename)) return "java";
  if (/\.cs$/.test(filename)) return "csharp";
  if (/\.go$/.test(filename)) return "go";
  if (/\.php$/.test(filename)) return "php";
  if (/\.rb$/.test(filename)) return "ruby";
  return "generic";
}
```

## Exemplo de Saída

### Resumo do PR
```
## Resumo do PR

Atualizar 2 arquivo(s).

### Alterações

| Arquivo | Resumo |
|---------|--------|
| `src/api/users.ts` | Modificado. Alterações em 3 trecho(s) com aproximadamente 45 linha(s). |
| `src/api/auth.ts` | Modificado. Alterações em 1 trecho(s) com aproximadamente 12 linha(s). |
```

### Resultado da Análise

```
### Pontos de Ação (2)

- File: src/api/users.ts [25]
  > security: "Possível vulnerabilidade XSS detectada"
  
  Análise: innerHTML ou dangerouslySetInnerHTML encontrado no código. 
  Isso pode permitir injeção de código malicioso. Use textContent ao invés.

- File: src/api/auth.ts [15]
  > possible bug: "Possível referência a variável nula"
  
  Análise: A variável pode ser nula ou indefinida. Considere adicionar uma 
  verificação antes de acessar suas propriedades.
```

## Estendendo a Análise SonarCloud

Para adicionar novas regras de análise, ajuste o Quality Profile e as regras habilitadas no SonarCloud do seu projeto. Se você precisar adaptar a forma como os findings são convertidos em comentários, faça isso em [src/static-analysis.ts](src/static-analysis.ts).

Exemplo de configuração do fluxo:

```bash
SONARCLOUD_TOKEN=seu_token
SONARCLOUD_ORGANIZATION=sua_organizacao
SONARCLOUD_PROJECT_KEY=seu_projeto
SONARCLOUD_URL=https://sonarcloud.io
```

Se você preferir regras próprias, configure uma Quality Profile dedicada no SonarCloud para o seu projeto.

## Limites e Considerações

1. **Falsos Positivos** - Algumas regras podem sinalizar trechos corretos como suspeitos
2. **Falsos Negativos** - Bugs complexos podem não ser detectados por regras estáticas
3. **Sem Contexto Total** - O SonarCloud vê padrões locais e métricas de qualidade, não a intenção de negócio completa
4. **Cobertura por linguagem** - O resultado depende da maturidade das regras e da linguagem analisada

## Troubleshooting

**P: Por que não há comentários nos arquivos?**
A: Verifique se o SonarCloud está autenticado no runner, se o workflow faz checkout do repositório e se há findings nas linhas alteradas.

**P: Como contribuir com melhorias na análise estática?**
A: Ajuste a Quality Profile do SonarCloud, adicione regras próprias no projeto ou refine o adaptador em [src/static-analysis.ts](src/static-analysis.ts).

Se você quiser mudar as regras, ajuste a configuração do SonarCloud no projeto ou edite o valor padrão em [src/static-analysis.ts](src/static-analysis.ts).

## Performance

- Tempo de análise: depende do tamanho do PR e do conjunto de regras do SonarCloud
- Sem custos de API
- Sem limites de chamadas a serviços externos
- Funciona offline
