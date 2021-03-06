# Step9 - RNN
#  初步认识的RNN

本小节中，我们将学习具有两个时间步的DNN组成的简单RNN网络，用于拟合功能。

##  提出问题

我们先用一个最简单的序列问题来了解一下RNN的基本运作方式。

假设有一个随机信号发射器，每秒产生一个随机信号，随机值为(0,1)之间。信号发出后，碰到一面墙壁反射回来，来回的时间相加正好是1秒，于是接收器就收到了1秒钟之前的信号。对于接收端来说，可以把接收到的数据序列列表如下：

|时刻|t1|t2|t3|t4|t5|t6|...|
|---|---|---|---|---|---|---|---|
|发射随机信号X|0.35|0.46|0.12|0.69|0.24|0.94|...|
|接收回波信号Y|0|0.35|0.46|0.12|0.69|0.24|...|

具体的描述此问题：当接收端接收到两个连续的值，如0.35、0.46时，系统响应为0.35；下一个时间点接收到了0.12，考虑到上一个时间点的0.46，则二者组合成0.46、0.12序列，此时系统响应为0.46；依此类推，即接收到第二个数值时，总要返回相邻的第一个数值。

我们可以把发射信号看作X，把接收信号看作是Y，则此问题变成了给定样本X和标签值Y，训练一个神经网络，令其当接收到两个序列的值时，总返回第一个值。

读者可能会产生疑问：这个问题用一个最简单不过的程序就可以解决，我们为什么还要大动干戈地使用神经网络呢？如：

```Python
def echo(x1,x2):
    return x2
```

因为这是一个最基本的序列问题，我们先用它投石问路，逐步地理解RNN的精髓所在。

如果把发射信号和回波信号绘制成图，如下图所示：

|样本图|局部放大图|
|---|---|
|<img src="./media/9/random_number_echo_data.png" ch="500" />|<img src="./media/9/random_number_echo_data_zoom.png" ch="500" />|

图一：信号及回波样本序列

其中，红色叉子为样本数据点，蓝色圆点为标签数据点，它总是落后于样本数据一个时间步。还可以看到以上数据形成的曲线完全随机，毫无规律。

与前面学习的DNN和CNN的样本数据都不同，此时的样本数据为三维：
- 第一维：样本 x[0,:,:]表示第0个样本
- 第二维：时间 x[:,1,:]表示第1个时间点
- 第三维：特征 x[:,:,2]表示第2个特征

举个例子来说，x[10, 2, 4] 表示第10个样本的第2个时间点的第4个特征数据。

标签数据为两维：
- 第一维：样本
- 第二维：标签值

## 用DNN的知识来解决问题

### 搭建网络

我们回忆一下，在验证万能近似定理时，我们学习了曲线拟合问题，即带有一个隐层和非线性激活函数的前馈神经网络，可以拟合任意曲线。但是在这个问题里，有几点不同：

1. 不是连续值，而是时间序列的离散值
2. 完全随机的离散值，而不是满足一定的规律
3. 测试数据不在样本序列里，完全独立

所以，即使使用DNN技术中曲线拟合技术得到了一个拟合网络，也不能正确地预测不在样本序列里的测试集数据。但是，我们可以把DNN做一个变形，让它能够处理时间序列数据：

<img src="./media/9/random_number_echo_net.png" width="600" />

图二：两个时间步的DNN

图二中含有两个简单的DNN网络，t1和t2，每个节点上都只有一个神经元，其中，各个节点的名称和含义是：

|名称|含义|在t1,t2上的取值|
|---|---|---|
|x|输入层样本|根据样本值|
|U|x到h的权重值|相同|
|h|隐层|不同|不同|
|bh|h节点的偏移值|相同|
|tanh|激活函数|函数相同|
|s|隐层激活状态|不同|
|V|s到z的权重值|相同|
|z|输出层|不同|
|bz|z的偏移值|相同|
|loss|损失函数|函数相同|
|y|标签值|根据标签值|

