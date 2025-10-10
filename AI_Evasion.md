# Skills Assessment: Feature Obfuscation Attack - Write-up

**Desafio:** Skills Assessment - Feature Obfuscation Attack
**Categoria:** AI Evasions / Machine Learning Security
**Dificuldade:** Intermediário
**Flag:** `HTB{f34tur3_0bfu5c4t10n_m45t3r3d}`

---

## 📋 Sumário Executivo

Este desafio explora técnicas de **adversarial attacks** contra modelos de classificação de texto, especificamente ataques de **feature obfuscation** (ofuscação de características). O objetivo é manipular classificadores Naive Bayes para inverter suas predições através da adição estratégica de palavras, demonstrando vulnerabilidades em sistemas de análise de sentimento.

O desafio é dividido em duas fases:
1. **White-box Attack:** Acesso completo ao modelo para inverter 10 reviews positivas → negativas
2. **Black-box Attack:** Apenas API de predição para inverter 10 reviews negativas → positivas

---

## 🎯 Objetivos

### Fase 1: White-Box Attack
- **Input:** 10 reviews de filmes com sentimento positivo
- **Objetivo:** Adicionar até 30 palavras para fazer o modelo classificar como negativo
- **Recursos:** Acesso completo ao modelo serializado (pickle)
- **Constraint:** Apenas adicionar palavras (append-only)

### Fase 2: Black-Box Attack
- **Input:** 10 reviews de filmes com sentimento negativo
- **Objetivo:** Adicionar até 40 palavras para fazer o modelo classificar como positivo
- **Recursos:** Apenas endpoint `/predict` para consultas
- **Constraint:** Apenas adicionar palavras (append-only)

---

## 🔍 Reconhecimento

### Análise Inicial

```bash
# Verificar health do servidor
curl -s "http://94.237.51.21:51153/health"
# {"service":"skills_assessment_lab","status":"healthy"}
```

### Endpoints Disponíveis

| Endpoint | Método | Descrição |
|----------|--------|-----------|
| `/health` | GET | Verifica status do servidor |
| `/challenge/whitebox` | GET | Retorna dados do desafio white-box |
| `/challenge/blackbox` | GET | Retorna dados do desafio black-box |
| `/model/download` | GET | Download do modelo (white-box) |
| `/predict` | POST | Predição de sentimento (black-box) |
| `/submit/whitebox` | POST | Submete soluções white-box |
| `/submit/blackbox` | POST | Submete soluções black-box |
| `/status` | GET | Status do progresso |

---

## 🛠️ Fase 1: White-Box Attack

### Estratégia

Com acesso completo ao modelo, podemos analisar os **log-probabilities** de cada feature (palavra) para identificar quais têm maior impacto negativo.

### Análise do Modelo

```python
import pickle
import numpy as np
import requests

BASE_URL = "http://94.237.51.21:51153"

# Download do modelo
r = requests.get(f"{BASE_URL}/model/download")
with open("model.pkl", "wb") as f:
    f.write(r.content)

# Carregar modelo
with open("model.pkl", "rb") as f:
    bundle = pickle.load(f)

classifier = bundle['classifier']  # MultinomialNB
vectorizer = bundle['vectorizer']
feature_names = bundle['feature_names']  # 89,972 features
```

### Encontrando Palavras Mais Negativas

O modelo Naive Bayes usa log-probabilities. A diferença entre as probabilidades das classes indica o "peso" de cada palavra:

```python
# feature_log_prob_[0] = classe negativa
# feature_log_prob_[1] = classe positiva
neg_log_probs = classifier.feature_log_prob_[0]
pos_log_probs = classifier.feature_log_prob_[1]

# Log-ratio: valores negativos = fortes indicadores de sentimento negativo
log_ratios = pos_log_probs - neg_log_probs

# Top palavras negativas
neg_indices = np.argsort(log_ratios)[:50]
negative_words = [feature_names[i] for i in neg_indices]
```

### Top Palavras Negativas Identificadas

