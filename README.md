# Análise exploratória de dados e Churn

Nesta análise, decidi tomar um rumo diferente com as coisas: Andei tendo muita dificuldade para conseguir estudar com as bases de dados disponíveis na internet, me senti muito limitidado e com muita difículdade de localizar bases que remetessem situações reais.

Neste caso, decidi que eu mesmo criaria minha própria base, com uma pesquisa rápida, descobri a biblioteca Faker, do python, que é feita justamente pra isso, não vou me aprofundar muito nos detalhes dessa base de dados que criei, mas o código para gerar uma parecida está nesse repositório, com o nome de "database_generator", farei um post explicando melhor sobre como utilizei no futuro.

Então, gerando essa base, coloquei como objetivo fazer uma breve análise exploratória de dados e calcular o churn ao longo do tempo dessa minha lista de compras gerada. 

Comecei importando os dados: 
churn_data = pd.read_csv(r"https://raw.githubusercontent.com/vine0alien/ChurnAnalysis/refs/heads/main/churn_data.csv")
Quis tratar os dados em formato de data, que geralmente são os mais problemáticos:
churn_data['acquisition_date'] = pd.to_datetime(churn_data['acquisition_date'])
churn_data['last_purchase'] = pd.to_datetime(churn_data['last_purchase'])
churn_data['year'] = churn_data['acquisition_date'].dt.year

Aqui, já usando minhas experiências dos relatórios passados utilizando análise cohort, decidi que já criaria a função de análise cohort para uso posterior: 

def cohort_analysis(data, year, date_column, customerID):
Esta é a função principal que realiza a análise. Ela recebe os seguintes parâmetros:
data: O DataFrame com os dados.
year: O ano específico para a análise.
date_column: O nome da coluna que contém as datas das transações.
customerID: O nome da coluna que contém os identificadores dos clientes.

Filtrei os dados apenas para o nome especificado na função: 
data_filtered = data[data[date_column].dt.year == year]

Transformei todos os dias do mês em dia 1, para facilitar os cálculos:
def getting_months(m):
    return dt.datetime(m.year, m.month, 1)

Adicionei 2 colunas:

data_filtered['invoice_month'] = data_filtered[date_column].apply(getting_months)
data_filtered['cohort_month'] = data_filtered.groupby(customerID)['invoice_month'].transform('min')

invoice_month: O mês da transação.
cohort_month: O mês da primeira transação do cliente (data de entrada na cohort).

Separei os elementos da data em dia, mês e ano: 
def get_elements_date(df, column):
    day = df[column].dt.day
    month = df[column].dt.month
    year = df[column].dt.year
    return day, month, year

Calculei o cohort_index, que representa o número de meses desde a entrada do cliente no cohort:

invoiceday, invoicemonth, invoiceyear = get_elements_date(data_filtered, date_column)
cohortday, cohortmonth, cohortyear = get_elements_date(data_filtered, 'cohort_month')
YearDiferrence = invoiceyear - cohortyear
MonthDiferrence = invoicemonth - cohortmonth
data_filtered['cohort_index'] = YearDiferrence * 12 + MonthDiferrence + 1

Agrupei e pivotei os dados:

cohort_final_date = data_filtered.groupby(['cohort_month', 'cohort_index'])[customerID].apply(pd.Series.nunique).reset_index()
cohort_pivot = cohort_final_date.pivot(index='cohort_month', columns='cohort_index', values=customerID)
cohort_pivot.index = cohort_pivot.index.strftime('%B %Y')

E por fim, plotei os dados em 2 visualizações, uma com valores absolutos, e uma com as porcentagens:

Valores absolutos:
![CohortTotalValues](https://github.com/user-attachments/assets/f333f511-f234-49d6-bcc9-175f932b1645)

Porcentagens: 
![CohortPercentages](https://github.com/user-attachments/assets/8dc257aa-4825-4186-aa4f-d5c9e403b506)


Depois disso, continuei a análise exploratória de maneira muito simples:
Fiz um countplot de vendas de genêros de clientes: 
sns.countplot(data=churn_data, x='gender')
plt.title('Distribuição de vendas')
plt.show()
![Distribuição](https://github.com/user-attachments/assets/9889daa3-5236-46e5-9f3b-3cf1777a11d5)

E dai me surgiu a primeira dúvida, como estão as vendas por ano? Estamos subindo? Descendo? Mantendo a mesma venda?

A partir disso, somei a venda do ano:

anual_sales = churn_data.groupby('year')['Price'].sum().reset_index()
anual_sales = pd.DataFrame(anual_sales)

E plotei um gráfico:
![AnualSales](https://github.com/user-attachments/assets/3ac5a08b-a1c2-444c-b2d1-d42716be1083)

Mas a imagem me respondeu pouco, qual o melhor ano? qual o pior? tudo muito próximo. 2024 foi quantos % melhor que 2023?

Gerei o segundo gráfico, com um código muito mais complexo, mas que ficou extremamente mais interessante de visualizar:
![AnualSales2 0](https://github.com/user-attachments/assets/1eebb149-26c2-4a47-959b-b754a72aa038)

Bom, perguntas respondidas! Mas e o churn?

Primeiro, defini o período: 720, gostaria de entender, individualmente, de 1 até 720 o comportamento do cliente de retorno.

Comecei criando o período:
periods = range(1, 721)
churn_info = []
Junto com uma lista vazia de informações para armazenar os dados do churn, então criei um loop para o cálculo:

for period in periods:
    churn = (churn_rate(client_base, period, 'last_purchase'))
    churn_info.append({'period_days': int(period), 'Churn_rate': float(churn)})

Funcionou muito bem, o código calcula dia a dia a taxa de churn e armazena no meu dicionário vazio.

Então transformei o dicionário em um data frame:
df_churn = pd.DataFrame(churn_info)

Mas bom, as informações ficaram muito grandes, 720 colunas em um gráfico não ficaria legal, como posso melhorar essa visualização?

Filtrei a base de dados em um segundo data frame, contendo apenas informações de 30 em 30 dias, totalizando 24 meses.

filtered_data = df_churn[df_churn['period_days']%30 ==0]

Comecei a montar meu gráfico:
sns.set_theme(style="ticks")
plt.figure(figsize=(16,9))
sns.barplot(data=filtered_data, y='Churn_rate', x='period_days', hue='period_days', legend=False)
plt.title('Churn rate since purchase')
plt.ylabel('Churn rate %')
plt.xlabel('Days since last purchase')

Precisava criar um rótulo de dados para facilitar, com um loop for, isso fica fácil:

for i, value in enumerate(filtered_data['Churn_rate']):
    percent = value *100
    offset = value +.01
    format_value = f"{percent:.0f}%"
    plt.text(i, offset, format_value, fontsize=12, ha='center', va='center')

E pronto: pergunta inicial respondida com o gráfico:
![churnratesincepurchase](https://github.com/user-attachments/assets/d12265c3-4381-4445-98c6-f76ea8790a4e)

Vimos que os clientes demoram para retornar, quanto maior a porcentagem do churn, menor o número de clientes.
Considere como 100% = 100% dos clientes foram "perdidos" no período.
Em 30 dias, apenas 3% dos clientes retornar, em 6 meses, 21%.
No final da análise de 2 anos, vimos que perdemos 42% da base inicial de clientes.

Todos os códigos estão disponíveis neste repositório!