由于是一个值拟合的网络，所以在输出层不使用分类函数，损失函数使用 MSE 均方差。在这个具体的问题中，t2的标签值y应该和t1的样本值x相同。

请读者注意，在很多RNN的文字资料中，通常把h和s合并在一起。在这里我们把它们分开画，便于后面的反向传播的推导和理解。

还有一个问题是，为什么t1的后半部分是虚线的？因为在这个问题中，我们只对t2的输出感兴趣，检测t2的输出值z和y的差距是多少，而不关心t1的输出是什么，所以不必计算t1的z值和loss值，处于无监督状态。

### 前向计算

t1和t2是两个独立的网络，在t1和t2之间，用一个W连接t1的隐层激活状态值到t2的隐层输入，对t2来说，相当于有两个输入：一个是t2时刻的样本值x，一个是t1时刻的隐层激活值s。所以，它们的前向计算公式为：

对于t1：

$$
h_{t1}=x_{t1} \cdot U + b_h\tag{1}
$$
$$
s_{t1} = tanh(h_{t1}) \tag{2}
$$

对于t2：
$$
h_{t2}=x_{t2} \cdot U + s_{t1} \cdot W +b_h\tag{3}
$$
$$
s_{t2} = tanh(h_{t2}) \tag{4}
$$
$$
z_{t2} = s_{t2} \cdot V + b_z\tag{5}
$$
$$
loss = \frac{1}{2}(z_{t2}-y_{t2})^2 \tag{6}
$$

公式1至公式6中，所有的变量均为标量，这就有利于我们对反向传播的推导，不用考虑矩阵、向量的求导运算。

### 反向传播

我们首先对t2网络进行反向传播推导：

$$
\frac{\partial loss}{\partial z_{t2}}=z_{t2}-y_{t2} \rightarrow dz_{t2} \tag{7}
$$
$$
\begin{aligned}
\frac{\partial loss}{\partial h_{t2}}&=\frac{\partial loss}{\partial z_{t2}}\frac{\partial z_{t2}}{\partial s_{t2}}\frac{\partial s_{t2}}{\partial h_{t2}} \\
&=dz_{t2} \cdot V \cdot tanh'(s_{t2}) \\
&=dz_{t2} \cdot V \cdot (1-s_{t2}^2) \rightarrow dh_{t2} \tag{8}
\end{aligned}
$$

$$
\frac{\partial loss}{\partial b_z}=\frac{\partial loss}{\partial z_{t2}}\frac{\partial z_{t2}}{\partial b_z}=dz_{t2} \rightarrow db_{z_{t2}} \tag{9}
$$

$$
\frac{\partial loss}{\partial b_h}=\frac{\partial loss}{\partial h_{t2}}\frac{\partial h_{t2}}{\partial b_h}=dh_{t2} \rightarrow db_{h_{t2}} \tag{10}
$$

$$
\frac{\partial loss}{\partial V}=\frac{\partial loss}{\partial z_{t2}}\frac{\partial z_{t2}}{\partial V}=dz_{t2} \cdot s \rightarrow dV_{t1}  \tag{11}
$$

$$
\frac{\partial loss}{\partial U}=\frac{\partial loss}{\partial h_{t2}}\frac{\partial h_{t2}}{\partial U}=dh_{t2} \cdot x_{t2} \rightarrow dU_{t1} \tag{12}
$$

$$
\frac{\partial loss}{\partial W}=\frac{\partial loss}{\partial h_{t2}}\frac{\partial h_{t2}}{\partial W}=dh_{t2} \cdot s_{t1} \rightarrow dW_{t2} \tag{13}
$$

下面我们对t1网络进行反向传播推导。由于t1的后半部分输出是没有监督的，所以我们不必考虑后半部分的反向传播问题，只从s节点开始向后计算。

