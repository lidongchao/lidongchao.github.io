---
layout:     post
title:      "Beancount使用经验"
subtitle:   "通过Beancount导入支付宝csv账单"
date:       2018-07-20
author:     "Li Dongchao"
header-img: "img/default-post-bg.jpg"
tags:
    - Beancount
---



# 一 背景

## 1.1 介绍

Beancount是一款强大的复式记账工具，使用python作为开发语言，以文本文档作为账本，记录日常生活中的每一笔交易，能够有效地管理资产状况和交易记录，同时生成财务报表用于后续的分析。

以下是关于Beancount的两篇中文博客：

[Beancount —— 命令行复式簿记](https://wzyboy.im/post/1063.html) by wzyboy

[Beancount 起步](http://morefreeze.github.io/2016/10/beancount-thinking.html) by MoreFreeze

以下是Beancount的官方文档和官方网站：

[Beancount Documentation](https://docs.google.com/document/d/1RaondTJCS_IUPBHFNdT8oqFKJjVJDsfsn6JEjBG04eA/edit#)

[Beancount on bitbucket](https://bitbucket.org/blais/beancount/commits/all)



## 1.2 安装

Linux环境下安装beancount只需要一条命令，此外还推荐安装fava，能够提供比原生页面更加丰富的内容。

```bash
pip install beancount
pip install fava
```

## 1.3 常用语法

- 断言操作：在时间点[time]的0:00时刻，账户[account]持有[currency unit]货币的数量为[amount]。

```beancount
[time] balance [account] [amount] [currency-unit]
```

- 账户操作：创立账户、备注账户、注销账户。

```beancount
[time] open [account] [currency-unit]
[time] note [account] [note]
[time] close [account]
```

- 账户填充

```beancount
[time] pad [account] [equity-account]
```


## 1.4 初始化

项目的目录结构：

```
Beancount
|--Csv
    |--2018-01-2.csv (非初始化文件，后续生成，仅用于演示)
|--Data
    |--accounts.beancount
    |--2018.beancount
    |--2018-01-1.beancount (非初始化文件，后续生成，仅用于演示)
    |--2018-01-2.beancount (非初始化文件，后续生成，仅用于演示)
|--Importers
    |--\__init__.py
    |--alipay.py
|--my.config
|--strip_blank.py
|--processing.sh
```

[此处](https://github.com/lidongchao/BeancountSample)是已建好的目录结构




---

# 二 最佳实践

## 2.1 前期工作

每结束一个记账周期（推荐一个自然月或一个季度），以该周期为名在Data目录下创建一个beancount文件，例如在2月1日准备对1月的账单进行统计，可以创建```2018-01-1.beancount```文件，末尾的1代表手动生成的账单。然后在文件的开始部分对本月初的负债（Liabilities）和资产（Assets）账户进行断言操作（balance），便于防止错误扩散。

## 2.2 账单探索

前期工作完成以后，下面开始正式的账单统计工作。

### 2.2.1 下载账单

首先登陆[支付宝官网](https://www.alipay.com)，搜索想要保存和分析的一个时间段的交易记录，下载Excel格式的压缩文件，解压缩得到所需要的csv账单文件。

根据账单文件的时间对其进行重命名，例如```2018-Q1-2.csv```，代表2018年第一季度的交易记录，或者```2018-01-2.csv```，代表2018年1月的交易记录，末尾的2代表支付宝自动生成的账单。

### 2.2.2 查看编码(optional)

在终端通过`file`命令查询文件格式

```bash
$ file 2018-01-2.csv

2018-01-2.csv: ISO-8859 text, with very long lines, with CRLF line terminators
```
结果表明文件采用ISO-8859编码（实际上是GBK编码），且换行符为CRLF(\r\n)

### 2.2.3 打开账单

**该步骤建议在windows下进行。**

csv账单文件如果直接通过Excel打开，所有的中文都将无法正常显示。

- 方法一：在Excel2016中新建一个空白的工作簿，通过“数据”-“自文本”-选择csv账单文件-“文本导入向导”的方式进行导入，导入向导的第二步将逗号指定为分隔符号，点击“完成”，即可完成账单文件的打开。
- 方法二：使用记事本(Notepad)打开csv账单文件，选中“文件(File)”-“另存为(Save As)”，“编码(Encoding)”选择UTF-8，覆盖保存，然后就可以直接通过Excel打开。

注：不建议在linux下使用```iconv```功能，会自动清除掉BOM，导致乱码。

### 2.2.4 查看账单(optional)

**该步骤建议在windows下进行。**

打开csv账单文件之后能够发现，前四行和末七行属于统计信息，第五行为字段名称，通过“交易创建时间”降序排列。beancount文件的精度为天，虽然不同天中交易记录的顺序不会产生影响，但是默认同一天内先记录的交易早于后记录的交易。如果直接开始导入，虽然后续输出会自动对日期进行升序排序，但同一天内的交易仍保持降序，由此产生的交易顺序的不一致将会影响后续对账工作。

最后可以看到，目前通过支付宝下载得到的账单无法查看付款方式、支付明细等信息，但这些信息能够有效地对每一条记录进行分类，因此建议通过手工对账的方式对账单文件进行修改，主要修改内容为“备注”栏。

### 2.3 账单预处理

### 2.3.1 手工对账

**该步骤建议在windows下进行。**

鉴于上述情况，打开账单文件之后，删除前四行和末七行的无效信息，然后在“交易创建时间”上右键-“排序”-“升序”。

接着在手机上打开支付宝，选择“我的”-“账单“，右上角选择“...”-“资金明细”-“花呗额度明细”，开始进行花呗账单的对账工作，同时在对应的“备注”栏中注明“花呗”。其余同理，可参考下表：

|账单|备注|
|:--|--:|
|花呗|花呗|
|余额宝|余额宝|
|余额|支付宝|
|招商银行信用卡|招行信用卡|
|建设银行储蓄卡|建行储蓄卡|

需要注意的地方是：
- 手机账单显示余额宝每月最后一天(如12月31日)发放的收益会记录在csv账单文件的下个月第一天(如1月1日)。
- “收/支”栏包括“收入”、“支出”、空白三种情况，前两种情况只需在“备注”栏注明收入支出所用账户，最后一种情况属于内部转账，例如提现时需要注明“支付宝-建行储蓄卡”、“余额宝-建行储蓄卡”，花呗和信用卡还款时需要注明“余额宝-花呗”、“建行储蓄卡-招行信用卡”等等。
- 对余额进行对账时，相同时间的一对增减金额记录需要忽略。
- 注意明细的收支金额下方是余额信息，可以辅助进行月初的断言操作。
- **无论通过上述哪种方式打开，完成修改之后，必须通过“另存为”，然后覆盖保存为同名CSV(逗号分隔符)(\*.csv)格式文件，同时移动到Beancount/Csv/目录下，用于备份以及稍后的处理。**

### 2.3.2 变更编码

Excel另存后的csv文件会变回ISO-8859编码，为了便于后续操作，需要将文件的编码变更为UTF-8，且换行符变更为LF(\n)

```bash
$ cd Csv/
$ iconv -f gbk -t UTF-8 2018-01-2.csv > 2018-01-2_tmp.csv
$ dos2unix 2018-01-2_tmp.csv
$ file 2018-01-2_tmp.csv
2018-01-2_tmp.csv: UTF-8 Unicode text, with very long lines
```

---

### 2.3.3 格式处理

此外，还需要删除文件中的所有多余空格。

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
Created on Wed Apr 18 14:58:23 2018
@File: strip_blank.py
@author: lidongchao
"""

import sys, csv

contents = []
with open(sys.argv[1]) as csvfile:
    csvreader = csv.reader(csvfile, delimiter=',', quotechar='"')
    for row in csvreader:
        contents.append([x.strip() for x in row])
for content in contents:
    print(','.join(content))
```

```bash
$ python strip_blank.py 2018-01-2_tmp.csv > 2018-01-2.csv
$ rm 2018-01-2_tmp.csv
$ cd ..
```
**注意：当iconv和strip\_blank.py的输出值重定向到目标文件时，">"重定向符号左右两端的源文件和目标文件不能相同。否则会丢失源文件和目标文件的所有内容。**


---


## 2.4 生成beancount文件

提取操作（bean-extract）用于从csv账单文件中提取交易记录，提取规则依赖my.config文件。

my.config文件负责定义如何阅读并提取csv账单文件，内容在[这里](https://github.com/lidongchao/BeancountSample/blob/master/my.config)，可根据个人情况进行自定义。

定义完成以后，通过bean-extract操作自动生成beancount文件。

```bash
$ bean-extract my.config Csv/2018-01-2.csv > Data/2018-01-2.beancount
```

## 2.5 升级版

结合前面2.3.2变更编码、2.3.3格式处理、2.4生成文件的三组操作，整合为[processing.sh](https://github.com/lidongchao/BeancountSample/blob/master/processing.sh)。
因此可以运行如下命令一次性完成前三步的所有操作
```bash
$ sh processing.sh Csv/2018-01-2.csv
```


## 2.6 核对信息

最后需要再次检查的点包括：

- 确认资产账户之间的转账信息无误，例如信用卡还款、余额宝向银行卡提现等等。
- 某些交易的商家收款人为个人，需要修改其花费账户，建议每一笔个人支付均在手机端记录其用途标签，养成良好习惯，方便归类。此外，对于某些不明开销，建议增加备注信息。
- 退款信息，需联系上下文进行修改。
例如下列两条交易记录形成一对交易-退款记录，需要将退款记录的收支账户设置为与交易记录的相同。
```beancount
2018-01-01 * "北京****有限公司" "**票务 动车DXXXX A-B(招行信用卡)"
  Liabilities:CreditCard:CMB  -111.00 CNY
  Expenses:Travelling          111.00 CNY
2018-01-01 * "北京****有限公司" "退款-**票务 动车DXXXX A-B(招行信用卡)"
  Liabilities:CreditCard:CMB   111.00 CNY
  Expenses:Travelling         -111.00 CNY
```
- 人情往来，AA等信息。
AA可以分为两种情况，如果已还清，那么与退款类似，简单记录即可。
```beancount
2018-01-01 * "****饭店" "餐饮优惠券(招行信用卡)"
  Liabilities:CreditCard:CMB  -100.00 CNY
  Expenses:Daily:Food          100.00 CNY
2018-01-02 * "阿猫" "****AA(支付宝)"
  Assets:VirtualCard:Alipay    50.00 CNY
  Expenses:Daily:Food         -50.00 CNY
```
如果没有还清，则稍微复杂一点。等到下一次记账周期再行补上。
```beancount
2018-01-01 * "****饭店" "餐饮优惠券(招行信用卡)"
  Liabilities:CreditCard:CMB  -100.00 CNY
  Expenses:Daily:Food          50.00 CNY
  Assets:Receivables:AMao      50.00 CNY
2018-02-01 * "阿猫" "****AA(支付宝)"
  Assets:VirtualCard:Alipay    50.00 CNY
  Assets:Receivables:AMao     -50.00 CNY
```
- 投资及其所得。
投资所得最好分为两部分，一部分为本金，另一部分为利息，方便后期统计。
```beancount
2018-01-01 * "****有限责任公司" "定期理财赎回-****(支付宝)"
  Assets:VirtualCard:Alipay   30133.03 CNY
  Assets:MoneyFound:XXXXXX   -30000.00 CNY
  Income:MoneyFound          -133.03 CNY
```
- 删掉投资过程中的冻结信息。
某些交易会产生冻结信息，代表该笔交易属于延迟交易，随后交易完成才会产生真正的交易信息，所以需要删掉该信息。


## 2.7 手工录入

- 由于微信账单只能在手机上查看，因此需要手动统计信息。
- 以下未经过支付宝所产生的交易记录，由于总体量偏少（否则需要优化账单结构，简化记账流程），可以手动记录在2018-01-1.beancount文件中。例如
    + 微信账单，包括微信支付和红包，可以通过微信“我”-“钱包”-右上角“支付中心”-“交易记录”查看每个月的记录。红包信息偏多时可以考虑统一记录为一条。此外，“钱包”-“零钱”-“零钱明细”，选择任意一笔交易，还可以查看该笔交易之后的剩余零钱。
    + 信用卡交易记录。
    + 银行卡转账记录，以及使用银行卡对信用卡进行还款，可通过银行账单进行核对。
    + 钱包零钱，在手机上单独记录每一笔交易的用途和时间。
- 手工录入时可以通过可视化界面进行快速筛选遗漏项。

## 2.8 include

最后在Data/2018.beancount文件中输入以下信息，用于将已统计的1月账单纳入本年的账单中。
```beancount
include "accounts.beancount"
include "2018-01-1.beancount"
include "2018-01-2.beancount"
```


## 3 可视化界面

完成前述所有流程以后，开始欣赏自己的劳动成果吧。通过fava启动可视化界面，随后在浏览器输入localhost:5000，进入浏览界面。

```bash
fava Data/2018.beancount
```

Enjoy yourself.