```python
['hours life', 'boll', 'acting horrible', 'prom night',
 'worst movies seen', 'acting awful', 'worst films',
 'easily worst', 'br br worst', 'br worst', 'uwe',
 'worst acting', 'slater', 'avoid like', 'worst movies ve']
```

### Implementação do Ataque

```python
wb_challenge = requests.get(f"{BASE_URL}/challenge/whitebox").json()

wb_solutions = []
for review in wb_challenge['reviews']:
    original = review['text']
    augmented = original
    words_added = 0

    # Adicionar palavras negativas até mudar classificação
    for word in negative_words:
        if words_added >= 30:  # max_added_words
            break

        test = augmented + " " + word
        vec = vectorizer.transform([test])
        pred = classifier.predict(vec)[0]

        augmented = test
        words_added += 1

        if pred == 'negative':
            break  # Sucesso!

    wb_solutions.append({
        "id": review['id'],
        "augmented_text": augmented
    })

# Submeter
result = requests.post(
    f"{BASE_URL}/submit/whitebox",
    json={"solutions": wb_solutions}
).json()
```

### Resultados da Fase 1

✅ **10/10 reviews invertidas com sucesso**

| Review ID | Palavras Adicionadas | Status |
|-----------|---------------------|--------|
| wb_0 | 5 | ✓ |
| wb_1 | 3 | ✓ |
| wb_2 | 3 | ✓ |
| wb_3 | 3 | ✓ |
| wb_4 | 4 | ✓ |
| wb_5 | 6 | ✓ |
| wb_6 | 4 | ✓ |
| wb_7 | 4 | ✓ |
| wb_8 | 12 | ✓ |
| wb_9 | 5 | ✓ |

**Média:** 4.9 palavras por review (de 30 permitidas)

---

## 🛠️ Fase 2: Black-Box Attack

### Estratégia

Sem acesso ao modelo, precisamos usar uma abordagem **trial-and-error** inteligente:

1. Compilar vocabulário de palavras positivas fortes
2. Testar impacto individual de cada palavra
3. Adicionar iterativamente as mais efetivas
4. Monitorar mudança de probabilidade

### Vocabulário Positivo

```python
positive_words = [
    # Superlativos
    "masterpiece", "brilliant", "genius", "perfection", "magnificent",
    "spectacular", "extraordinary", "phenomenal", "outstanding", "exceptional",

    # Qualificadores fortes
    "superb", "excellent", "amazing", "wonderful", "fantastic",
    "incredible", "remarkable", "stunning", "breathtaking", "marvelous",
    "terrific", "fabulous", "splendid", "glorious", "impressive",

    # Estéticos
    "beautiful", "perfect", "great", "awesome", "lovely",
    "delightful", "charming", "gorgeous", "exquisite", "elegant",
    "sublime", "divine", "heavenly", "sensational", "dazzling",

    # Advérbios intensificadores
    "superbly", "wonderfully", "beautifully", "brilliantly", "perfectly",
    "timeless", "unforgettable", "captivated", "mesmerized", "finest"
]
```

### Implementação do Ataque

```python
def predict(text):
    r = requests.post(f"{BASE_URL}/predict", json={"text": text})
    return r.json()

bb_challenge = requests.get(f"{BASE_URL}/challenge/blackbox").json()

bb_solutions = []
for review in bb_challenge['reviews']:
    original = review['text']
    augmented = original
    words_added = 0

    # Adicionar palavras até inverter sentimento
    for word in positive_words:
        if words_added >= 40:  # max_added_words
            break

        augmented = augmented + " " + word
        words_added += 1

        result = predict(augmented)
        if result['label'] == 'positive':
            break  # Sucesso!

    bb_solutions.append({
        "id": review['id'],
        "augmented_text": augmented
    })

# Submeter
result = requests.post(
    f"{BASE_URL}/submit/blackbox",
    json={"solutions": bb_solutions}
).json()
```

### Desafio: bb_6 - A Review Mais Difícil

