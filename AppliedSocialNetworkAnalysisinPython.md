#https://www.coursera.org/learn/python-social-network-analysis/
---

_You are currently looking at **version 1.2** of this notebook. To download notebooks and datafiles, as well as get help on Jupyter notebooks in the Coursera platform, visit the [Jupyter Notebook FAQ](https://www.coursera.org/learn/python-social-network-analysis/resources/yPcBs) course resource._

---

# Assignment 4


```python
import networkx as nx
import pandas as pd
import numpy as np
import pickle
```

---

## Part 1 - Random Graph Identification

For the first part of this assignment you will analyze randomly generated graphs and determine which algorithm created them.


```python
P1_Graphs = pickle.load(open('A4_graphs','rb'))
P1_Graphs
```




    [<networkx.classes.graph.Graph at 0x7f252f106828>,
     <networkx.classes.graph.Graph at 0x7f252f106940>,
     <networkx.classes.graph.Graph at 0x7f252f106978>,
     <networkx.classes.graph.Graph at 0x7f252f1069b0>,
     <networkx.classes.graph.Graph at 0x7f252f1069e8>]



<br>
`P1_Graphs` is a list containing 5 networkx graphs. Each of these graphs were generated by one of three possible algorithms:
* Preferential Attachment (`'PA'`)
* Small World with low probability of rewiring (`'SW_L'`)
* Small World with high probability of rewiring (`'SW_H'`)

Anaylze each of the 5 graphs and determine which of the three algorithms generated the graph.

*The `graph_identification` function should return a list of length 5 where each element in the list is either `'PA'`, `'SW_L'`, or `'SW_H'`.*


```python
def graph_identification():
    
    res = []
    for G in P1_Graphs:
        name = str(G)
        if "barabasi" in name:
            res.append("PA")
        if "watts" in name:
            if "4" in name:
                res.append("SW_H")
            else:
                res.append("SW_L")

    
    
    return res
```

---

## Part 2 - Company Emails

For the second part of this assignment you will be workking with a company's email network where each node corresponds to a person at the company, and each edge indicates that at least one email has been sent between two people.

The network also contains the node attributes `Department` and `ManagementSalary`.

`Department` indicates the department in the company which the person belongs to, and `ManagementSalary` indicates whether that person is receiving a management position salary.

### Part 2A - Salary Prediction

Using network `G`, identify the people in the network with missing values for the node attribute `ManagementSalary` and predict whether or not these individuals are receiving a management position salary.

To accomplish this, you will need to create a matrix of node features using networkx, train a sklearn classifier on nodes that have `ManagementSalary` data, and predict a probability of the node receiving a management salary for nodes where `ManagementSalary` is missing.



Your predictions will need to be given as the probability that the corresponding employee is receiving a management position salary.

The evaluation metric for this assignment is the Area Under the ROC Curve (AUC).

Your grade will be based on the AUC score computed for your classifier. A model which with an AUC of 0.88 or higher will receive full points, and with an AUC of 0.82 or higher will pass (get 80% of the full points).

Using your trained classifier, return a series of length 252 with the data being the probability of receiving management salary, and the index being the node id.

    Example:
    
        1       1.0
        2       0.0
        5       0.8
        8       1.0
            ...
        996     0.7
        1000    0.5
        1001    0.0
        Length: 252, dtype: float64


```python
def salary_predictions():
    
    G = nx.read_gpickle('email_prediction.txt')

    dep = []
    sal = []
    for i in (G.nodes(data = True)):
        for sub in i[1].items():
            if "Department" in str(sub):
                dep.append(sub[1])
            if "Management" in str(sub):
                sal.append(sub[1])


    df = pd.DataFrame(list(zip(dep,sal)),columns = ["department", "ManSalary"])


    clustering = pd.Series(nx.clustering(G))
    degree = pd.Series(G.degree())


    df = pd.concat([df, pd.DataFrame(clustering)], axis = 1)
    df = pd.concat([df, pd.DataFrame(degree)], axis = 1)

    df.columns = ["dep", "sal", "clust", "degree"]
    #df["sal"] = pd.to_numeric(pd.Series(df.sal), downcast="integer").values

    test = df[df["sal"].isnull()]
    train = df[~df["sal"].isnull()]

    train.loc[:,"sal"] = train["sal"].astype(int)


    from sklearn import svm
    clf = svm.SVC(probability=True)
    clf.fit(train[['dep', 'clust', "degree"]], train['sal'])

    #sal_pred = clf.predict(test[['dep', 'clust', "degree"]])
    proba = clf.predict_proba(test[['dep', 'clust', "degree"]])

    te = [(x[1]) for x in proba]

    res = pd.Series(te, index=test.index.values)

    
    return res
```

### Part 2B - New Connections Prediction

For the last part of this assignment, you will predict future connections between employees of the network. The future connections information has been loaded into the variable `future_connections`. The index is a tuple indicating a pair of nodes that currently do not have a connection, and the `Future Connection` column indicates if an edge between those two nodes will exist in the future, where a value of 1.0 indicates a future connection.

Using network `G` and `future_connections`, identify the edges in `future_connections` with missing values and predict whether or not these edges will have a future connection.

To accomplish this, you will need to create a matrix of features for the edges found in `future_connections` using networkx, train a sklearn classifier on those edges in `future_connections` that have `Future Connection` data, and predict a probability of the edge being a future connection for those edges in `future_connections` where `Future Connection` is missing.



Your predictions will need to be given as the probability of the corresponding edge being a future connection.

The evaluation metric for this assignment is the Area Under the ROC Curve (AUC).

Your grade will be based on the AUC score computed for your classifier. A model which with an AUC of 0.88 or higher will receive full points, and with an AUC of 0.82 or higher will pass (get 80% of the full points).

Using your trained classifier, return a series of length 122112 with the data being the probability of the edge being a future connection, and the index being the edge as represented by a tuple of nodes.

    Example:
    
        (107, 348)    0.35
        (542, 751)    0.40
        (20, 426)     0.55
        (50, 989)     0.35
                  ...
        (939, 940)    0.15
        (555, 905)    0.35
        (75, 101)     0.65
        Length: 122112, dtype: float64


```python
def new_connections_predictions():
    G = nx.read_gpickle('email_prediction.txt')
    future_connections = pd.read_csv('Future_Connections.csv', index_col=0, converters={0: eval})
    
    # create variables which describe the nodes of the network, how well they are connected and 
    pa_preds = nx.preferential_attachment(G,future_connections.index)
    jc_preds = nx.jaccard_coefficient(G,future_connections.index)


    pa_my_l = [] # preferntial attachment
    pa_my_val = []

    jc_my_l = [] # jacard attachment
    jc_my_val = []

    my_dfs = [pa_preds, jc_preds]
    count = 0
    for my_df in my_dfs:
        if count == 0:
            for u,v,p in my_df:
                pa_my_l.append((u,v))
                pa_my_val.append(p)
        if count > 0:
            for u,v,p in my_df:
                jc_my_l.append((u,v))
                jc_my_val.append(p)
        count += 1

    # write a code to check if the two nodes are in the same or a different department, if so add either a one or a zero

    pa_preds = nx.preferential_attachment(G,future_connections.index)

    same_dep = []
    for u,v,p in pa_preds:
        if list(G.node[u].values())[0] == list(G.node[v].values())[0]:
            same_dep.append(1)
            va1 = list(G.node[u].values())[0]
            va2 = list(G.node[v].values())[0]
            #print(str(va1)+ "and" + str(va2))
        else:
            same_dep.append(0)


    # get the transformed data set 
    df_pa = pd.DataFrame({"pa_score":pa_my_val, "same_depart": same_dep}, index= pa_my_l)
    df_jc = pd.DataFrame({"jc_score":jc_my_val}, index= jc_my_l)            


    df_score = pd.merge(df_jc, df_pa, left_index=True, right_index=True)
    df_fin = pd.merge(future_connections, df_score, left_index=True, right_index=True)

    # machine learning

    # train test set
    test_X = df_fin[df_fin["Future Connection"].isnull()].iloc[:,[1,2,3]]
    train_X = df_fin[df_fin["Future Connection"].notnull()].iloc[:,[0,1,2,3]]
    train_X["Future Connection"] = train_X["Future Connection"].astype(int)


    # going with a logistic regression
    from sklearn.linear_model import LogisticRegression
    reg = LogisticRegression().fit(train_X.iloc[:,[1,2,3]], train_X.iloc[:,0])
    res = reg.predict(test_X.iloc[:,[0,1,2]])


    fin = reg.predict_proba(test_X.iloc[:,[0,1,2]])
    te = [(x[1]) for x in fin]
    res = pd.Series(te, index= test_X.index.values)

    return res
```


```python
new_connections_predictions()
```




    (107, 348)    0.017752
    (542, 751)    0.009316
    (20, 426)     0.506415
    (50, 989)     0.009061
    (942, 986)    0.008795
    (324, 857)    0.009095
    (13, 710)     0.120126
    (19, 271)     0.296486
    (319, 878)    0.008974
    (659, 707)    0.009290
    (49, 843)     0.008897
    (208, 893)    0.009227
    (377, 469)    0.030832
    (405, 999)    0.012015
    (129, 740)    0.014506
    (292, 618)    0.021528
    (239, 689)    0.008957
    (359, 373)    0.018528
    (53, 523)     0.226330
    (276, 984)    0.008910
    (202, 997)    0.008858
    (604, 619)    0.459551
    (270, 911)    0.008961
    (261, 481)    0.099536
    (200, 450)    0.898377
    (213, 634)    0.009397
    (644, 735)    0.161515
    (346, 553)    0.010079
    (521, 738)    0.011560
    (422, 953)    0.020842
                    ...   
    (672, 848)    0.008961
    (28, 127)     0.613717
    (202, 661)    0.010201
    (54, 195)     0.999580
    (295, 864)    0.009135
    (814, 936)    0.009357
    (839, 874)    0.008795
    (139, 843)    0.009113
    (461, 544)    0.012746
    (68, 487)     0.012968
    (622, 932)    0.009196
    (504, 936)    0.024478
    (479, 528)    0.009078
    (186, 670)    0.009034
    (90, 395)     0.160080
    (329, 521)    0.032423
    (127, 218)    0.159073
    (463, 993)    0.008782
    (123, 142)    0.517217
    (764, 885)    0.008961
    (144, 824)    0.008833
    (742, 985)    0.008791
    (506, 684)    0.009316
    (505, 916)    0.008824
    (149, 214)    0.954665
    (165, 923)    0.012350
    (673, 755)    0.008782
    (939, 940)    0.008795
    (555, 905)    0.009470
    (75, 101)     0.022816
    Length: 122112, dtype: float64




```python

```