$$
\frac{\partial loss}{\partial h_{t1}}=\frac{\partial loss}{\partial h_{t2}}\frac{\partial h_{t2}}{\partial s_{t1}}\frac{\partial s_{t1}}{\partial h_{t1}}=dh_{t2} \cdot W \cdot (1-s_{t1}^2) \rightarrow dh_{t1} \tag{14}
$$

$$
\frac{\partial loss}{\partial b_h}=\frac{\partial loss}{\partial h_{t1}}\frac{\partial h_{t1}}{\partial b_h}=dh_{t1} \rightarrow db_{h_{t1}} \tag{15}
$$
$$
db_{z_{t1}} = 0 \tag{16}
$$
$$
\frac{\partial loss}{\partial U}=\frac{\partial loss}{\partial h_{t1}}\frac{\partial h_{t1}}{\partial U}=dh_{t1} \cdot x_{t1} \rightarrow dU_{t1} \tag{17}
$$
$$
dV_{t1} = 0 \tag{18}
$$
$$
dW_{t1}=0 \tag{19}
$$

### 代码实现


#### 时序1的网络实现

时序1的类名叫做timestep_1，其前向计算过程遵循公式1、2，其反向传播过程遵循公式14至公式19。

```Python
class timestep_1(object):
    def forward(self,x,U,V,W,bh):
        self.W = W
        self.x = x
        # 公式1
        self.h = np.dot(self.x, U) + bh
        # 公式2
        self.s = Tanh().forward(self.h)
        self.z = 0

    def backward(self, y, dh_t2):
        # 公式14
        self.dh = np.dot(dh_t2, self.W.T) * Tanh().backward(self.s)
        # 公式15
        self.dbh = self.dh       
        # 公式16
        self.dbz = 0
        # 公式17
        self.dU = np.dot(self.x.T, self.dh)
        # 公式18
        self.dV = 0        
        # 公式19
        self.dW = 0
```

#### 时序2的网络实现

时序2的类名叫做timestep_2，其前向计算过程遵循公式3至公式3，其反向传播过程遵循公式7至公式13。

```Python
class timestep_2(object):
    def forward(self,x,U,V,W,bh,bz,s_t1):
        self.V = V
        self.x = x
        # 公式3
        self.h = np.dot(x, U) + np.dot(s_t1, W) + bh
        # 公式4
        self.s = Tanh().forward(self.h)
        # 公式5
        self.z = np.dot(self.s, V) + bz

    def backward(self, y, s_t1):
        # 公式7
        self.dz = self.z - y
        # 公式8
        self.dh = np.dot(self.dz, self.V.T) * Tanh().backward(self.s)
        # 公式9
        self.dbz = self.dz
        # 公式10
        self.dbh = self.dh        
        # 公式11
        self.dV = np.dot(self.s.T, self.dz)
        # 公式12
        self.dU = np.dot(self.x.T, self.dh)
        # 公式13
        self.dW = np.dot(s_t1.T, self.dh)
```

#### 网络训练代码

在初始化函数中，先建立好一些基本的类，如损失函数计算、训练历史记录，再建立好两个时序的类，分别命名为t1和t2。

```Python
class net(object):
    def __init__(self, dr):
        self.dr = dr
        self.loss_fun = LossFunction_1_1(NetType.Fitting)
        self.loss_trace = TrainingHistory_3_0()
        self.t1 = timestep_1()
        self.t2 = timestep_2()
```

在训练函数中，仍然采用DNN/CNN中学习过的双重循环的方法，外循环为epoch，内循环为iteration，每次只用一个样本做训练，分别取出它的时序1和时序2的样本值和标签值，先做前向计算，再做反向传播，然后更新参数。

