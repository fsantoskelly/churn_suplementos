
# Avaliação de retenção e *churn* de um E-commerce de suplementos alimentares

👩‍💻 Autor: Kelly Ferreira

🛠️ Linguagem: Python

🛠️ Ferramentas adicionais: Power BI

📈 Desenvolvido durante a mentoria da instrutora Renata Magner, Product Manager do Itaú

Fonte do Dataset: disponibilizado pela mentora

## Objetivo do Modelo

O objetivo do modelo é avaliar o comportamento dos clientes de um E-commerce de Suplementos alimentos e verificar o risco de *churn*.

## 📚Bibliotecas utilizadas/ aplicabilidade

🔹 Pandas

O `pandas` foi usado para manipulação e análise dos dados, facilitando a leitura do arquivo e o pré-processamento das variáveis. 

🔹 MatplotLib

O `matplotlib.pyplot` foi usado para a apresentação gráfica e visualização das informações.

🔹 DateTime

O `Datetime` foi utilizada para manipulação de datas em forma de números absolutos e realizar cálculos com as datas contidas no dataset.

## 🧮Desenvolvimento das análises

🔹 Cálculo da taxa de retenção:

`df = pd.read_excel("data.xlsx")`

▸ Visualização das colunas/linhas da tabela:

`df.head()`

