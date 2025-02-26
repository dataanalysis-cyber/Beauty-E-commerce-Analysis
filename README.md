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

  可以看出‘相宜本草’的总销售量和销售额都是最高的
  
<mark>待分析</mark>  

### 3.3主类别和此类别销售量占比图
```python
#计算主次类别各品牌总销售量
data1 = data_deal.groupby('main_type')['sale_count'].sum()
data2 = data_deal.groupby('sub_type')['sale_count'].sum()

#画饼状图
fig,axes = plt.subplots(1,2,figsize = (12,10))
data1 = data_deal.groupby('main_type')['sale_count'].sum()
ax1 = data1.plot(kind = 'pie',ax = axes[0],autopct = '%.1f%%',  #设置百分比格式，并保留一位小数
pctdistance = 0.6,#百分比标签到圆心的距离
labels = data1.index,
labeldistance = 1.05 ,#标签与圆心的距离
startangle = 60 ,#初始角度
radius = 2 ,#饼图半径
counterclock = False, #设置为顺时针
wedgeprops = {'linewidth':1.2,'edgecolor' : 'k'}, #设置饼图内外边界的属性
textprops = {'fontsize' :10,'color': 'k'}  #设置文本属性
)#设计饼图的细节

ax1.set_title('主类别销售量占比',fontsize = 12,pad =100) 
ax1.yaxis.set_visible(False)#隐藏纵轴标签

data2 = data_deal.groupby('sub_type')['sale_count'].sum()
ax2 = data2.plot(kind = 'pie',ax = axes[1],autopct = '%.1f%%' ,    #设置百分比格式，并保留一位小数
pctdistance = 0.8,#百分比标签到圆心的距离
labels = data2.index,
labeldistance = 1.05 ,#标签与圆心的距离
startangle = 60 ,#初始角度
radius = 2,#饼图半径
counterclock = False, #设置为顺时针
wedgeprops = {'linewidth':1.2,'edgecolor' : 'k'}, #设置饼图内外边界的属性
textprops = {'fontsize' :6,'color': 'k'}  #设置文本属性
)#设计饼图的细节

ax2.set_title('子类别销售量占比',fontsize = 12,pad =100) 
ax2.yaxis.set_visible(False)#隐藏纵轴标签

plt.subplots_adjust(wspace = 1)
plt.show()
```
![download](https://github.com/user-attachments/assets/263a96eb-dda0-43b5-b32f-42cf8c44915d)

### 3.4各品牌各总类的总销售量堆叠图
```python
#准备数据
stores = data_deal['店名'].unique()
main_types = data_deal['main_type'].unique()
sales_data = {store: {t: 0 for t in main_types} for store in stores}
for _, row in data_deal.iterrows():
    sales_data[row['店名']][row['main_type']] += row['sale_count']

# 绘制堆叠柱状图
fig, ax = plt.subplots(figsize=(20, 10))

# 初始化底部位置
bottom = [0] * len(stores)

# 遍历每个类型，绘制堆叠条形
for t in main_types:
    sales = [sales_data[store][t] for store in stores]
    ax.bar(stores, sales, bottom=bottom, label=t)
    bottom = [b + s for b, s in zip(bottom, sales)]

# 添加标题和标签
plt.title('各品牌各总类的总销量堆叠图')
plt.xlabel('店名')
plt.ylabel('销量')
plt.legend(title='商品类型')

plt.show()
```
![download](https://github.com/user-attachments/assets/93d16b34-6304-44ca-84e3-c14676eadbf2)

### 3.5各品牌各总类的总销售额堆叠图
```python
#准备数据
sales_money = {store: {t: 0 for t in main_types} for store in stores}
for _, row in data_deal.iterrows():
    sales_money[row['店名']][row['main_type']] += row['销售额']

# 绘制堆叠柱状图
fig, ax = plt.subplots(figsize=(20, 10))

# 初始化底部位置
bottom = [0] * len(stores)

# 遍历每个类型，绘制堆叠条形
for t in main_types:
    sales = [sales_money[store][t] for store in stores]
    ax.bar(stores, sales, bottom=bottom, label=t)
    bottom = [b + s for b, s in zip(bottom, sales)]

# 添加标题和标签
plt.title('各品牌各总类的总销售额堆叠图')
plt.xlabel('店名')
plt.ylabel('销售额')
plt.legend(title='商品类型')



plt.show()
```
![download](https://github.com/user-attachments/assets/973ab0f1-f598-4e20-84dd-bb07072bb714)

### 3.6各品牌各子类的总销售量堆叠图
```python
# 准备数据
stores = data_deal['店名'].unique()
sub_types = data_deal['sub_type'].unique()
sales_data1 = {store: {t: 0 for t in sub_types} for store in stores}
for _, row in data_deal.iterrows():
    sales_data1[row['店名']][row['sub_type']] += row['sale_count']

# 绘制堆叠柱状图
fig, ax = plt.subplots(figsize=(20, 12))

# 初始化底部位置
bottom = [0] * len(stores)

# 遍历每个类型，绘制堆叠条形
for t in sub_types:
    sales = [sales_data1[store][t] for store in stores]
    ax.bar(stores, sales, bottom=bottom, label=t)
    bottom = [b + s for b, s in zip(bottom, sales)]

# 添加标题和标签
plt.title('各品牌各子类的总销量堆叠图')
plt.xlabel('店名')
plt.ylabel('销售量')
plt.legend(title='商品类型')

# 设置 y 轴范围
max_sales = max([sum(sales_data1[store].values()) for store in stores])  # 计算最大总销量
plt.ylim(0, max_sales * 1.05)  # 设置 y 轴上限为最大总销量的 1.05 倍

# 显示图表
plt.show()
```
![download](https://github.com/user-attachments/assets/83a58e34-286d-41d5-b156-533ce43d9766)

### 3.7各品牌各子类的总销售额堆叠图
```python
# 准备数据
stores = data_deal['店名'].unique()
sub_types = data_deal['sub_type'].unique()
sales_money1 = {store: {t: 0 for t in sub_types} for store in stores}
for _, row in data_deal.iterrows():
    sales_money1[row['店名']][row['sub_type']] += row['销售额']

# 绘制堆叠柱状图
fig, ax = plt.subplots(figsize=(20, 12))

# 初始化底部位置
bottom = [0] * len(stores)

# 遍历每个类型，绘制堆叠条形
for t in sub_types:
    sales = [sales_money1[store][t] for store in stores]
    ax.bar(stores, sales, bottom=bottom, label=t)
    bottom = [b + s for b, s in zip(bottom, sales)]

# 添加标题和标签
plt.title('各品牌各子类的总销售额堆叠图')
plt.xlabel('店名')
plt.ylabel('销售额')
plt.legend(title='商品类型')

# 设置 y 轴范围
max_sales = max([sum(sales_money1[store].values()) for store in stores])  # 计算最大总销量
plt.ylim(0, max_sales * 1.05)  # 设置 y 轴上限为最大总销量的 1.1 倍

# 显示图表
plt.show()
```
![download](https://github.com/user-attachments/assets/45cf5f7a-c4a6-4a46-acef-129b7f89ffc1)

### 3.8各品牌热度图
```python
data_deal.groupby('店名').comment_count.mean().sort_values(ascending = False).plot(kind = 'bar',color = colors,figsize=(18, 6),title='各品牌商品平均评论数',xlabel = '品牌',ylabel='评论数',rot=0)

plt.show()
```
![download](https://github.com/user-attachments/assets/e2b540fd-90e9-4657-8866-9f25836d918b)

### 3.9各品牌销量热度散点图
```python
# 数据准备
x = data_deal.groupby('店名')['sale_count'].mean()
y = data_deal.groupby('店名')['comment_count'].mean()
s = data_deal.groupby('店名')['price'].mean()
txt = data_deal.groupby('店名')['price'].mean().index

# 绘制散点图
plt.figure(figsize=(12, 10))
sns.scatterplot(x=x, y=y, size=s, hue=s, sizes=(100, 1500))

# 添加店名标注
for i in range(len(txt)):
    plt.annotate(txt[i], xy=(x[i], y[i]))

# 添加标题和标签
plt.ylabel('热度')
plt.xlabel('销量')

# 显示图例
plt.legend(loc='upper left')

# 显示图表
plt.show()
```
![download](https://github.com/user-attachments/assets/647709fc-f3e9-415d-b424-146fbacc6d0c)

### 3.10各品牌价格箱线图
```python
#查看价格的箱型图
plt.figure(figsize=(14,6))
sns.boxplot(x='店名',y='price',data=data_deal, palette='Accent')
plt.ylim(0,3000)#如果不限制，就不容易看清箱型，所以把Y轴刻度缩小为0-3000
plt.show()
```
![download](https://github.com/user-attachments/assets/63158bea-55b8-4b52-9c59-493d6eaf6964)

### 3.11各品牌平均价格
```python
#查看各品牌平均价格
avg_price = data_deal.groupby('店名').price.sum()/data_deal.groupby('店名').price.count()
fig = plt.figure(figsize=(12,6))
avg_price.sort_values(ascending=False).plot(kind = 'bar',width = 0.8,alpha = 0.6,color = 'green',label = '各品牌平均价格')
total_mean = data_deal['price'].mean()
plt.axhline(total_mean,0,1,label='全平台平均价格')
plt.ylabel('各品牌平均价格')
plt.title('各品牌产品的平均价格',fontsize = 24)
plt.legend(loc='best')
    
plt.show()
```
![download](https://github.com/user-attachments/assets/093bbf3e-34b6-4422-ba48-a281a8e2af1e)

### 3.12各品牌销售额和销量的散点图
```python
#销售额和销售量的散点图
plt.figure(figsize = (12,10))

x= data_deal.groupby('店名')['sale_count'].mean()
y= data_deal.groupby('店名')['销售额'].mean()
s=avg_price
sns.scatterplot(x=x,y=y,size =s,sizes=(100,1500),marker = 'v',alpha=0.5,color='b',data=data_deal)
for i in range(len(txt)):
    plt.annotate(txt[i],xy=(x[i],y[i]),xytext = (x[i]+0.2, y[i]+0.2))  #在散点后面增加品牌信息的标签
plt.xlabel('销量')
plt.show()
```
![download](https://github.com/user-attachments/assets/e06fc4de-dbde-4fd7-be00-7db5fe1506a5)

### 3.13男士专用产品销量情况
```python
man_data= data_deal[data_deal['是否男士专用']=='是']
categories = ['护肤品', '化妆品']
man_data_distinct = man_data[man_data.main_type.isin(categories)]
plt.figure(figsize=(10, 6))
sns.barplot(x='店名',y='sale_count',hue='main_type',data=man_data_distinct,saturation=0.75,ci=0)
plt.show()
```
![download](https://github.com/user-attachments/assets/c82ba09b-163e-4f42-984e-c3fc834bb1a7)


### 3.14男士专用产品销量排名和男士专用产品销售额排名
```python
f,[ax1,ax2]=plt.subplots(1,2,figsize = (12,6))
man_data.groupby('店名').sale_count.sum().sort_values(ascending=True).plot(kind='barh',width=0.8,ax=ax1,color = colors)
ax1.set_title('男士产品销量排名')
man_data.groupby('店名').销售额.sum().sort_values(ascending=True).plot(kind='barh',width=0.8,ax=ax2,color = colors)
ax2.set_title('男士产品销售额排名')

plt.show()
```
![download](https://github.com/user-attachments/assets/8814f4b9-2b9c-4ac7-93bc-6c2dcdc72891)

### 3.15双十一前后时间销量关系图
```python
plt.figure(figsize = (12,6))
data_deal.groupby('day')['sale_count'].sum().plot()
plt.grid(linestyle="-.", color="gray", axis="x", alpha=0.5)
x_major_locator = MultipleLocator(1)
ax = plt.gca()
ax.xaxis.set_major_locator(x_major_locator)
plt.xlabel('日期(11月)')
plt.ylabel('销量')
ax.set_title('双十一前后时间销量关系图')
plt.show()
```
![download](https://github.com/user-attachments/assets/a63414e8-1531-48e3-b81d-09bd5937383c)