```Python
    def train(self):
        num_input = 1
        num_hidden = 1
        num_output = 1
        max_epoch = 100
        eta = 0.1
        self.U = np.random.random((num_input,num_hidden))*2-1
        self.W = np.random.random((num_hidden,num_hidden))*2-1
        self.V = np.random.random((num_hidden,num_output))*2-1
        self.bh = np.zeros((1,num_hidden))
        self.bz = np.zeros((1,num_output))
        max_iteration = dr.num_train
        for epoch in range(max_epoch):
            for iteration in range(max_iteration):
                # get data
                batch_x, batch_y = self.dr.GetBatchTrainSamples(1, iteration)
                xt1 = batch_x[:,0,:]
                xt2 = batch_x[:,1,:]
                yt1 = batch_y[:,0]
                yt2 = batch_y[:,1]
                # forward
                self.t1.forward(xt1,self.U,self.V,self.W,self.bh)
                self.t2.forward(xt2,self.U,self.V,self.W,self.bh,self.bz,self.t1.s)
                # backward
                self.t2.backward(yt2, self.t1.h)
                self.t1.backward(yt1, self.t2.dh)
                # update
                self.U = self.U - (self.t1.dU + self.t2.dU)*eta
                self.V = self.V - (self.t1.dV + self.t2.dV)*eta
                self.W = self.W - (self.t1.dW + self.t2.dW)*eta
                self.bh = self.bh - (self.t1.dbh + self.t2.dbh)*eta
                self.bz = self.bz - (self.t1.dbz + self.t2.dbz)*eta
            #end for
            total_iteration = epoch * max_iteration + iteration
            if (epoch % 5 == 0):
                loss_vld,acc_vld = self.check_loss(dr)
                self.loss_trace.Add(epoch, total_iteration, None, None, loss_vld, acc_vld, None)
                print(epoch)
                print(str.format("validation: loss={0:6f}, acc={1:6f}", loss_vld, acc_vld))
        #end for
        self.loss_trace.ShowLossHistory("Loss and Accuracy", XCoordinate.Epoch)
```

###  运行结果

<img src="./media/9/random_number_echo_loss.png" width="800"/>

图三：损失函数值和准确度的历史记录曲线

从图三的训练过程看，网络收敛情况比较理想。由于使用单样本训练，所以train loss和train accuracy计算不准确，所以在图中没有画出。

以下是打印输出的最后几行信息：

```
...
98
loss=0.001396, acc=0.952491
99
loss=0.001392, acc=0.952647
testing...
loss=0.002230, acc=0.952609
```

使用完全不同的测试集数据，得到的准确度为95.26%。最后在测试集上得到的拟合结果：

<img src="./media/9/random_number_echo_result.png" width="500" />

红色x是测试集样本，蓝色圆点是模型的预测值，可以看到波动的趋势全都预测准确，具体的值上面有一些微小的误差。

以下是训练出来的各个参数的值：

```
U=[[-0.54717934]], bh=[[0.26514691]],
V=[[0.50609376]], bz=[[0.53271514]],
W=[[-4.39099762]]
```

可以看到W的值比其他值大出一个数量级，这就意味着在t2上的输出主要来自于t1的样本输入，这也符合我们的预期，即：接收到两个序列的数值时，返回第一个序列的数值。






#  LSTM的反向传播

和RNN一样，LSTM也采用基于时间的反向传播算法（Backpropagation Through Time, BPTT）。LSTM的误差项也是沿两个方向传播：
1. 沿时间的反向传播
2. 向上一层网络的反向传播

上一节中介绍了LSTM前向计算的过程，重新列出如下：
$$
f_t=\sigma(W_f\cdot[h_{t-1}, x_t] + b_f) \tag{公式 1}
$$
$$
i_t=\sigma(W_i\cdot[h_{t-1}, x_t] + b_i) \tag{公式 2}
$$
$$
\tilde{c}_t=\tanh(W_c\cdot[h_{t-1}, x_t] + b_c) \tag{公式 3}
$$
$$
c_t=f_t \circ c_{t-1}+i_t \circ \tilde{c}_t \tag{公式 4}
$$
$$
o_t=\sigma(W_o\cdot[h_{t-1}, x_t] + b_o) \tag{公式 5}
$$
$$
h_t=o_t \circ \tanh(c_t) \tag{公式 6}
$$
$$
\hat{y}_t = \sigma(Vh_t + b_y) \tag{公式 7}
$$