A review `bb_6` foi particularmente resistente. Análise especial foi necessária:

```
Original: "This is not a good film by an standards. It is very
          poorly written and the acting is just a little above
          amateur."
Baseline probability: 0.000000 (extremamente negativo)
```

**Solução:** Testar palavras individualmente e medir impacto:

```python
# Testar impacto de cada palavra
baseline = predict(original)
word_impacts = []

for word in positive_vocab:
    test_text = original + " " + word
    result = predict(test_text)
    impact = result['positive_probability'] - baseline['positive_probability']
    word_impacts.append((word, impact))

# Ordenar por maior impacto
word_impacts.sort(key=lambda x: x[1], reverse=True)
```

A review bb_6 precisou de **38 palavras** em sequência, com as últimas sendo críticas:

```
...dazzling → awesome → glorious → heavenly
```

Palavra "**heavenly**" foi a que finalmente inverteu para positivo:
- Após 37 palavras: `pos_prob = 0.489530` (ainda negativo)
- Após 38 palavras: `pos_prob = 0.732816` ✓ (positivo!)

### Resultados da Fase 2

✅ **10/10 reviews invertidas com sucesso**

| Review ID | Palavras Adicionadas | Dificuldade |
|-----------|---------------------|-------------|
| bb_0 | 18 | Média |
| bb_1 | 31 | Alta |
| bb_2 | 17 | Média |
| bb_3 | 14 | Média |
| bb_4 | 3 | Baixa |
| bb_5 | 29 | Alta |
| bb_6 | **38** | **Muito Alta** |
| bb_7 | 3 | Baixa |
| bb_8 | 11 | Baixa |
| bb_9 | 4 | Baixa |

**Média:** 16.8 palavras por review (de 40 permitidas)

---

## 🎌 Flag Capturada

```json
{
  "flag": "HTB{f34tur3_0bfu5c4t10n_m45t3r3d}",
  "message": "Congratulations! Both phases complete!",
  "phase_complete": true
}
```

### Status Final

```json
{
  "whitebox_completed": 10,
  "whitebox_required": 10,
  "blackbox_completed": 10,
  "blackbox_required": 10,
  "total_queries": 396
}
```

---

## 💡 Análise Técnica

### Por Que Este Ataque Funciona?

#### 1. **Modelo Bag-of-Words Ingênuo**

O Multinomial Naive Bayes trata texto como um "saco de palavras" sem considerar:
- Contexto
- Ordem das palavras
- Sintaxe
- Semântica

#### 2. **Independência de Features**

O modelo assume que cada palavra contribui independentemente para a classificação. Adicionar muitas palavras positivas "overwhelm" o sinal negativo original.

#### 3. **Log-Probability Additivity**

```
P(classe|texto) ∝ P(classe) × ∏ P(palavra_i|classe)

log P(classe|texto) = log P(classe) + Σ log P(palavra_i|classe)
```

Adicionar palavras com alto `log P(palavra|positive)` aumenta o score da classe positiva.

### Limitações do Ataque

1. **Append-Only:** Só podemos adicionar palavras, não modificar ou remover
2. **Budget Limitado:** 30-40 palavras máximo
3. **Detecção:** Textos resultantes são obviamente manipulados (reviews "Frankenstein")

---

## 🛡️ Defesas Possíveis

### 1. **Contexto e Sintaxe**

Usar modelos que entendem contexto:
- **LSTM/GRU:** Redes recorrentes que capturam sequência
- **Transformers (BERT, RoBERTa):** Atenção e contexto bidirecional
- **Sentiment Analysis Específico:** Modelos treinados para resistir a ruído

### 2. **Validação de Input**

```python
def validate_review(text, original):
    # Detectar append-only attacks
    if not text.startswith(original):
        return False

    added = text[len(original):].strip().split()

    # Limitar palavras adicionadas
    if len(added) > threshold:
        return False

    # Detectar repetição suspeita
    word_freq = Counter(added)
    if max(word_freq.values()) > 3:
        return False

    return True
```