![image](https://github.com/user-attachments/assets/08f12d60-86c1-4cce-8482-bef4a719462b)

▸ Obter a primeira compra de cada usuário agrupando pela última data":

`first_purchase = df.groupby('user_id')['purchase_date'].min().reset_index()
first_purchase.columns = ['user_id', 'first_purchase_date']`

![image](https://github.com/user-attachments/assets/57fe93d9-6246-47fa-933f-fe8e27618565)

▸ Obter a última compra de cada usuário, agrupando as maiores datas da tabela:

`last_purchase = df.groupby('user_id')['purchase_date'].max().reset_index()
last_purchase.columns = ['user_id', 'last_purchase_date']`

![image](https://github.com/user-attachments/assets/b188366f-90d7-42fc-8743-ba75e2469f2c)

▸ Mesclar com o dataframe original para obter todas as compras com a data da primeira e ultima compra (Simular ao PROCV no Excel)

▸Tabela original x first_purchase

`df = df.merge(first_purchase, on='user_id')`

▸ Tabela original x last_purchase

`df = df.merge(last_purchase, on='user_id')`

![image](https://github.com/user-attachments/assets/ab06180c-681c-46d7-964e-a071c007ddbb)

▸ Identificar clientes novos (primeira compra no mês):

`df['first_purchase_month'] = df['first_purchase_date'].dt.to_period('M')
df['purchase_month'] = df['purchase_date'].dt.to_period('M')
df['is_new_customer'] = df['first_purchase_month'] == df['purchase_month']`

🔹 **Análise da retenção**

Foram considerados como clientes retidos aqueles que realizaram uma nova compra antes do período de 30 dias após a primeira compra.
Para isto, foi criada uma coluna para verificar se houve uma compra dentro de 30 dias após a primeira compra:

`df['cliente_retido_aux'] = (df['purchase_date'] <= df['first_purchase_date'] + pd.Timedelta(days=31)) & (df['purchase_date'] > df['first_purchase_date'])`

▸  Foi criada uma nova tabela (df_retencao) para verificar a retenção apenas uma vez por usuário:
`df_retencao = df.groupby('user_id').agg({
    'cliente_retido_aux': 'max'
}).reset_index()`

![image](https://github.com/user-attachments/assets/d4b237d7-7320-4cb3-8293-109fbdf43ff6)


🔹**Análise de reativação**

Neste ponto foram avaliados os clientes que estavam inativos por um tempo e voltaram a comprar após 30 dias a contar da primeira compra, excluindo os clientes retidos. Esta marcação será importante para marcar este cliente reativado e identificar o comportamento do cliente.

▸ Adicionar uma coluna que verifica se houve uma compra depois de 30 dias após a primeira compra e o cliente não é retido

`df['cliente_reativado_aux'] = (df['purchase_date'] >= df['first_purchase_date'] + pd.Timedelta(days=31)) & (df['cliente_retido_30'] != True)`

![image](https://github.com/user-attachments/assets/c1a6427d-f517-4671-b03f-e7fa9647056f)      

▸ Agrupado por user_id para obter a informação de reativação apenas uma vez por usuário

`df_reativacao= df.groupby('user_id').agg({
    'cliente_reativado_aux': 'max'
}).reset_index()`

▸  O próximo passo foi juntar a classificação de retenção ao dataframe original:

`df = df.merge(df_reativacao[['user_id', 'cliente_reativado_aux']], on='user_id', how='left',  suffixes=('', '_'))# Remover a coluna temporária do dataframe original
df.drop(columns=['cliente_reativado_aux'], inplace=True)`

▸E renomear a coluna para cliente_reativado:
 
`df.rename(columns={'cliente_reativado_aux_': 'cliente_reativado'}, inplace=True)`

![image](https://github.com/user-attachments/assets/fe0fa83b-4476-4bb1-8a9a-de9ed27e8706)


🔹**Análise de Churn**

Foram considerados os clientes que compraram apenas uma vez há mais de 90 dias, ou seja, deixaram de comprar.

▸ Os passos para avaliar a retenção e reativação são os mesmos, porém será aplicada outra regra, pois churn são os clientes não retidos, Nào reativados e que não realizaram uma outra compra até 90 dias após a primeira compra.

▸ Utilizada a data de 01-01-2024 como "Data Atual" e calcular a diferença de dias desde a última data de compra: 

`data_atual="01-01-2024"
data_atual=pd.to_datetime(data_atual)
df['days_since_last_purchase'] = (data_atual - df['last_purchase_date']).dt.days`

▸Adicionada a coluna `cliente_churn` para classificar os clientes que se encaixa nesta condição:

`data_atual="01-01-2024"
data_atual=pd.to_datetime(data_atual)
df['days_since_last_purchase'] = (data_atual - df['last_purchase_date']).dt.days`

▸Os clientes que voltaram a comprar 30 dias após a primeira compra ou ser um cliente reativado, mas que está há mais de 90 dias sem voltar a comprar, foi classificado como cliente em risco de churn:

`df['cliente_risco_churn'] = (df['cliente_reativado'] == True) | (df['cliente_retido_30'] == True) & (df['days_since_last_purchase'] > 90)`

🔹**Cálculo de taxa mensal de retenção (%):**

`retention_data = df[df['cliente_retido_30']].groupby(df['first_purchase_date'].dt.to_period('M')).agg({'user_id': 'nunique'})
retention_rate = (retention_data / total_data).fillna(0) * 100`

![image](https://github.com/user-attachments/assets/0f9f2eaf-c703-4709-932c-8e82fd34e354)

🔹**Cálculo de taxa mensal de churn (%):**

`churn_data = df[df['cliente_churn']].groupby(df['first_purchase_date'].dt.to_period('M')).agg({'user_id': 'nunique'})
churn_rate = (churn_data / total_data).fillna(0) * 100`

![image](https://github.com/user-attachments/assets/592f120c-87c1-44be-a5d7-7c1fff07e82a)

🔹**Cálculo de taxa mensal de risco churn (%):**

Os clientes que voltaram a comprar 30 dias após a primeira compra ou ser um cliente reativado, mas que está há mais de 90 dias sem voltar a comprar, foi classificado como cliente em risco de churn:

`risk_churn_data = df[df['cliente_risco_churn']].groupby(df['first_purchase_date'].dt.to_period('M')).agg({'user_id': 'nunique'})
risk_churn_rate = (risk_churn_data / total_data).fillna(0) * 100`

![image](https://github.com/user-attachments/assets/b4eaecb0-c17b-4bdb-9c24-8f916f8629e5)

🔹**Cálculo de taxa mensal de reativação (%):**

Clientes que já compraram,  mas retornam após algum tempo.

`revival_data = df[df['cliente_reativado']].groupby(df['first_purchase_date'].dt.to_period('M')).agg({'user_id': 'nunique'})
revival_rate = (revival_data / total_data).fillna(0) * 100`

![image](https://github.com/user-attachments/assets/dc3c8c10-57ac-4740-ab23-c7fd7b66582b)

🔹**Exportação dos dados para Excel para análise em Power BI:**

`retention_rate.to_excel("taxa_retencao.xlsx")
churn_rate.to_excel("taxa_churn.xlsx")
risk_churn_rate.to_excel("taxa_risco_churn.xlsx")
revival_rate.to_excel("taxa_reativacao.xlsx")`

<h2>📊 Dashboard Power BI</h2>

<a href="https://app.powerbi.com/view?r=eyJrIjoiYTE2MmQ1ZDEtNDk4YS00MmNiLWIxOGItNmQyM2Y2YTRiZjA2IiwidCI6IjE0Y2JkNWE3LWVjOTQtNDZiYS1iMzE0LWNjMGZjOTcyYTE2MSIsImMiOjh9" target="_blank">
  <img src="https://github.com/user-attachments/assets/c3175304-d3f1-4585-b833-65a54396ae43" alt="Dashboard Power BI" style="max-width:100%;" />
</a>

🔗 **Clique na imagem acima para acessar o dashboard interativo.**

##  📝Considerações Finais

🔹 É possível verificar que a taxa de retenção está em queda desde o mês de Fevereiro/2023;

🔹O número total de novos clientes apresentou uma alta em Junho/23, porém não conseguiu se manter ao longo dos demais meses;

🔹A taxa de clientes reativados apresentou uma alta em Março/23, porém não se manteve ao longo dos demais meses;

🔹A taxa de churn apresentou um pico em Julho/23 e mantêm tendência de alta;

🔹O risco de churn está em tendência de baixa, porém, considerando o aumento do churn e a queda no número total de clientes, é inevitável que este parâmetro esteja reduzido.

💡 Com os dados apresentados, podemos observar um comportamento de baixa tava de recompra/reativação e uma tendência de aumento da taxa de churn ao longo dos meses. São necessários dados complementares para avaliar quais os fatores podem ter contribuído para estes resultados, como campanhas publicitárias e nível de satisfação dos clientes.






