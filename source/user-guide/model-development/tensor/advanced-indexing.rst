.. _tensor-advanced-indexing:

===============
Tensor 高级索引
===============

.. seealso::

   在阅读这部分的内容前，你需要 :ref:`tensor-slice` 。

.. admonition:: NumPy 用户请注意！
   :class: warning

   不能将 NumPy 中存在的一些概念和设计直接应用于 MegEngine.

和 NumPy 索引对比
-----------------

.. seealso::

   在 MegEngine 中，想要 :ref:`access-tensor-element` ，可以使用标准的 ``x[obj]`` 语法。
   看上去一切都和 NumPy 很相似，后者的官方文档中也对 :class:`~.numpy.ndarray` 的各种索引方式都
   `进行了解释 <https://numpy.org/doc/stable/reference/arrays.indexing.html>`_ 。
   但 MegEngine 的 Tensor 实现和 NumPy 还是略有不同，如果不清楚某些细节，可能无法对一些现象做出解释。


索引得到的对象不同
~~~~~~~~~~~~~~~~~~

.. panels::
   :container: +full-width
   :card:

   MegEngine 
   ^^^^^^^^^
   >>> x = tensor([[1., 2.], [3., 4.]])
   >>> y = x[0]
   >>> y[1] = 6
   >>> x
   Tensor([[1. 2.]
    [3. 4.]], device=xpux:0) 
   ---
   NumPy
   ^^^^^
   >>> x = array([[1., 2.], [3., 4.]])
   >>> y = x[0]
   >>> y[1] = 6
   >>> x
   array([[1., 6.],
          [3., 4.]]) 

出现这种情况的原因是，在 NumPy 中使用索引时，得到的是原数组的 `视图（View）
<https://numpy.org/doc/stable/reference/generated/numpy.ndarray.view.html>`_ 。
改变视图中的元素，原始数组中的元素也会发生变化 —— 这是很多 NumPy 用户初学时容易困扰的地方。
而在 MegEngine 中没有视图 ``view`` 这一属性，
通过索引或切片得到的元素或子 Tensor 和原 Tensor 占用的是不同的内存区域。

在其它地方的一些设计，二者还是一致的，接下来我们将进行介绍。

.. _slice-will-not-reduce-dim:

切片索引不会降低维度
~~~~~~~~~~~~~~~~~~~~

MegEngine 和 NumPy 在进行切片时，都不会改变对象 :ref:`tensor-ndim` ：

>>> M = tensor([[1, 2, 3], [4, 5, 6], [7, 8, 9]])
>>> M[1:2], M[1:2].ndim
(Tensor([[4 5 6]], dtype=int32, device=xpux:0), 2)
>>> M[1:2][0:1], M[1:2][0:1].ndim 
(Tensor([[4 5 6]], dtype=int32, device=xpux:0), 2)

整个过程中，切片得到的都是一个 ``ndim=2`` 的 Tensor.

* 执行 ``M[1:2]`` 得到的结果是 ``[[4, 5, 6]]`` 而不是 ``[4, 5, 6]``.
* 对 ``[[4, 5, 6]]`` 进行 ``[0:1]`` 切片，得到的还是 ``[[4, 5, 6]]``.

错误的理解思路可能是这样的：

* 执行 ``M[1:2]`` 得到的结果是 ``[4, 5, 6]``. —— 错！切片不会降维！
* 对 ``[4, 5, 6]`` 进行 ``[0:1]`` 切片，得到的是 ``4``. —— 降维了，因此也不对！

.. note::

   * 切片的作用是从整体中取出一部分，因此不会产生降低维度的行为。
   * 如果你希望切片操作后能去掉冗余的维度，可以使用 :func:`~.squeeze` .

都可以使用数组索引
~~~~~~~~~~~~~~~~~~

实际上除了切片索引，我们还可以使用数组（列表）进行索引：

.. panels::
   :container: +full-width
   :card:

   MegEngine 
   ^^^^^^^^^
   >>> x = tensor([1., 2., 3.])
   >>> y = x[[0, 2]]
   >>> y
   Tensor([1. 3.], device=xpux:0) 
   ---
   NumPy
   ^^^^^
   >>> x = array([1., 2., 3.])
   >>> y = x[[0, 2]]
   >>> y
   array([1., 3.]) 

