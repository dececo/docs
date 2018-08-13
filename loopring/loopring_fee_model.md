# 路印手续费模型（Loopring’s Fee Model）介绍

## 参考文档

Daniel Wang 的 [Remarks on Loopring’s Fee Model](https://medium.com/loopring-protocol/remarks-on-looprings-fee-model-7b6713f59fb7)

黑客派 https://hacpai.com/article/1507650776845#toc_h3_12

## 利润（`margin`）的概念

使用白皮书里的例子
> Alice想用15个A买4个B（订单1）；  
Bob想用10个B买30个A（订单2）；

最后成交时
> Bob用4个B买了`13.41640786`个A，比期望的12个多。  
Alice用`13.41640786`个A买到4个B，比期望的15个给的少。

在这个例子中，Alice的`margin`是`13.41640786-12=1.41640786`个A，Bob的`margin`是`15-13.41640786=1.58359214`个A。

注意：  
1、此处仅为简单举例，实际情况会更复杂。  
2、**利润（`margin`）与交易成交总量直接相关**。  

## 分润比例（`margin split`）
下订单时，可以指定LRC手续费（LRC Fee, `LrcFee`，数量固定）的同时，指定分润比例（百分比）。分润比例的含义是把利润的这一部分比例奖励给旷工。比如，收益的50%奖励给旷工抵扣手续费。

## 路印当前的收费模型

旷工如果选择LRC手续费，则无需操作；
如果选择分润比例，则必须把LRC手续费等量的资产返还给交易发起者，因此只有按分润比例方式计算出的手续费大于LRC手续费的**2倍**，旷工才会选择后者。
![路印收费模型](/loopring_fee_model.png)