### 3. **Ensemble Methods**

Combinar múltiplos modelos:
- Naive Bayes + BERT
- Votação entre modelos diversos
- Dificulta ataque universal

### 4. **Adversarial Training**

Treinar modelo com exemplos adversariais:

```python
# Gerar exemplos atacados durante treino
for review, label in training_data:
    # Original
    train_model(review, label)

    # + Versão atacada
    attacked = add_adversarial_words(review)
    train_model(attacked, label)  # Mesmo label!
```

### 5. **Perplexity Filtering**

Calcular "naturalidade" do texto:

```python
def is_natural(text):
    perplexity = language_model.perplexity(text)
    return perplexity < threshold  # Textos não-naturais têm alta perplexity
```

---

## 📊 Métricas de Ataque

### Eficiência do Ataque

| Métrica | White-Box | Black-Box |
|---------|-----------|-----------|
| **Taxa de Sucesso** | 100% (10/10) | 100% (10/10) |
| **Palavras Médias** | 4.9 | 16.8 |
| **Queries Necessárias** | 0 (offline) | 396 |
| **Tempo de Execução** | ~2s | ~45s |
| **Budget Usado** | 16.3% | 42.0% |

### Correlação: Dificuldade vs Palavras

```
Review Difficulty vs Words Needed (Black-Box)

40│                                            ● bb_6
  │
30│                          ● bb_1       ● bb_5
  │
20│               ● bb_0
  │          ● bb_2
10│     ● bb_3           ● bb_8
  │
 0│ ● bb_4, bb_7, bb_9
  └─────────────────────────────────────────────────
   Low        Medium        High        Very High
```

---

## 🔬 Experimentos Adicionais

### Teste: Ordem das Palavras Importa?

```python
# Teste 1: Ordem original
words = ["excellent", "amazing", "wonderful"]
success_rate_1 = test_attack(words)

# Teste 2: Ordem reversa
words = ["wonderful", "amazing", "excellent"]
success_rate_2 = test_attack(words)

# Resultado: Ordem NÃO importa para Naive Bayes (bag-of-words)
assert success_rate_1 == success_rate_2
```

### Teste: Palavras Compostas vs Simples

```python
# Palavras compostas têm maior impacto no white-box
"worst movies seen" > "worst" + "movies" + "seen"

# Razão: São tratadas como features únicas com log-probs mais extremos
```

---

## 📚 Lições Aprendidas

### 1. **Modelos Simples São Vulneráveis**

Naive Bayes, apesar de eficiente, é facilmente enganado por ataques de **feature stuffing**.

### 2. **White-Box vs Black-Box**

- **White-Box:** Mais eficiente (menos palavras), mas requer acesso ao modelo
- **Black-Box:** Menos eficiente (mais queries), mas mais realista

### 3. **Defense-in-Depth**

Segurança de ML requer múltiplas camadas:
- Modelo robusto
- Validação de input
- Monitoramento de anomalias
- Rate limiting

### 4. **Interpretabilidade vs Robustez**

Modelos interpretáveis (Naive Bayes) são mais fáceis de atacar. Trade-off entre:
- Explicabilidade
- Robustez
- Performance

---

## 🛠️ Scripts de Exploração

### Script Completo Final