此时列表的长度对应了被索引的元素的个数，在一些情况下这种机制十分有帮助。

此时 NumPy 将不会生成原始数组的视图，与 MegEngine 的逻辑一致。

注意语法细节，一些用户容易将列表索引写成如下形式：

>>> x = tensor([1., 2., 3.])
>>> y = x[0, 1, 2]
IndexError: too many indices for tensor: tensor is 1-dimensional, but 3 were indexed

实际上这是对 Tensor 的 n 个维度分别进行索引的语法。引出了下一小节的解释 ——

.. _multi-dim-indexing:

在多个维度进行索引
------------------

以下面这个由矩阵（2 维数组） :math:`M` 表示的 Tensor 为例：

.. math::

   M = 
   \begin{bmatrix}
	1 & 2 & 3 \\
	4 & 5 & \color{blue}{6} \\
	7 & 8 & 9 \\
   \end{bmatrix}
   \quad
   M_{(1,2)} = 6

虽然我们可以使用 ``M[1][2]`` 这样的语法得到 6 这个值，但效率并不高（参考 :ref:`access-tensor-element` 的解释）。

.. note::

   * 实际上 Tensor 支持在多个维度直接进行索引；
   * 语法使用 ``,`` 进行分隔，而没有必要使用多个方括号 ``[]`` .

>>> M = tensor([[1, 2, 3], [4, 5, 6], [7, 8, 9]])
>>> M[1,2]
Tensor(6, dtype=int32, device=xpux:0)

可以理解成，在第 0 轴索引值为 1，第 1 轴索引值为 2 的位置去直接访问元素。

推广到一般情况，在访问 n 维 Tensor （假定为 :math:`T` ）的特定某个元素时，可以使用如下语法：

.. math::

   T_{[i_1, i_2, \ldots i_n]}

即我们要提供 :math:`i_1, i_2, \ldots ,i_n` 共 n 个索引值，此时不需要层层降维索引，而是直接得到对应元素。

如果提供的索引数组个数不足 n, 则需要了解 :ref:`default-indexing` 。

.. _multi-dim-array-indexing:

结合数组索引使用
~~~~~~~~~~~~~~~~

对于有 n 个维度的 Tensor, 我们可以传入 n 个列表来进行索引：

>>> M = tensor([[1, 2, 3], [4, 5, 6], [7, 8, 9]])
>>> M[[0, 1], [1, 2]]
Tensor([2 6], dtype=int32, device=xpux:0)
>>> M[[0, 1]] 
Tensor([[1 2 3]
 [4 5 6]], dtype=int32, device=xpux:0)

需要注意的是，数组的长度需要一致，数组长度表示需要被索引的元素个数；

如果提供的索引个数不足 n, 则需要了解 :ref:`default-indexing` 。

.. seealso::

   * NumPy 官网文档中对 `整数列表索引（Integer array indexing）
     <https://numpy.org/doc/stable/reference/arrays.indexing.html#integer-array-indexing>`_ 
     进行了更详细的说明；
   * 除了整数列表索引外，MegEngine 还支持和 NumPy 一致的 `布尔列表索引
     <https://numpy.org/doc/stable/reference/arrays.indexing.html#boolean-array-indexing>`_ 。


.. _multi-dim-slicing:

在多个维度进行切片
------------------
.. note::

   在某个维度上进行索引，除了索引特定元素以外，还可以进行切片操作，来获取特定部分元素。

   * 既然我们可以在多个维度进行索引，自然地，我们可以从多个维度进行切片；
   * 问题在于，用户容易忽视 :ref:`slice-will-not-reduce-dim` 这一特点，尤其是和多个 ``[]`` 使用时。

现在需要从下面这个 2 维 Tensor 中切出蓝色部分的元素：

.. math::

   M = 
   \begin{bmatrix}
	1 & 2 & 3 \\
	\color{blue}{4} & \color{blue}{5} & 6 \\
	\color{blue}{7} & \color{blue}{8} & 9 \\
   \end{bmatrix}

一些人会写成 ``M[1:3][0:2]`` , 此时将得到非预期结果：

