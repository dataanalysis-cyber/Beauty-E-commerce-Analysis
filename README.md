# 一、数据集描述

迈尔斯·布里格斯性格类型指标（简称 MBTI）是一种性格类型系统，它将每个人分为 4 个轴上的 16 种不同的性格类型：

内向 (I) – 外向 (E)
直觉 (N) – 感觉 (S)
思考 (T) – 情感 (F)
判断 (J) – 感知 (P

```python

import numpy as np   #linear algebra   
import pandas as pd    # data processing, CSV file I/O (e.g. pd.read_csv)
import seaborn as sns   #for visualization of the data
import matplotlib.pyplot as plt


from sklearn.feature_extraction.text import TfidfVectorizer      #for feature scaling
from sklearn.model_selection import train_test_split   #to split train and test data set
from sklearn.linear_model import LogisticRegression     #algorithm to the model
from sklearn.ensemble import RandomForestClassifier
from sklearn.tree import DecisionTreeClassifier
```