```python
import requests
import pickle
import numpy as np

np.random.seed(1337)
BASE_URL = "http://94.237.51.21:51153"

# ============================================================
# PHASE 1: WHITE-BOX
# ============================================================

# Download model
r = requests.get(f"{BASE_URL}/model/download")
with open("model.pkl", "wb") as f:
    f.write(r.content)

with open("model.pkl", "rb") as f:
    bundle = pickle.load(f)

classifier = bundle['classifier']
vectorizer = bundle['vectorizer']
feature_names = bundle['feature_names']

# Find negative words
neg_log_probs = classifier.feature_log_prob_[0]
pos_log_probs = classifier.feature_log_prob_[1]
log_ratios = pos_log_probs - neg_log_probs
neg_indices = np.argsort(log_ratios)[:50]
negative_words = [feature_names[i] for i in neg_indices]

# Attack white-box reviews
wb_challenge = requests.get(f"{BASE_URL}/challenge/whitebox").json()
wb_solutions = []

for review in wb_challenge['reviews']:
    original = review['text']
    augmented = original

    for word in negative_words:
        test = augmented + " " + word
        vec = vectorizer.transform([test])
        pred = classifier.predict(vec)[0]
        augmented = test
        if pred == 'negative':
            break

    wb_solutions.append({
        "id": review['id'],
        "augmented_text": augmented
    })

# Submit white-box
wb_result = requests.post(
    f"{BASE_URL}/submit/whitebox",
    json={"solutions": wb_solutions}
).json()

# ============================================================
# PHASE 2: BLACK-BOX
# ============================================================

positive_words = [
    "superbly", "perfection", "wonderfully", "breathtaking",
    "delightful", "beautifully", "superb", "timeless",
    "unforgettable", "extraordinary", "brilliantly", "wonderful",
    "captivated", "magnificent", "finest", "fantastic",
    "terrific", "outstanding", "marvelous", "excellent",
    "splendid", "exceptional", "amazing", "elegant",
    "remarkable", "stunning", "divine", "perfect",
    "brilliant", "perfectly", "phenomenal", "sublime",
    "exquisite", "fabulous", "dazzling", "awesome",
    "glorious", "heavenly"
]

def predict(text):
    r = requests.post(f"{BASE_URL}/predict", json={"text": text})
    return r.json()

# Attack black-box reviews
bb_challenge = requests.get(f"{BASE_URL}/challenge/blackbox").json()
bb_solutions = []

for review in bb_challenge['reviews']:
    original = review['text']
    augmented = original

    for word in positive_words:
        augmented = augmented + " " + word
        result = predict(augmented)
        if result['label'] == 'positive':
            break

    bb_solutions.append({
        "id": review['id'],
        "augmented_text": augmented
    })

# Submit black-box
bb_result = requests.post(
    f"{BASE_URL}/submit/blackbox",
    json={"solutions": bb_solutions}
).json()

# FLAG!
print(f"\nFLAG: {bb_result['flag']}")
```

---

## 📖 Referências

### Acadêmicas

1. **Adversarial Examples in NLP**
   - Ebrahimi et al. (2018) - "HotFlip: White-Box Adversarial Examples for Text Classification"

2. **Robustness in Sentiment Analysis**
   - Alzantot et al. (2018) - "Generating Natural Language Adversarial Examples"

3. **Defense Mechanisms**
   - Jia & Liang (2017) - "Adversarial Examples for Evaluating Reading Comprehension Systems"

### Técnicas

- **Naive Bayes Classifier:** McCallum & Nigam (1998)
- **Feature Obfuscation:** Goodfellow et al. (2014) - Explaining and Harnessing Adversarial Examples
- **Adversarial Training:** Madry et al. (2018)

---

## 🏁 Conclusão

Este desafio demonstra de forma prática as vulnerabilidades de modelos de ML tradicionais a **ataques adversariais**. Através de feature obfuscation, conseguimos:

✅ Inverter 100% das classificações (20/20 reviews)
✅ Operar dentro das constraints do desafio
✅ Demonstrar a diferença entre white-box e black-box attacks
✅ Identificar estratégias de defesa

**Flag Capturada:** `HTB{f34tur3_0bfu5c4t10n_m45t3r3d}`

### Impacto no Mundo Real

Ataques similares são relevantes para:
- **Spam filters:** Evading detection
- **Content moderation:** Bypassing toxic content filters
- **Fraud detection:** Manipulating risk scores
- **Recommendation systems:** Gaming algorithms

A segurança de sistemas de ML requer constante atenção a adversarial robustness!

---

**Autor:** Yuri Braz
**Data:** 10 de Outubro de 2025
**Plataforma:** HackTheBox - AI Security Module
**Dificuldade:** ⭐⭐⭐ (Intermediário)

