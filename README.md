# 双十一美妆销售数据分析
## 一、读取数据
```python
#导入模块
import pandas as pd
import numpy as np
import jieba  #中文分词库
import matplotlib.pyplot as plt
import seaborn as sns
%matplotlib inline 
from matplotlib.pyplot import MultipleLocator
data = pd.read_csv('meizhuang.csv')
```
## 二、数据清洗和处理
### 2.1重复数据处理
```python
data_deal = data.drop_duplicates(inplace=False)
data_deal.reset_index(inplace=True,drop=True)#重置索引
```
### 2.2缺失值处理
```python
mode_01 = data_deal.sale_count.mode()
mode_02 = data_deal.comment_count.mode()
print('mode_01:',mode_01)
print('mode_02:',mode_02)
```
运行结果：  
mode_01: 0    0.0  
Name: sale_count, dtype: float64  
mode_02: 0    0.0  
Name: comment_count, dtype: float64  
仅有sale_count和comment_count这两列有空值，且售卖量和评论量的众数都是0，并且空值一般代表的是一件商品未卖出因此用0填充空值  
```python
#填充空值
data_deal= data_deal.fillna(0)
```
### 2.3对商品标题进行分类
```python
title_cut =[]
for i in data_deal.title:
    j = jieba.lcut(i)
    title_cut.append(j)
data_deal['cut_name']=title_cut
data_deal[['title','cut_name']].head()

# 给商品添加分类
sub_type = []   #子类别
main_type = []  #主类别
basic_config_data = """护肤品	套装	套装							
护肤品	乳液类	乳液	美白乳	润肤乳	凝乳	柔肤液'	亮肤乳	菁华乳	修护乳
护肤品	眼部护理	眼霜	眼部精华	眼膜					
护肤品	面膜类	面膜													
护肤品	清洁类	洗面	洁面	清洁	卸妆	洁颜	洗颜	去角质	磨砂						
护肤品	化妆水	化妆水	爽肤水	柔肤水	补水露	凝露	柔肤液	精粹水	亮肤水	润肤水	保湿水	菁华水	保湿喷雾	舒缓喷雾
护肤品	面霜类	面霜	日霜	晚霜	柔肤霜	滋润霜	保湿霜	凝霜	日间霜	晚间霜	乳霜	修护霜	亮肤霜	底霜	菁华霜
护肤品	精华类	精华液	精华水	精华露	精华素										
护肤品	防晒类	防晒霜	防晒喷雾												
化妆品	口红类	唇釉	口红	唇彩											
化妆品	底妆类	散粉	蜜粉	粉底液	定妆粉 	气垫	粉饼	BB	CC	遮瑕	粉霜	粉底膏	粉底霜		
化妆品	眼部彩妆	眉粉	染眉膏	眼线	眼影	睫毛膏									
化妆品	修容类	鼻影	修容粉	高光	腮红										
其他	其他	其他"""

#解析表格中的分类信息
category_config_map = {}
for config_line in basic_config_data.split('\n'):
    basic_cateogry_list = config_line.strip().strip('\n').strip('\t').split('\t')
    main_category = basic_cateogry_list[0]
    sub_category = basic_cateogry_list[1]
    unit_category_list = basic_cateogry_list[2:-1]
    for unit_category in unit_category_list:
        if unit_category and unit_category.strip().strip('\t'):
            category_config_map[unit_category] = (main_category,sub_category)

#分出主要类别和次要类别
for i in range(len(data_deal)):
    exist = False
    for temp in data_deal.cut_name[i]:
        if temp in category_config_map:
            sub_type.append(category_config_map.get(temp)[1])
            main_type.append(category_config_map.get(temp)[0])
            exist = True
            break
    if not exist:
        sub_type.append('其他')
        main_type.append('其他')
#套装也归入了其它类

#放入表格中生成新的列
data_deal['sub_type'] = sub_type
data_deal['main_type'] = main_type

#添加是否为男士专用为一列
gender = []
for i in range(len(data_deal)):
    if '男' in data_deal.cut_name[i]:
        gender.append('是')
    elif '男士' in data_deal.cut_name[i]:
        gender.append('是')
    elif '男生' in data_deal.cut_name[i]:
        gender.append('是')
    else:
        gender.append('否')

data_deal['是否男士专用'] = gender

# 新增销售额列：销售额=销售量*价格
data_deal['销售额'] = data_deal.sale_count*data_deal.price

#新增购买日期为一列
data_deal['update_time'] = pd.to_datetime(data_deal['update_time'])
data_deal= data_deal.set_index('update_time')
data_deal['day'] = data_deal.index.day

#删除中文分词那一列
del data_deal['cut_name']
```

## 三、数据分析
### 3.1各品牌商品数(条形图)
```python
plt.rcParams['font.sans-serif']=['SimHei']  #指定默认字体  
plt.rcParams['axes.unicode_minus']=False  #解决负号'-'显示为方块的问题


plt.figure(figsize=(8,6))
data_deal['店名'].value_counts().sort_values(ascending=False).plot.bar(width=0.8,alpha = 0.6,color = 'b')
plt.title('各品牌SKU数',fontsize = 18)#品牌SKU数就是各品牌的商品数
plt.ylabel('商品数量',fontsize = 14)
plt.show()
```
![download](https://github.com/user-attachments/assets/12c303a1-bd1b-4bc9-b4e2-7618162af1e6)
由该图可以明显看到‘悦诗风吟’品牌的商品数远远高于其它品牌，其次就是‘佰草集’和‘欧莱雅’

### 3.2各品牌总销售量和总销售额（横向条形图）
```python
#各品牌的总销售量和销售额
cmap = plt.cm.tab20  # 使用 tab20 颜色映射
fig,axes = plt.subplots(1,2,figsize = (14,10))
colors = cmap(np.linspace(0, 1, data_deal['店名'].nunique()))
ax1 = data_deal.groupby('店名').sale_count.sum().sort_values(ascending = True).plot(kind = 'barh',ax = axes[0],color = colors)

ax1.set_title('品牌总销售量',fontsize = 15)
ax1.set_xlabel('总销售量')

ax2 = data_deal.groupby('店名')['销售额'].sum().sort_values().plot(kind = 'barh',ax = axes[1],color = colors)
ax2.set_title('品牌总销售额',fontsize = 15)
ax2.set_xlabel('总销售额')

plt.show()
```
![download](https://github.com/user-attachments/assets/b0494119-3fb3-4338-8633-3486c4f3dae4)







