# methods_techniques_risk_relative
Repositório com a descrição dos principais métodos e técnicas utilizadas nesse projeto

<details>

<summary> <h2> 1. Risco Relativo </h2></summary>

Para o caso desse projeto, será analisado se, por exemplo, para uma variável como a idade, uma determinada faixa etária apresenta maior risco de inadimplência que outra, identificando se certos grupos são mais propensos a não cumprir com suas obrigações financeiras.  Esses grupos são os quartis da amostra, segmentados previamente como mostrado em **Cálculo dos quartis**: 

<details> <summary> <h4> Cálculo dos quartis </h4>  - Clique em ▶ para ver os detalhes </summary> 

****

Aqui, foi obtida a segmentação das seguintes variáveis em quartis:

**age | last_month_salary | number_dependents | total_loan | total_90_days_overdue | using lines not secured personal assets | debt_ratio**

O código utilizado para a segmentação foi o seguinte:

```sql
  SELECT *,
    NTILE(4) OVER (ORDER BY age) AS age_quartiles,
    NTILE(4) OVER (ORDER BY last_month) AS last_month_quartiles,
    NTILE(4) OVER (ORDER BY number_dependents) AS number_dependents_quartiles,
    NTILE(4) OVER (ORDER BY total_loan) AS total_loan_quartiles,
    NTILE(4) OVER (ORDER BY total_90_days_overdue) AS total_90_days_overdue_quartiles,
    NTILE(4) OVER (ORDER BY total_lines) AS total_lines_quartiles,
    NTILE(4) OVER (ORDER BY debt_ratio) AS debt_ratio_quartiles
  FROM `dados_proj3.proj3_unified_table2`
```
</details>

O cálculo do risco relativo será feito para cada quartil, conforme a fórmula:

$$
𝑅𝑅_{Q_n}= \tau_{Q_n}/\tau_{Q_x, Q_y, Q_z}
$$

Onde, 

$\tau_{Q_n}$ é a taxa de inadimplência do quartil *n*

$\tau_{Q_x, Q_y, Q_z}$ é a taxa de inadimplência dos outros quartis.

Para o cálculo de Risco Relativo no Big Query, foi utilizado o seguinte raciocínio:

$\tau_{Q_1}$ é calculado como a média da variável *default_flag* no quartil *n*. Como a variável *default_flag* tem valor 1 para inadimplentes e 0 para adimplentes, a soma dessa variável dentro do quartil 1, dividida pelo número de observações no quartil, nos dá a taxa de inadimplência para aquele grupo:

$$
\tau_{Q_n}= media(\textup{default flag(Qn)})
$$

e

$$
\tau_{Q_x, Q_y, Q_z}= \frac{\sum (\textup{default flag}) - \Sigma (\textup{default flag (Qn)})}{N-N_{Qn}}
$$

Onde,