LSTM输出层的损失函数 $L$ 采用交叉熵损失函数：
$$
L_t = -y_t\log\hat{y_t} \\
L = \sum_tL_t = -\sum_t{y_t\log\hat{y_t}}
$$
输出层的激活函数 $\sigma$ 采用 softmax 函数。

接下来，我们复习几个在推导过程中会使用到的函数，以及其导数公式。
$$
\begin{aligned}
    \sigma(z) &= y = \frac{1}{1+e^{-z}} \\
    \sigma^{'}(z) &= y(1-y) \\
    \tanh(z) &= y = \frac{e^z - e^{-z}}{e^z + e^{-z}} \\
    \tanh^{'}(z) &= 1-y^2
\end{aligned}
$$

假设某一线性函数 $z_i$ 经过Softmax函数之后的预测输出为 $\hat{y}_i$，该输出的标签值为 $y_i$，则：
$$
\begin{aligned}
    softmax(z_i) &= \hat{y}_i = \frac{e^{z_i}}{\sum_{j=1}^me^{z_j}} \\
    \frac{\partial{loss}}{\partial{z_i}} &= \hat{y}_i - y_i
\end{aligned}

$$


下面开始推导反向传播的过程。

## 误差项沿时间的反向传播

在RNN中，误差项通过隐藏状态 $h$ 的梯度 $\delta$ 向前一时刻传播。在LSTM中，有一个隐藏状态 $h$ 和一个单元状态 $c$，所以定义两个误差项：
$$
\delta_{h}^{t} = \frac{\partial{L}}{\partial{h^t}}
$$
$$
\delta_{c}^{t} = \frac{\partial{L}}{\partial{c^t}}
$$

**注意**：在反向传播过程中，$\delta_{c}^{t}$ 用于反向传播， $\delta_{h}^{t}$ 用于当前层的计算，并没有参与反向传播。

首先，计算在最终时刻 $\tau$ 时的误差项 $\delta_{c}^{\tau}$ 和 $\delta_{h}^{\tau}$。

令 $O^\tau = Vh_t + b_y$，则：
$$
\delta_h^{\tau} = \frac{\partial{L}}{\partial{y^\tau}} \frac{\partial{y^\tau}}{\partial{O^\tau}} \frac{\partial{O^\tau}}{\partial{h^t}} = V^\tau(\hat{y}^\tau - y^\tau) \tag 1
$$
$$
\delta_c^{\tau} = \frac{\partial{L}}{\partial{h^\tau}} \frac{\partial{h^\tau}}{\partial{c^\tau}} = \delta_h^{\tau} \circ o^\tau \circ (1 - \tanh^2(c^\tau))  \tag 2
$$

下面我们来计算任意时刻 $t$ $(t < \tau)$ 的误差项。
$\delta_h^t$ 由当前层的输出梯度决定：
$$
\delta_h^t = \frac{\partial{L}}{\partial{h^t}} = V^t(\hat{y}^t - y^t) \tag 3
$$

$\delta_c^t$ 由后一时刻 $c$ 的梯度误差 $\delta_{c}^{t+1}$ 和本层的输出梯度 $\delta_h^t$ 组成：

$$
\begin{aligned}
\delta_c^t &= \frac{\partial{L}}{\partial{c^{t+1}}} \frac{\partial{c^{t+1}}}{\partial{c^t}} + \frac{\partial{L}}{\partial{h^t}} \frac{\partial{h^t}}{\partial{c^t}} \\

&= \delta_c^{t+1} \circ f^{t+1} + \delta_h^t \circ o^t \circ (1 - \tanh^2(c^t)) \tag 4
\end{aligned}
$$

最后，来更新 LSTM 的权重。LSTM 权重的梯度是每一时刻梯度的和。所以需先求得 $t$ 时刻的梯度，再求得最终梯度。

令：
$$
O_f^t=W_f\cdot[h_{t-1}, x_t] + b_f \\

O_i^t=W_i\cdot[h_{t-1}, x_t] + b_i \\

O_{\tilde{c}}^t=W_c\cdot[h_{t-1}, x_t] + b_c \\

O_o^t=W_o\cdot[h_{t-1}, x_t] + b_o
$$

$W_{fh}$ 的梯度如下：

$$
\begin{aligned}
\frac{\partial{L}}{\partial{W_{fh}}} &= \sum_{t=1}^\tau \frac{\partial{L}}{\partial{c^t}} \frac{\partial{c^t}}{\partial{f^t}} \frac{\partial{f^t}}{\partial{O_f^t}} \frac{\partial{O_f^t}}{\partial{W_{fh}}} \\

&= \sum_{t=1}^\tau \delta_c^t \circ c^{t-1} \circ f^t \circ (1- f^t)(h^{t-1})^T \tag 5
\end{aligned}
$$

同理可得：
$$
\begin{aligned}
\frac{\partial{L}}{\partial{W_{ih}}} &= \sum_{t=1}^\tau \frac{\partial{L}}{\partial{c^t}} \frac{\partial{c^t}}{\partial{i^t}} \frac{\partial{i^t}}{\partial{O_i^t}} \frac{\partial{O_i^t}}{\partial{W_{ih}}} \\

&= \sum_{t=1}^\tau \delta_c^t \circ \tilde{c}^t \circ i^t \circ (1- i^t)(h^{t-1})^T \tag 6
\end{aligned}
$$

$$
\begin{aligned}
\frac{\partial{L}}{\partial{W_{ch}}} &= \sum_{t=1}^\tau \frac{\partial{L}}{\partial{c^t}} \frac{\partial{c^t}}{\partial{\tilde{c}^t}} \frac{\partial{\tilde{c}^t}}{\partial{O_{\tilde{c}}^t}} \frac{\partial{O_{\tilde{c}}^t}}{\partial{W_{ch}}} \\

&= \sum_{t=1}^\tau \delta_c^t \circ i^t \circ (1- (\tilde{c}^t)^2)(h^{t-1})^T \tag 7
\end{aligned}
$$

$$
\begin{aligned}
\frac{\partial{L}}{\partial{W_{oh}}} &= \sum_{t=1}^\tau \frac{\partial{L}}{\partial{h^t}} \frac{\partial{h^t}}{\partial{o^t}} \frac{\partial{o^t}}{\partial{O_o^t}} \frac{\partial{O_o^t}}{\partial{W_{oh}}} \\

&= \sum_{t=1}^\tau \delta_h^t \circ \tanh{c}^t \circ o^t \circ (1- o^t)(h^{t-1})^T \tag 8
\end{aligned}
$$

同理可得：

$$
\begin{aligned}
\frac{\partial{L}}{\partial{b_f}} &= \sum_{t=1}^\tau \frac{\partial{L}}{\partial{h^t}} \frac{\partial{h^t}}{\partial{o^t}} \frac{\partial{o^t}}{\partial{O_o^t}} \frac{\partial{O_o^t}}{\partial{b_f}} \\

&= \sum_{t=1}^\tau \delta_c^t \circ c^{t-1} \circ f^t \circ (1- f^t) \tag 9
\end{aligned}
$$

$$
\begin{aligned}
\frac{\partial{L}}{\partial{b_i}} &= \sum_{t=1}^\tau \frac{\partial{L}}{\partial{c^t}} \frac{\partial{c^t}}{\partial{i^t}} \frac{\partial{i^t}}{\partial{O_i^t}} \frac{\partial{O_i^t}}{\partial{b_i}} \\

&= \sum_{t=1}^\tau \delta_c^t \circ \tilde{c}^t \circ i^t \circ (1- i^t) \tag {10}
\end{aligned}
$$

$$
\begin{aligned}
\frac{\partial{L}}{\partial{b_c}} &= \sum_{t=1}^\tau \frac{\partial{L}}{\partial{c^t}} \frac{\partial{c^t}}{\partial{\tilde{c}^t}} \frac{\partial{\tilde{c}^t}}{\partial{O_{\tilde{c}}^t}} \frac{\partial{O_{\tilde{c}}^t}}{\partial{b_c}} \\

&= \sum_{t=1}^\tau \delta_c^t \circ i^t \circ (1- (\tilde{c}^t)^2) \tag {11}
\end{aligned}
$$

$$
\begin{aligned}
\frac{\partial{L}}{\partial{b_o}} &= \sum_{t=1}^\tau \frac{\partial{L}}{\partial{h^t}} \frac{\partial{h^t}}{\partial{o^t}} \frac{\partial{o^t}}{\partial{O_o^t}} \frac{\partial{O_o^t}}{\partial{b_o}} \\

&= \sum_{t=1}^\tau \delta_h^t \circ \tanh{c}^t \circ o^t \circ (1- o^t) \tag {12}
\end{aligned}
$$


##  误差项向上一层的反向传播

假设 $t$ 时刻当前层为 $l$ , 则定义第 $l-1$ 层的误差项为：
$$
\delta_t^{l-1} \xlongequal{def} \frac{\partial{L}}{\partial{O_t^{l-1}}}
$$
$$
x_t^l = \sigma(O_t^{l-1})
$$
其中， $\sigma$ 为 $l-1$ 层输出项的激活函数。

则：

$$
\begin{aligned}
    \frac{\partial{L}}{\partial{O_t^{l-1}}} &= \frac{\partial{L}}{\partial{f_t^l}} \frac{\partial{f_t^l}}{\partial{O_{ft}^l}} \frac{\partial{O_{ft}^l}}{\partial{x_t^l}}  \frac{\partial{x_t^l}}{\partial{O_t^{l-1}}}

    + \frac{\partial{L}}{\partial{i_t^l}} \frac{\partial{i_t^l}}{\partial{O_{it}^l}} \frac{\partial{O_{it}^l}}{\partial{x_t^l}}  \frac{\partial{x_t^l}}{\partial{O_t^{l-1}}}

    + \frac{\partial{L}}{\partial{{\tilde{c}}_t^l}} \frac{\partial{{\tilde{c}}_t^l}}{\partial{O_{{\tilde{c}}t}^l}} \frac{\partial{O_{{\tilde{c}}t}^l}}{\partial{x_t^l}}  \frac{\partial{x_t^l}}{\partial{O_t^{l-1}}}

    + \frac{\partial{L}}{\partial{o_t^l}} \frac{\partial{o_t^l}}{\partial{O_{ot}^l}} \frac{\partial{O_{ot}^l}}{\partial{x_t^l}}  \frac{\partial{x_t^l}}{\partial{O_t^{l-1}}}
    \\
    &= (\delta_c^t \circ c^{t-1} \circ f^t \circ (1- f^t)) W_{fx} \circ \sigma^{'}(O_t^{l-1})
    + (\delta_c^t \circ \tilde{c}^t \circ i^t \circ (1- i^t)) W_{ix} \circ \sigma^{'}(O_t^{l-1}) \\
    &+ (\delta_c^t \circ i^t \circ (1- (\tilde{c}^t)^2)) W_{cx} \circ \sigma^{'}(O_t^{l-1})
    + (\delta_h^t \circ \tanh{c}^t \circ o^t \circ (1- o^t)) W_{ox} \circ \sigma^{'}(O_t^{l-1})
\end{aligned}
$$

