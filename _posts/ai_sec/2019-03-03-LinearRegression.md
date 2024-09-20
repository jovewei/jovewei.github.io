---

layout: post
title:  线性回归 Linear Regression
subtitle: 从假设、损失函数、优化、正则到代码实现，简述线性回归的核心
date:   2019-03-03 22:04:18 +0800
author: "Bryce"
header-img:  'images/gallery/The-disturbing-muses.jpg'
tags:   [AI安全, 机器学习]
mathjax: true
---

> <cite>Title: The disturbing muses  
Creator: Giorgio de Chirico  
Creator Lifespan: 1888/1978  
Date: 1955/1960  
Location: Galleria d'arte Frediano Farsetti  
Physical Dimensions: w70 x h100 cm  
Type: painting  
Medium: oil on canvas  </cite>  

## Hypothesis  

我们先来假设：
$$
h_\omega(x)=\omega x+b
$$
  
- $\omega$与$x$为标量时，则为单变量线性回归  
- $\omega$与$x$为向量时，则为多变量线性回归  

## Cost(loss，采用平方损失函数)

损失函数：
$$
L(f)=L(h_\omega(x))=\sum_{i=1}^{n}{(\omega x_i+b - y_i)^2}
$$

## Optimization

问题即求cost最小时的$\omega$的取值  

采用**Gradient Descent**方法来求解$minimize(loss)$  
> 实际上线性回归的最优化问题可以直接使用最小二乘法获得解析解  

令：
$W = (\omega_1, \omega_2, \omega_3, ..., \omega_n, b)^T $  
$X = (x_1, x_2, x_3, ..., x_n, 1)^T $
$Y = (y_1, y_2, y_3, ..., y_n)^T $

则：
$$
J(\omega) = minimize(loss) = arg min(WX-Y)^2
$$

为求极小值，对其进行求导：  
$$
\frac{\partial J(\omega)}{\partial \omega_j} = 2(h_\omega(x)-y)x_j
$$

梯度方向即为导数方向，沿着梯度下降则为：  
$$
\omega_j = \omega_j - \alpha \frac{\partial J(\omega)}{\partial \omega_j}
$$

更新$\omega$的值，即为训练阶段  

## Regularization

参数过多时容易过拟合，引用正则化对所有参数$\omega$进行惩罚，确保每个$\omega$都取较小值  

L1 (Lasso Regression)：
$$
minimize(loss) = arg min(WX-Y)^2 + \lambda||\omega||
$$

L2 (Ridge Regression)：
$$
minimize(loss) = arg min(WX-Y)^2 + \lambda||\omega||^2
$$

L1 & L2 (Elastic Regression)：
$$
minimize(loss) = arg min(WX-Y)^2 + \frac{\lambda(1-\rho)}{2}||\omega||^2
$$

## 代码实现

```python

# coding=UTF-8
'''
@Author: Bryce Wei
@Date: 2019-08-02 15:45:54
@LastEditors: Bryce Wei
@LastEditTime: 2019-08-02 18:01:23
@Description: Handwriting Linear Regression
'''
import numpy as np 
from sklearn import datasets
import matplotlib.pyplot as plt

class LinearRegression():
    def __init__(self, X_train, Y_train, learning_rate=0.1, epochs=10000, threshold=0.0001):
        self.X_train = X_train
        self.Y_train = Y_train
        self.theta =  np.random.rand(1, self.X_train.shape[1] + 1)
        self.grad = []
        self.learning_rate = learning_rate
        self.epochs = epochs
        self.threshold = threshold
            
    def linear_hypo(self, x):
        '''
        @description: 计算hypothesis
            x = (x1, x2, x3, ..., xn, 1)
        @param {type} 
        @return: 
        '''
        return np.dot(np.c_[np.ones(x.shape[0]), x], self.theta.T)

    def linear_grad(self):
        '''
        @description: 计算梯度
        @param {type} 
        @return: 
        '''
        self.grad = np.dot((self.linear_hypo(self.X_train) - self.Y_train).T, np.c_[np.ones(self.X_train.shape[0]), self.X_train])
        self.grad = self.grad / self.X_train.shape[0]
        # print(self.grad)

    def linear_cost(self, x, y):
        '''
        @description: 计算损失
        @param {type} 
        @return: 
        '''
        self.cost = 0.5 * np.mean((self.linear_hypo(x) - y) ** 2)

    def linear_grad_desc(self):
        '''
        @description: 用梯度下降法来进行优化
        @param {type} 
        @return: 
        '''
        self.linear_cost(self.X_train, self.Y_train)
        cost_change = 1
        i = 1
        while cost_change > self.threshold and i < self.epochs:
            pre_cost = self.cost
            # 沿着梯度的方向更新参数theta的值
            self.linear_grad()
            self.theta = self.theta - self.learning_rate*self.grad
            self.linear_cost(self.X_train, self.Y_train)
            cost_change = abs(pre_cost - self.cost)
            i = i + 1
        

    def fit(self):
        '''
        @description: emm, sklearn里面的fit函数 
        @param {type} 
        @return: 
        '''
        self.linear_grad_desc()
        
    def predict(self, x):
        '''
        @description: 预测
        @param {type} 
        @return: 
        '''
        # print(self.theta)
        y_pre = self.linear_hypo(x)
        return y_pre
    

def plot_line(x, y, y_hat, line_color="blue"):
    '''
    plot outputs
    '''
    plt.scatter(x, y,  color='black')
    plt.plot(x, y_hat, color=line_color, linewidth=3)
    plt.xticks(())
    plt.yticks(())

    plt.show()

    
def main():
    # test
    # 加载sklean中自带的数据集
    dataset = datasets.load_diabetes()
    # 选择维度
    X = dataset.data[:, 2]
    Y = dataset.target
    # 切分数据集为训练集和测试集
    X_train = X[:-50, None]
    Y_train = Y[:-50, None]
    X_test = X[-50:, None]
    Y_test = Y[-50:, None]
    
    lr = LinearRegression(X_train, Y_train)
    lr.fit()
    y_pre = lr.predict(X_test)
    plot_line(X_test, Y_test, y_pre)

main()

```
