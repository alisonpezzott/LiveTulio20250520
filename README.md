## Roteiro  

### 1ª parte: Power BI

- Demonstrar o Repo do GitHub;  
- Mostrar as opções do Power BI Desktop on/off; 
-  
- Ingerir os [dados](./dados) com Power BI Desktop;  
  - https://raw.githubusercontent.com/alisonpezzott/preco-versus-ipca/refs/heads/main/dados/item.csv  
  - https://raw.githubusercontent.com/alisonpezzott/preco-versus-ipca/refs/heads/main/dados/compras.csv  

- Ingerir também a query IPCA:  

```
let
    DataInicial = Date.ToText(#date(2021, 1, 1), "dd/MM/yyyy"),
    DataFinal = Date.ToText(Date.From(DateTime.LocalNow()), "dd/MM/yyyy"),

    Url = "https://api.bcb.gov.br",
    RelativePath = "dados/serie/bcdata.sgs.433/dados", 
    
    Query = [
        formato = "json",
        dataInicial = DataInicial,
        dataFinal = DataFinal
    ],

    Request = Web.Contents(
        Url,
        [ 
            RelativePath = RelativePath,
            Query = Query 
        ]
    ),

    Response = Json.Document(Request),
    
    Tabela = Table.FromRows(
        List.Transform(Response, Record.FieldValues), 
        {"Data", "VarMensal"}
    ),
    
    Adequacao = Table.TransformColumns(
        Tabela,
        {
            {"Data", Date.From, type date}, 
            {"VarMensal", each Number.FromText(_,"en-US") * 0.01, type number} 
        }
    )

in
    Adequacao
```

- Trazer a tabela [Calendario](https://github.com/alisonpezzott/calendario/blob/main/TMDLDesktop/TMDLDesktopPortuguese.tmdl);  
- Aplicar o tema;
- Fazer os relacionamentos;
- Criar a tabela de medidas;
- Desenvolver o problema e seguindo a lógica abaixo:  
  - Colocar um visual de tabela no canvas;
  - Incluir segmentadores de data e itens;  
  - Colocar a data e o item na tabela;  

#### Criação das medidas   

```dax
Preco Unitario = AVERAGE('Compras'[PrecoUnit])
```

Encontrar a data anterior a última venda.  

```
Data Anterior = 

VAR __CompraAnterior = 
    OFFSET(
        -1,
        SUMMARIZE(
            ALL('Compras'), 
            'Calendario'[Data],
            'Item'[Item]
        ),
        ORDERBY('Calendario'[Data], ASC),
        PARTITIONBY('Item'[Item])
    )

VAR __Data = 
    MAXX(__CompraAnterior, [Data])

RETURN
    IF(
        ISINSCOPE('Calendario'[Data]) 
            && ISINSCOPE('Item'[Item])
            && NOT ISBLANK([Preco Unitario]),
        __Data
    ) 

```   

Explicar a função OFFSET e PARTITIONBY.

Agora, que a data da última compra foi encontrada vamos trazer o preço da última compra.  

```
Preco Anterior = 

VAR __DataAnterior = [Data Anterior]

RETURN

    CALCULATE(
        [Preco Unitario],
        'Calendario'[Data] = __DataAnterior
    ) 

```


Vamos agora encontrar a variação percentual entre o preçõ atual e o anterior.  

```
Preco Var = 
VAR __PrecoAnterior =  [Preco Anterior]

RETURN
    DIVIDE([Preco Unitario]-__PrecoAnterior, __PrecoAnterior)

```  

Feito isso, aoga vamos encontrar a variação do IPCA acumulado dinâmico.  

```
IPCA Acumulado = 

VAR __Window = 
    WINDOW(
        1, ABS,
        0, REL,
        ALLSELECTED('IPCA'[Data])
    )

RETURN
    CALCULATE(
        PRODUCTX(
            'IPCA',
            1 + 'IPCA'[VarMensal]
        ),
        __Window
    ) - 1

```

Focar na explicação na função Window.  

Como as datas do IPCA são apenas inícios do mês então temos que encontrar as datas iniciais e finais do período.  

```
Data IPCA Ini = 
VAR __DataRef = [Data Anterior]
RETURN
    IF(
        __DataRef,
        EOMONTH(__DataRef, -1) + 1
    ) 
```

```
Data IPCA Fim = 
    IF(
        [Data Anterior],
        EOMONTH(SELECTEDVALUE('Compras'[DataPedido]), -1) + 1
    ) 

```  

Tendo as datas em mãos nós conseguimos calcular a variação do IPCA no período.  

```
IPCA Var = 
    IF(
        [Data Anterior],
        CALCULATE(
            [IPCA Acumulado],
            DATESBETWEEN('Calendario'[Data], [Data IPCA Ini], [Data IPCA Fim]),
            ALL('Calendario'[Data]) 
        )
    ) 
```  

Com isso agora conseguimos comparar a diferença entre as variações do IPCA e do Preço  

```
Preco - IPCA = [Preco Var] - [IPCA Var]  
```  

Vamos então criar um flag que demonstrará a variação ou se é a primeira compra.  

```
Flag Var =
			
    VAR __DifPrecoVar = [Preco - IPCA]

    VAR __DataAnterior = [Data Anterior]

    RETURN

        SWITCH(
            TRUE(),
            NOT ISBLANK(__DataAnterior) && __DifPrecoVar > 0,  "🡥 acima IPCA",
            NOT ISBLANK(__DataAnterior) && __DifPrecoVar <= 0, "🡧 abaixo IPCA",
            NOT ISBLANK([Preco Unitario]) && ISBLANK(__DataAnterior), "Primeira compra"
        )
```  

Agora queremos criar um segmentador de dados para filtrar as linhas de acordo com cada Flag.  Para isso vamos criar uma nova tabela chamada Flag

```Flag =  
    DATATABLE(
        "Flag", STRING, "FlagID", INTEGER,
        {
            {"🡥 acima IPCA", 0},
            {"🡧 abaixo IPCA", 1},
            {"Primeira compra", 2}

        }
    )
```  

Com a tabela criada podemos incluir um segmentador de dados na tela.
E com este filtro vamos utilizar uma media no filtro lateral.  

```
Flag Filtro = 
IF( [Flag Var] IN VALUES('Flag'[Flag]), 1 )
```  

e para finalizar vamos criar duas medidas para formatação condicional da tabela.  


```
Flag Var Fundo =
			
    SWITCH(
        [Flag Var], 
        "🡥 acima IPCA", "#ffb7b7", 
        "🡧 abaixo IPCA", "#d5f2cd",
        "Primeira compra", "#aed2e7"
    )

```  

```
Flag Var Fonte =

    SWITCH(
        [Flag Var], 
        "🡥 acima IPCA", "#802626", 
        "🡧 abaixo IPCA", "#4b7041",
        "Primeira compra", "#1b4862"
    )
```  

Lembrar de organizar em pastas.
Demonstrar o GitHub Copilot para criar descrições de medidas.  

Salvar como arquivo PBIX PrecoVersusIPCA.pbix e publicar.  


### 2ª parte: Demonstrar integração com Git  
Salvar como arquivo PBIP. Abrir a pasta e excluir o arquivo PBIP fora da pasta.  
Fazer o teste de BPA e deploy.  


### 3ª parte: Fabric  

Utilizar os notebooks e lakehouse 
Demonstrar o modelo semântico em Direct Lake
Integrar com git  