>>> M[1:3][0:2]
Tensor([[4 5 6]
 [7 8 9]], dtype=int32, device=xpux:0)

这是因为 ``[]`` 操作是顺序进行解释的，它背后的逻辑顺序是：

.. math::

   \begin{bmatrix}
	1 & 2 & 3 \\
	4 & 5 & 6 \\
	7 & 8 & 9 \\
   \end{bmatrix}
   _{\downarrow{\text{1:3}}}
   =
   \begin{bmatrix}
	4 & 5 & 6 \\
	7 & 8 & 9 \\
   \end{bmatrix}
   \quad
   \begin{bmatrix}
	4 & 5 & 6 \\
	7 & 8 & 9 \\
   \end{bmatrix}
   _{\downarrow{\text{0:2}}}
   =
   \begin{bmatrix}
	4 & 5 & 6 \\
	7 & 8 & 9 \\
   \end{bmatrix}

>>> T = M[1:3]
>>> T
Tensor([[4 5 6]
 [7 8 9]], dtype=int32, device=xpux:0)
>>> T[0:2]
Tensor([[4 5 6]
 [7 8 9]], dtype=int32, device=xpux:0)

.. warning::

   由于切片操作并不会降低维度，所以上面的写法等于每次都在 axis=0 进行切片。

.. seealso::

   如果你不清楚 axis 的概念，可以参考 :ref:`tensor-axes` 。

正确的做法是像 :ref:`multi-dim-indexing` 一样，使用 ``,`` 对维度进行区分：

>>> M[1:3,0:2]
Tensor([[4 5]
 [7 8]], dtype=int32, device=xpux:0)

可以理解成在第 0 轴使用 ``1:3`` 切片，在第 1 轴使用 ``0:2`` 切片，求它们的交集：

.. math::

   \begin{bmatrix}
	1 & 2 & 3 \\
	\color{red}{4} & \color{red}{5} & \color{red}{6}\\
	\color{red}{7} & \color{red}{8} & \color{red}{9}\\
   \end{bmatrix}
   _{\downarrow{\text{1:3}}}
   \cap
   \begin{bmatrix}
	\color{green}{1} & \color{green}{2} & 3 \\
	\color{green}{4} & \color{green}{5} & 6 \\
	\color{green}{7} & \color{green}{8} & 9 \\
   \end{bmatrix} 
   _{\xrightarrow{\text{0:2}}}
   =
   \begin{bmatrix}
	1 & 2 & 3 \\
	\color{blue}{4} & \color{blue}{5} & 6 \\
	\color{blue}{7} & \color{blue}{8} & 9 \\
   \end{bmatrix}

推广到一般情况，在访问 n 维 Tensor （假定为 :math:`T` ）的特定部分的元素时，要求使用如下语法：

.. math::

   T_{[s_1, s_2, \ldots s_n]}

即我们要提供 :math:`s_1, s_2, \ldots ,s_n` 共 n 个切片，每个切片针对特定第维度。

如果提供的切片个数不足 n, 则需要了解 :ref:`default-indexing` 。

.. note::

   多维切片时， ``x[obj]`` 内部的 ``obj`` 由给定的不同维度的切片组成。

.. seealso::

   * 对于 ``ndim`` 特别大的 Tensor （假设超过 1000 维）， 有些时候我们只想对某一个轴进行索引，
     或进行特定操作，此时我们可以使用 :py:func:`~.functional.gather` 或 :py:func:`~.functional.scatter`
   * 这两个方法分别对应于 :py:func:`numpy.take_along_axis` 和 :py:func:`numpy.put_along_axis`

.. _default-indexing:

多维索引的缺省情况
------------------

如果索引一个多维 Tensor 时给定的索引数少于实际的维数 ``ndim``, 将得到一个子 Tensor:

>>> M[2]
Tensor([7 8 9], dtype=int32, device=xpux:0)
>>> M[2,:]
Tensor([7 8 9], dtype=int32, device=xpux:0)
>>> M[:,2]
Tensor([3 6 9], dtype=int32, device=xpux:0)

* 此时其它维度的元素将被完整地保留，等同于使用 ``:`` 作为缺省维度的默认索引；
* 根据给定的明确索引数，得到的子 Tensor 维度个数将对应地减少。