- $\sum (\textup{default flag})$ é a soma total dos valores da variável default_flag.
- $\Sigma (\textup{default flag (Qn)}$ é a soma dos valores da variável default_flag no quartil n.
- $N$ é o número total de observações na amostra.
- $N_{Qn}$ é o número de observações no quartil *n*.

Esse cálculo nos permitirá comparar a taxa de inadimplência do quartil n com os outros quartis, ajudando a identificar se existe um grupo de cada variável com maior risco de inadimplência.

O risco relativo e a classificação dos grupos foram feitas em um mesmo código e será melhor detalhado abaixo:

```sql
-- Dividindo os dados em quartis com NTILE
WITH quartiles_table AS (
  SELECT *,
    NTILE(4) OVER (ORDER BY age) AS age_quartiles,
    NTILE(4) OVER (ORDER BY last_month) AS last_month_quartiles,
    NTILE(4) OVER (ORDER BY number_dependents) AS number_dependents_quartiles,
    NTILE(4) OVER (ORDER BY total_loan) AS total_loan_quartiles,
    NTILE(4) OVER (ORDER BY total_90_days_overdue) AS total_90_days_overdue_quartiles,
    NTILE(4) OVER (ORDER BY total_lines) AS total_lines_quartiles,
    NTILE(4) OVER (ORDER BY debt_ratio) AS debt_ratio_quartiles
  FROM `dados_proj3.proj3_unified_table2`
),

total_measurements AS (
  SELECT
    SUM(default_flag) AS sum_default,
    COUNT(*) AS total
  FROM quartiles_table
),

age_relative_risks AS (
  SELECT
    age_quartiles AS quartile,
    min(age) AS lower_age,
    max(age) AS upper_age,
    AVG(default_flag) / ((total_measurements.sum_default - SUM(default_flag)) / (total_measurements.total - COUNT(*))) AS relative_risk
  FROM quartiles_table, total_measurements
  GROUP BY age_quartiles, total_measurements.sum_default, total_measurements.total
),

last_month_relative_risks AS (
  SELECT
    last_month_quartiles AS quartile,
    min(last_month) AS lower_last_month,
    max(last_month) AS upper_last_month,
    AVG(default_flag) / ((total_measurements.sum_default - SUM(default_flag)) / (total_measurements.total - COUNT(*))) AS relative_risk
  FROM quartiles_table, total_measurements
  GROUP BY last_month_quartiles, total_measurements.sum_default, total_measurements.total
),

number_dependents_relative_risks AS (
  SELECT
    number_dependents_quartiles AS quartile,
    min(number_dependents) AS lower_number_dependents,
    max(number_dependents) AS upper_number_dependents,
    AVG(default_flag) / ((total_measurements.sum_default - SUM(default_flag)) / (total_measurements.total - COUNT(*))) AS relative_risk
  FROM quartiles_table, total_measurements
  GROUP BY number_dependents_quartiles, total_measurements.sum_default, total_measurements.total
),

total_loan_relative_risks AS (
  SELECT
    total_loan_quartiles AS quartile,
    min(total_loan) AS lower_total_loan,
    max(total_loan) AS upper_total_loan,
    AVG(default_flag) / ((total_measurements.sum_default - SUM(default_flag)) / (total_measurements.total - COUNT(*))) AS relative_risk
  FROM quartiles_table, total_measurements
  GROUP BY total_loan_quartiles, total_measurements.sum_default, total_measurements.total
),

total_90_days_overdue_relative_risks AS (
  SELECT
    total_90_days_overdue_quartiles AS quartile,
    min(total_90_days_overdue) AS lower_total_90_days_overdue,
    max(total_90_days_overdue) AS upper_total_90_days_overdue,
    AVG(default_flag) / ((total_measurements.sum_default - SUM(default_flag)) / (total_measurements.total - COUNT(*))) AS relative_risk
  FROM quartiles_table, total_measurements
  GROUP BY total_90_days_overdue_quartiles, total_measurements.sum_default, total_measurements.total
),

total_lines_relative_risks AS (
  SELECT
    total_lines_quartiles AS quartile,
    min(total_lines) AS lower_total_lines,
    max(total_lines) AS upper_total_lines,
    AVG(default_flag) / ((total_measurements.sum_default - SUM(default_flag)) / (total_measurements.total - COUNT(*))) AS relative_risk
  FROM quartiles_table, total_measurements
  GROUP BY total_lines_quartiles, total_measurements.sum_default, total_measurements.total
),

debt_ratio_relative_risks AS (
  SELECT
    debt_ratio_quartiles AS quartile,
    min(debt_ratio) AS lower_debt_ratio,
    max(debt_ratio) AS upper_debt_ratio,
    AVG(default_flag) / ((total_measurements.sum_default - SUM(default_flag)) / (total_measurements.total - COUNT(*))) AS relative_risk
  FROM quartiles_table, total_measurements
  GROUP BY debt_ratio_quartiles, total_measurements.sum_default, total_measurements.total
)

SELECT 
  a.quartile AS quartiles,
  a.relative_risk AS age_relative_risk,
  a.lower_age,
  a.upper_age,
  CASE
    WHEN a.relative_risk < 1 THEN 'Grupo que não apresenta risco'
    WHEN a.relative_risk > 1 THEN 'Grupo de Risco'
  END AS age_classification,

  b.relative_risk AS last_month_relative_risk,
  b.lower_last_month,
  b.upper_last_month,
  CASE
    WHEN b.relative_risk < 1 THEN 'Grupo que não apresenta risco'
    WHEN b.relative_risk > 1 THEN 'Grupo de Risco'
  END AS last_month_classification,

  c.relative_risk AS number_dependents_relative_risk,
  c.lower_number_dependents,
  c.upper_number_dependents,
  CASE
    WHEN c.relative_risk < 1 THEN 'Grupo que não apresenta risco'
    WHEN c.relative_risk > 1 THEN 'Grupo de Risco'
  END AS number_dependents_classification,

  d.relative_risk AS total_loan_relative_risk,
  d.lower_total_loan,
  d.upper_total_loan,
  CASE
    WHEN d.relative_risk < 1 THEN 'Grupo que não apresenta risco'
    WHEN d.relative_risk > 1 THEN 'Grupo de Risco'
  END AS total_loan_classification,
  
  e.relative_risk AS total_90_days_overdue_relative_risk,
  e.lower_total_90_days_overdue,
  e.upper_total_90_days_overdue,
  CASE
    WHEN e.relative_risk < 1 THEN 'Grupo que não apresenta risco'
    WHEN e.relative_risk > 1 THEN 'Grupo de Risco'
  END AS total_90_days_overdue_classification,

  f.relative_risk AS total_lines_relative_risk,
  f.lower_total_lines,
  f.upper_total_lines,
  CASE
    WHEN f.relative_risk < 1 THEN 'Grupo que não apresenta risco'
    WHEN f.relative_risk > 1 THEN 'Grupo de Risco'
  END AS total_lines_classification,
  
  g.relative_risk AS debt_ratio_relative_risk,
  g.lower_debt_ratio,
  g.upper_debt_ratio,
  CASE
    WHEN g.relative_risk < 1 THEN 'Grupo que não apresenta risco'
    WHEN g.relative_risk > 1 THEN 'Grupo de Risco'
  END AS debt_ratio_classification

FROM 
  age_relative_risks a
  JOIN last_month_relative_risks b ON a.quartile = b.quartile
  JOIN number_dependents_relative_risks c ON a.quartile = c.quartile
  JOIN total_loan_relative_risks d ON a.quartile = d.quartile
  JOIN total_90_days_overdue_relative_risks e ON a.quartile = e.quartile
  JOIN total_lines_relative_risks f ON a.quartile = f.quartile
  JOIN debt_ratio_relative_risks g ON a.quartile = g.quartile
ORDER BY a.quartile;

```

Embaixo, detalho as tabelas auxiliares feitas e o que elas representam:

**quartiles_table:**

Nessa tabela foi obtida a segmentação em quartis das oito variáveis (**age | last_month_salary | number_dependents | total_loan | total_90_days_overdue | using lines not secured personal assets | debt_ratio).**

**total_measurements:**

Nessa tabela, foi feita a soma da variável default_flag ($\sum (\textup{default flag})$) e a contagem do total de linhas da tabela unificada (Para o cálculo de $*N*$. 

```sql
total_measurements AS (
  SELECT
    SUM(default_flag) AS sum_default,
    COUNT(*) AS total
  FROM quartiles_table
),
```

Onde,

SUM(default_flag) AS sum_default é $\sum (\textup{default flag})$

COUNT(*) AS total é $N$

***variável*_risks:**

Foram construídas um total de oito tabelas auxiliares, uma para cada variável para o cálculo do risco relativo de cada quartil. Ex.:

```sql
age_relative_risks AS (
  SELECT
    age_quartiles AS quartile,
    a.lower_age,
	  a.upper_age,
    AVG(default_flag) / ((total_measurements.sum_default - SUM(default_flag)) / (total_measurements.total - COUNT(*))) AS relative_risk
  FROM quartiles_table, total_measurements
  GROUP BY age_quartiles, total_measurements.sum_default, total_measurements.total
),
```

Onde,

- AVG(default_flag) é $\tau_{Q_n}= media(\textup{default flag(Qn)})$
- ((total_measurements.sum_default - SUM(default_flag)) / (total_measurements.total - COUNT(*))) é $\tau_{Q_x, Q_y, Q_z}$
- AVG(default_flag) / ((total_measurements.sum_default - SUM(default_flag)) / (total_measurements.total - COUNT(*))) AS relative_risk é $RR$

No final, foram agrupadas todas as tabelas pelos quartiles de cada um, usando o comando JOIN e os $RR$ de cada variável foi classificado de acordo com essa tabela:

| RR | Interpretação |
| --- | --- |
| =1 | ausência de associação |
| <1 | fator de proteção |
| >1 | fator de risco |

Usando o comando CASE. Ex:

```sql
  CASE
    WHEN a.relative_risk < 1 THEN 'Sem Risco de Inadimplência'
    WHEN a.relative_risk > 1 THEN 'Risco de Inadimplência'
  END AS age_classification,
```
**Tabela com a segmentação dos grupos de acordo com cada variável:
| quartis | last_month relative_risk | last_month (min) | last_month (max) | last_month classification | number_dependents relative_risk | number_dependents (min) | number_dependents (max) | number_dependents classification | total_lines relative_risk | total_lines (min) | total_lines (max) | total_lines classification | debt_ratio relative_risk | debt_ratio (min) | debt_ratio (max) | debt_ratio classification |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | 1,949529054 | 0 | R$3944,00 | Risco de Inadimplência | 0,658798838 | 0 | 0 | Sem Risco de Inadimplência | 0,039086483 | 0 | 0,028829832 | Sem Risco de Inadimplência | 0,746959956 | 0 | 0,181105577 | Sem Risco de Inadimplência |
| 2 | 0,961747393 | R$3947,00 | R$5400,00 | Sem Risco de Inadimplência | 0,769668123 | 0 | 0 | Sem Risco de Inadimplência | 0,004830737 | 0,028830012 | 0,144361012 | Sem Risco de Inadimplência | 0,823739618 | 0,181122062 | 0,369246406 | Sem Risco de Inadimplência |
| 3 | 0,987142489 | R$5400,00 | R$7495,00 | Sem Risco de Inadimplência | 1,101057634 | 0 | 1 | Risco de Inadimplência | 0,146706137 | 0,144373673 | 0,529282357 | Sem Risco de Inadimplência | 1,453406147 | 0,369278625 | 0,881271543 | Risco de Inadimplência |
| 4 | 0,392771434 | R$7495,00 | R$1560100,00 | Sem Risco de Inadimplência | 1,596238587 | 1 | 13 | Risco de Inadimplência | 46,1104476 | 0,52937284 | 22000 | Risco de Inadimplência | 1,047840157 | 0,881365417 | 307001 | Risco de Inadimplência |
</details>

<details> <summary> <h2>2. Modelo de Classificação Risco de Crédito Baseado no Risco Relativo </h2> </summary>

### Classificação com base nos Scores

Aqui, o objetivo é classificar os clientes como  “Mau pagador” e “Bom pagador”  de acordo com todas as variáveis disponíveis no banco de dados. Essa classificação, tem como propósito modelar meu banco de dados para que se possa avaliar futuros clientes de acordo com as diferentes variáveis do meu modelo.

Afim de obter essa classificação, as categorias “Mau pagador” e “Bom pagador” de cada uma das 7 variáveis foi convertida em variáveis Dummies:

- Mau pagador = 1
- Bom pagador = 0

Parte do código, somente com as classificações:

```sql
SELECT 
  a.user_id,
  a.default_flag,
  h.relative_risk AS age_relative_risk,
  a.age_quartiles,
  h.lower_age,
  h.upper_age,
  CASE
    WHEN h.relative_risk < 1 THEN 0 ELSE 1
  END AS age_risk_dummy,

  b.relative_risk AS last_month_relative_risk,
  a.last_month_quartiles,
  b.lower_last_month,
  b.upper_last_month,
  CASE
    WHEN b.relative_risk < 1 THEN 0 ELSE 1
  END AS last_month_risk_dummy,

  c.relative_risk AS number_dependents_relative_risk,
  a.number_dependents_quartiles,
  c.lower_number_dependents,
  c.upper_number_dependents,
  CASE
    WHEN c.relative_risk < 1 THEN 0 ELSE 1
  END AS number_dependents_risk_dummy,

  d.relative_risk AS total_loan_relative_risk,
  a.total_loan_quartiles,
  d.lower_total_loan,
  d.upper_total_loan,
  CASE
    WHEN d.relative_risk < 1 THEN 0 ELSE 1
  END AS total_loan_risk_dummy,
  
  e.relative_risk AS total_90_days_overdue_relative_risk,
  a.total_90_days_overdue_quartiles,
  e.lower_total_90_days_overdue,
  e.upper_total_90_days_overdue,
  CASE
    WHEN e.relative_risk < 1 THEN 0 ELSE 1
  END AS total_90_days_overdue_risk_dummy,

  f.relative_risk AS total_lines_relative_risk,
  a.total_lines_quartiles,
  f.lower_total_lines,
  f.upper_total_lines,
  CASE
    WHEN f.relative_risk < 1 THEN 0 ELSE 1
  END AS total_lines_risk_dummy,
  
  g.relative_risk AS debt_ratio_relative_risk,
  a.debt_ratio_quartiles,
  g.lower_debt_ratio,
  g.upper_debt_ratio,
  CASE
    WHEN g.relative_risk < 1 THEN 0 ELSE 1
  END AS debt_ratio_risk_dummy

FROM 
  quartiles_table a
  JOIN age_relative_risks h ON a.age_quartiles = h.quartile
  JOIN last_month_relative_risks b ON a.last_month_quartiles = b.quartile
  JOIN number_dependents_relative_risks c ON a.number_dependents_quartiles = c.quartile
  JOIN total_loan_relative_risks d ON a.total_loan_quartiles = d.quartile
  JOIN total_90_days_overdue_relative_risks e ON a.total_90_days_overdue_quartiles = e.quartile
  JOIN total_lines_relative_risks f ON a.total_lines_quartiles = f.quartile
  JOIN debt_ratio_relative_risks g ON a.debt_ratio_quartiles = g.quartile
),
```

Assim, foi possível somar os valores dummies, no qual o score final, chamado de risk_score, poderia ter um valor de de 0 a 7.

```sql
-- Calculando a pontuação final
final_scores AS (
  SELECT *,
    (age_risk_dummy + last_month_risk_dummy + number_dependents_risk_dummy + total_loan_risk_dummy + total_90_days_overdue_risk_dummy + total_lines_risk_dummy + debt_ratio_risk_dummy) AS risk_score
  FROM risk_classification
),
```

 Com esses valores em mãos foi estabelecido um valor de corte para a variável risk_score, no qual, acima desse valor, o cliente seria classificado como “mau pagador” e abaixo como “bom pagador”.

O valor de corte foi determinado pelo valor da taxa de inadimplência para cada um dos possíveis risk_score. A taxa de inadimplência ($\tau_i$ ) foi calculada dividindo o somatório do *default_flag* para cada risk_score pelo quantidade total de valores da variável *default_flag*. 

$$
\tau_i=\frac{\sum(\text{default flag)}_i}{N}
$$

Onde, 

- $N$ é o número total de observações na amostra.

Código:

```sql
-- Calcular a taxa de inadimplência para cada faixa de pontuação
risk_distribution AS (
  SELECT
    risk_score,
    --COUNT(*) AS total_users,
    --SUM(default_flag) AS total_defaults,
    SUM(default_flag) / COUNT(*) AS default_rate
  FROM final_scores
  GROUP BY risk_score
  ORDER BY risk_score
),
```

Com esse cálculo temos a taxa de inadimplência para cada grupo de risk_score. Os valores encontrados, se encontram na tabela abaixo:

| risk_score | $\tau_i $ |
| --- | --- |
| 1 | 0.00018601190476190475 |
| 2 | 0.0019365250134480904 |
| 3 | 0.00579496364977347 |
| 4 | 0.022197201222454561 |
| 5 | 0.07903780068728522 |
| 6 | 0.17396449704142011 |
| 7 | 0.255813953488 |

Baseado no valor das taxas $\tau_i$  pra 

Nesse código foi utilizado o seguinte critério de classificação:

taxa > 0.1:  “Mau pagador” 

Taxa < 0.1: “Bom pagador”

Essa classificação foi adicionada na tabela como a variável ***risk_class.***

```sql
classified_users AS (
  SELECT risk_score,
    default_rate,
    CASE
      WHEN default_rate >= 0.1 THEN 'Mau Pagador'
      ELSE 'Bom Pagador'
    END AS risk_class
  FROM risk_distribution
)

-- Visualizar a distribuição e taxa de inadimplência para cada faixa de pontuação
SELECT *
FROM final_scores final
JOIN  classified_users class ON final.risk_score = class.risk_score

```



### Validação do Modelo

Agora, para testar se essa classificação prevê bem o risco de crédito, foi utilizado a matriz de confusão no Google Colab.

A matriz de confusão é uma métrica de performance dos algoritmos de classificação. Vamos utilizá-la para testar esse modelo:

No Google Colab, exportei os dados da tabela com todas as variáveis dummy, assim como o risk_score e a variável  ***risk_class*** com a classificação do cliente como “Bons pagadores” e “Maus pagadores”

![1_ql1siLJsgfdmnabD4Jy8kg](https://github.com/annesantos1990/limpezadedados_spotify/assets/166059836/c7f6cdce-ed99-451b-8aa4-d416aa8a14ca)

Feito isso, a variável risk_class foi reclassificada em valores numéricas, onde Mau Pagador recebeu valor 1 e Bons Pagadores recebeu valor 0:

```python
from sklearn.metrics import confusion_matrix, classification_report
import seaborn as sns
import matplotlib.pyplot as plt

# Definir o ponto de corte e calcular a coluna 'predicted_default'
cut_off = 3  # Ajuste conforme necessário
df3['predicted_default'] = df3['risk_class'].apply(lambda x: 1 if x == "Mau Pagador" else 0)
```

E essa nova variável, no qual tem as informações do nosso modelo de classificação foi comparada com a variável default_flag que já é a variável com as informações dos adimplentes e inadimplentes.

Assim, podemos avaliar se o modelo de classificação construído para detectar “Bons Pagadores” e “Maus Pagadores” é um bom modelo.

Para avaliar o desempenho do modelo, foi utilizado a matriz de confusão, que fornece informações sobre a precisão das previsões em relação aos valores reais. A classificação é separada em valores positivos e negativos. Para esclarecer melhor, vejamos alguns exemplos:

- Verificar se o modelo detecta corretamente uma pessoa tem câncer:
    - Tem câncer (positivo), não tem câncer (negativo)
- Detecção de cachorro e gatos em fotografias:
    - gato (positivo), cachorro (negativo - não é um gato)

Como pode-se observar, o critério para determinar o que é positivo ou negativo varia conforme o caso. No entanto, o negativo pode ser interpretado como a negação do positivo.

Para o nosso caso, consideramos que positivo é um “bom pagador” e negativo é um “mau pagador”.

- Falsos negativos seriam bons pagadores sendo classificados como maus pagadores;
- Falsos positivo seriam maus pagadores sendo classificados como bons pagadores;

Junto com a matriz de confusão também tem outras métricas que também são avaliadas como:

- Acurácia: A proporção de todas as previsões corretas (positivas e negativas) em relação ao total de previsões;;
- Precisão: A proporção de verdadeiros positivos em relação ao total de previsões positivas. Isto mede a exatidão das previsões positivas do modelo;
- Recall: A proporção de verdadeiros positivos em relação ao total de casos realmente positivos. Isto mede a capacidade do modelo de identificar corretamente todos os casos positivos;
- F1-score: Uma medida que equilibra a precisão e o recall, calculada como a média harmônica dessas duas métricas.

Para o meu modelo de classificação, é importante que eu diminua os falsos positivos ou falsos negativos?

- Falsos positivos: Mal pagador sendo classificado como Bom pagador
- Falsos positivos: Bom pagador sendo classificado como Mau pagador

Se eu quero reduzir a probabilidade de inadimplências e perdas, temos que focar em diminuir os falsos positivos.

![1_ql1siLJsgfdmnabD4Jy8kg.jpg](https://prod-files-secure.s3.us-west-2.amazonaws.com/75dcc154-fd1d-4af5-b505-98330c057cd2/7e1c104d-a06d-45e7-a476-33fc1991848a/1_ql1siLJsgfdmnabD4Jy8kg.jpg)

- Assim, o **ppv** (**Positive Predictive Value (PPV) ou precisão)** precisa dar alto, porque teremos menos falsos positivos, aumentando a precisão positiva
- E o **recall da variável negativa** precisa  ser elevado, porque, entre todos os valores verdadeiramente negativos, precisamos que haja poucos falsos positivos (significando que o modelo previu um valor positivo quando, na verdade, é verdadeiramente negativo).

Na linguagem do nosso modelo, isso significa que:

- O ppv precisa ser alto, porque teremos menos maus Pagadores sendo classificado como bons pagadores, aumentando a precisão das previsões positiva.
- E o recall da variável negativa precisa ser alto, porque, entre todos os valores verdadeiramente de maus Pagadores, preciso que haja poucos maus pagadores sendo classificado como bons pagadores.

Em resumo, queremos que nosso modelo detecte bem os maus pagadores e nos traga poucos falsos positivos, garantindo que os bons pagadores não sejam erroneamente classificados como maus pagadores.

Código para avaliar o modelo com a matriz de confusão e as métricas supracitadas:

```python
# Variável real e predita
# Aqui default_flag é a variável real, pois ela já nos traz a informação se um cliente é inadimplente ou adimplente
# E temos o nosso modelo que é a variável "predicted_default"
y_true = df['default_flag']
y_pred = df3['predicted_default']

# Matriz de confusão para avaliar se meu modelo com a nova variável "predicted_default" é um bom modelo.
conf_matrix = confusion_matrix(y_true, y_pred)

# Relatório de classificação
class_report = classification_report(y_true, y_pred)

print("Matriz de Confusão:")
print(conf_matrix)
print("\nRelatório de Classificação:")
print(class_report)

# Visualizar a matriz de confusão
sns.heatmap(conf_matrix, annot=True, fmt='d', cmap='Blues', xticklabels=['Bom Pagador', 'Mau Pagador'], yticklabels=['Bom Pagador', 'Mau Pagador'])
plt.xlabel('Predicted')
plt.ylabel('Actual')
plt.title('Matriz de Confusão')
plt.show()
```
</details>

<details> <summary> <h2>3. Regressão Logística </h2></summary>

A modelo de regressão logística foi proposto como uma alternativa ao modelo anterior, que não alcançou métricas de desempenho satisfatórias. 

A regressão logística é um modelo de classificação onde as variáveis preditoras são contínuas e a variável resposta é binária, geralmente codificada como 0 ou 1. Este modelo é útil para prever a probabilidade de ocorrência de um evento com base nas variáveis independentes.

O código utilizado para a regressão logística e a avaliação das métricas do modelo encontra-se abaixo:

```python
# Selecionar as variáveis independentes e dependentes
X = df[['age', 'last_month', 'number_dependents', 'total_90_days_overdue', 'debt_ratio', 'total_loan']]
y = df['default_flag']
```

```python
# Convertendo y para inteiro.
y = y.astype(int)

# Certifique-se de que y só contém valores binários (0 e 1)
print(y.unique())
```

```python
# Dividir os dados em conjuntos de treinamento e teste
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.25, random_state=42)
```

Foram utilizadas três técnicas diferentes no modelo de Regressão Logística:

1. Class Weight
    
    O parâmetro `class_weight='balanced'` na função de regressão logística do Python foi utilizado para lidar com o problema das classes desbalanceadas na variável `default_flag`. Este parâmetro ajusta automaticamente o peso das classes inversamente proporcional às suas frequências na amostra de dados de treinamento. Em outras palavras, a função de custo é ajustada para penalizar mais os erros cometidos nas classes minoritárias e menos os erros nas classes majoritárias. Isso ajuda o modelo a considerar a importância relativa das classes minoritárias durante o treinamento, resultando em uma melhor performance na predição dessas classes.
    
2. Oversampling
    
    É uma técnica de reamostragem no qual tem com propósito equilibrar a diferença de classes aumentando o número de instâncias da classe minoritária.
    
3. Undersampling
    
    É uma técnica de reamostragem no qual tem com propósito equilibrar essa diferença diminuindo o número de instâncias da classe majoritária.
    

Para ver cada uma delas, acessar o Google Colab: https://colab.research.google.com/drive/1UTQUppbQ1UxoRsXbzcXoNZf4f_LWFN_I?usp=sharing

A que teve maior quantidade de acertos de *Maus Pagadores*, mas ainda assim com boas métricas foi a primeira. Abaixo está o código:

```python
# Inicializar e treinar o modelo de regressão logística com ajuste de pesos das classes
model = LogisticRegression(class_weight='balanced', max_iter=500)
model.fit(X_train, y_train)

# Fazer previsões
y_pred = model.predict(X_test)

# Avaliar o modelo
accuracy = accuracy_score(y_test, y_pred)
report = classification_report(y_test, y_pred)

print("Acurácia do modelo:", accuracy)
print("Relatório de classificação:\n", report)

# Calcular e exibir a matriz de confusão
conf_matrix = confusion_matrix(y_test, y_pred)

plt.figure(figsize=(8, 6))
sns.heatmap(conf_matrix, annot=True, fmt='d', cmap='Blues', xticklabels=['Previsto: Bom Pagador', 'Previsto: Mau Pagador'], yticklabels=['Real: Bom Pagador', 'Real: Mau Pagador'])
plt.xlabel('Classe Prevista')
plt.ylabel('Classe Verdadeira')
plt.title('Matriz de Confusão')
plt.show()
```

Conforme discutido na Seção 2, neste projeto a prioridade foi diminuir os falsos positivos e ter também boa quantidade de acertos de *Maus Pagadores*. Por isso, o formato de regressão logística com o ***Class Weight*** foi o escolhido, pois ele teve os fatores supracitados.




Para mais detalhes acessar o [Notebook do Google Colab](https://colab.research.google.com/drive/1UTQUppbQ1UxoRsXbzcXoNZf4f_LWFN_I?usp=sharing).

</details>
