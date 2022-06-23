图像去雾的是从被破坏的输入中恢复出干净的图像, 大气散射模型为雾霾效应提供了一个简单的近似:
$$\textit{\textbf{I}}(z) = \textit{\textbf{J}}(z)t(z) + \textit{\textbf{A}}\left( {1 - t(z)} \right)$$

其中, $\textit{\textbf{I}}(z)$为观测到的带雾图像, $\textit{\textbf{A}}$ 为全球大气光系数, $t(z)$为介质透射函数, $\textit{\textbf{J}}(z)$为原始的无雾图像. 此外, $t(z)=e^{-βd(z)}$, 其中$β$和$d(z)$分别为大气散射参数和场景深度. 大气散射模型表明图像去雾是一个不知道$A$和$t(z)$的欠确定问题, 因此上式可以表示为:
$$\textit{\textbf{J}}(z)=(\textit{\textbf{I}}(z)-A)/t(z) +\textit{\textbf{A}}$$ 
因此, 可以通过对$t(z)$和$\textit{\textbf{A}}$的估计, 就能从带雾图像$\textit{\textbf{I}}(z)$中, 恢复原始图像$\textit{\textbf{I}}(z)$.
