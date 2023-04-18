.. _inline-assembly:

###############
内联汇编
###############

.. index:: ! assembly, ! asm, ! evmasm

译者注：登链社区有一篇译文 `Solidity 中编写内联汇编(assembly)的那些事 <https://learnblockchain.cn/article/675>`_  推荐阅读。


你可以使用一种接近以太坊虚拟机语言的语言将 Solidity 语句与内联汇编交织在一起。这给你更细粒度的控制，在你通过编写库来增强语言时特别有用。
Solidity 中用于内联汇编的语言称为 :ref:`Yul <yul>`，它在其自己的章节中有记录。本节将仅介绍内联汇编代码如何与包围着的 Solidity 代码交互。


.. 提示::
    内联汇编是一种在低级别访问以太坊虚拟机的方法。这绕过了几个重要的安全特性和 Solidity 检查。
    你应该只将它用于需要它的任务，并且需要有足够信心用好它们；


内联汇编块由 ``assembly { ... }`` 标记，其中大括号内的代码是 :ref:`Yul <yul>` 语言的代码。

内联汇编代码可以访问本地 Solidity 变量，如下所述。

不同的内联汇编块不共享命名空间，即不可能调用 Yul 函数或访问在不同的内联汇编块中定义的 Yul 变量。

Example
-------

以下示例提供了库代码来访问另一个合约的代码并将其加载到“bytes”变量中。通过使用 ``<address>.code``，这对于“原生 Solidity”也是可能的。
但这里的要点是，可重用的汇编库可以在不更改编译器的情况下增强 Solidity 语言。

.. code-block:: solidity

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity >=0.4.16 <0.9.0;

    library GetCode {
        function at(address addr) public view returns (bytes memory code) {
            assembly {
                // 获取代码的大小，这需要使用汇编
                let size := extcodesize(addr)
                // 分配输出字节数组——这也可以在没有汇编的情况下完成
                // by using code = new bytes(size)
                code := mload(0x40)
                //新的“memory end”，包括填充
                mstore(0x40, add(code, and(add(add(size, 0x20), 0x1f), not(0x1f))))
                // 在内存中存储长度
                mstore(code, size)
                // 真正获取代码的大小，这需要使用汇编
                extcodecopy(addr, add(code, 0x20), 0, size)
            }
        }
    }

在优化器无法生成高效代码的情况下，内联汇编也很有用，例如:

.. code-block:: solidity

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity >=0.4.16 <0.9.0;


    library VectorSum {
        // 此函数效率较低，因为优化器当前无法删除数组访问中的边界检查.
        function sumSolidity(uint[] memory data) public pure returns (uint sum) {
            for (uint i = 0; i < data.length; ++i)
                sum += data[i];
        }

        // 我们知道我们只在边界内访问数组，所以我们可以避免检查.
        // 0x20 需要添加到数组中，因为第一个槽包含数组长度。
        function sumAsm(uint[] memory data) public pure returns (uint sum) {
            for (uint i = 0; i < data.length; ++i) {
                assembly {
                    sum := add(sum, mload(add(add(data, 0x20), mul(i, 0x20))))
                }
            }
        }

        // 与上面相同，但在内联汇编中完成整个代码。
        function sumPureAsm(uint[] memory data) public pure returns (uint sum) {
            assembly {
                //加载长度（前 32 个字节）
                let len := mload(data)

                // 跳过长度字段.
                //
                // 保留临时变量，以便它可以就地递增。
                //
                // 注意：递增数据将导致此汇编块后的数据变量不可用
                let dataElementLocation := add(data, 0x20)

                // 迭代直到不满足边界.
                for
                    { let end := add(dataElementLocation, mul(len, 0x20)) }
                    lt(dataElementLocation, end)
                    { dataElementLocation := add(dataElementLocation, 0x20) }
                {
                    sum := add(sum, mload(dataElementLocation))
                }
            }
        }
    }

.. index:: selector; of a function

访问外部变量、函数和库
-----------------------------------------------------

你可以使用它们的名称访问 Solidity 变量和其他标识符。

值类型的局部变量可直接在内联汇编中使用,它们都可以被读取和赋值。

引用内存的局部变量计算为变量在内存中的地址，而不是值本身。此类变量也可以赋值，但请注意，
赋值只会更改指针而不是数据，（你有责任遵守 Solidity 的内存管理机制）；
See :ref:`Solidity 中的约定 <conventions-in-solidity>`.

类似地，引用静态大小的 calldata 数组或 calldata 结构的局部变量计算为 calldata 中变量的地址，而不是值本身。
也可以为变量分配一个新的偏移量，但要注意没有执行任何验证以确保变量不会指向超出 ``calldatasize()``.

对于外部函数指针，可以使用访问地址和函数选择器的`x.address`` 和 ``x.selector``.
选择器由四个右对齐字节组成，这两个值都可以分配。例如:

.. code-block:: solidity
    :force:

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity >=0.8.10 <0.9.0;

    contract C {
        // 为返回变量分配一个新的选择器和地址
        function combineToFunctionPointer(address newAddress, uint newSelector) public pure returns (function() external fun) {
            assembly {
                fun.selector := newSelector
                fun.address  := newAddress
            }
        }
    }

对于动态 calldata 数组，您可以使用访问它们的 calldata 偏移量（以字节为单位）和长度（元素数）``x.offset`` 和 ``x.length``.
这两个表达式都可以被赋值, 但由于静态原因, 不会执行任何验证以确保生成的数据区域在 ``calldatasize()``的边界内.

对于本地存储变量或状态变量,单个 Yul 标识符是不够的, 因为它们不一定占用一个完整的存储槽.
因此，它们的“地址”由一个插槽和该插槽内的字节偏移量组成. 为了获取``x``变量指向的槽，使用``x.slot``即可，
为了获取字节偏移量，使用 ``x.offset``.
使用 ``x`` 本身会导致错误.

你还可以分配给本地存储变量指针的 ``.slot`` 部分.
对于这些（结构、数组或映射），``.offset`` 部分始终为零.
但是不能给状态变量的 ``.slot`` 或 ``.offset`` 赋值.

本地 Solidity 变量可用于赋值，例如:

.. code-block:: solidity
    :force:

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity >=0.7.0 <0.9.0;

    contract C {
        uint b;
        function f(uint x) public view returns (uint r) {
            assembly {
                // 我们忽略存储槽偏移量，我们知道在这种特殊情况下它是零。
                r := mul(x, sload(b.slot))
            }
        }
    }

.. 提示::
    如果您访问类型少于 256 位的变量
    (例如 ``uint64``, ``address``, 或 ``bytes16``),
    你是不能对不属于类型编码的位做出任何设想的。特别是，不要假设它们为零.
    为了安全起见，在重要的上下文中使用数据之前，请始终正确清除数据:
    ``uint32 x = f(); assembly { x := and(x, 0xffffffff) /* now use x */ }``
    要清理有符号类型，你可以使用 ``signextend`` 操作码:
    ``assembly { signextend(<num_bytes_of_x_minus_one>, x) }``


从Solidity 0.6.0开始，内联程序集变量的名称不能覆盖内联程序集块范围内的任何可见声明
(包括变量、合约和函数声明 ).

从Solidity 0.7.0开始，在内联程序集块中声明的变量和函数可能不包含 ``.``,
但从内联汇编程序块外访问使用``.``访问Solidity变量是可行的；

避免事项
---------------

内联汇编可能具有相当高级别的外观, 但它实际上是非常低级别的语言。函数调用、循环、ifs 和switch操作通过简单的重写规则进行转换，然后，
汇编器为你做的唯一一件事就是重新安排函数式操作码，计算变量访问的堆栈高度，并在到达块末尾时删除程序集局部变量的堆栈槽。

.. _conventions-in-solidity:

Solidity 中的约定
-----------------------

.. _assembly-typed-variables:

类型变量的值
=========================

与 EVM 汇编相比, Solidity 具有比 256 位更窄的类型,比如``uint24``。
为了提高效率，大多数算术运算都忽略了类型可以短于 256 位的事实。 必要时清理高阶位,
比如, 在将它们写入内存或进行比较前不久.
这意味着如果您从内联汇编中访问这样的变量，您可能必须先手动清除高阶位.

.. _assembly-memory-management:

内存管理
=================

Solidity 通过以下方式管理内存。在内存中的“0x40”位置有一个“空闲内存指针”。
如果要分配内存，请使用从该指针指向的位置开始的内存并更新它。
无法保证之前未使用过内存，因此您不能假定其内容为零字节。没有内置机制来释放分配的内存。
这是一个程序集片段，可用于按照上述过程分配内存：
.. code-block:: yul

    function allocate(length) -> pos {
      pos := mload(0x40)
      mstore(0x40, add(pos, length))
    }


内存的前 64 个字节可以用作短期分配的“暂存空间”。空闲内存指针后的 32 个字节（即从“0x60”开始）意味着永久为零，并用作空动态内存数组的初始值。
这意味着可分配内存从 ``0x80`` 开始，这是空闲内存指针的初始值。

Solidity 中内存数组中的元素总是占用 32 字节的倍数（对于 ``bytes1[]`` 也是如此，但对于 ``bytes`` 和 ``string`` 则不然）。
多维内存数组是指向内存数组的指针。动态数组的长度存储在数组的第一个槽位，后面是数组元素。

.. 提示::
    静态大小的内存数组没有长度字段，但之后可能会添加它以允许静态和动态大小的数组之间更好的转换；所以，不要依赖这个。

内存安全
=============

在不使用内联汇编的情况下，编译器可以依靠内存始终保持定义良好的状态。这与通过 Yul IR <ir-breaking-changes> 的新代码生成管道非常有关系：
此代码生成路径可以将局部变量从堆栈移动到内存以避免堆栈太深错误并执行额外的内存优化，如果它可以依赖于关于内存使用的某些假设。

虽然我们建议始终遵守 Solidity 的内存模型，但内联汇编允许您以不兼容的方式使用内存。
因此，在存在任何包含内存操作或分配给内存中的 Solidity 变量的内联汇编块的情况下，默认情况下，将堆栈变量移动到内存和额外的内存优化是全局禁用的。

但是，你可以专门注释一个汇编块，以表明它实际上遵循 Solidity 的内存模型，如下所示:

.. code-block:: solidity

    assembly ("memory-safe") {
        ...
    }

特别是，内存安全的汇编块只能访问以下内存范围:

- 使用类似上述``allocate``函数的机制自行分配内存。
- 由 Solidity 分配的内存，例如您引用的内存数组范围内的内存。
- 上面提到的内存偏移量0和64之间的暂存空间。
- 位于汇编块开头的空闲内存指针值之后的临时内存，即在空闲内存指针处“分配”的内存，而不更新空闲内存指针。

此外，如果汇编块分配给了内存中的 Solidity 变量，则需要确保对 Solidity 变量的访问仅访问这些内存范围。

因为这主要是关于优化器的，所以仍然需要遵守这些限制，即使汇编块恢复或终止。例如，以下程序集片段不是内存安全的，
因为 ``returndatasize()`` 的值可能超过 64 字节的暂存空间:

.. code-block:: solidity

    assembly {
      returndatacopy(0, 0, returndatasize())
      revert(0, returndatasize())
    }

另一方面，以下代码是内存安全的，因为超出空闲内存指针指向的位置的内存可以安全地用作临时暂存空间:

.. code-block:: solidity

    assembly ("memory-safe") {
      let p := mload(0x40)
      returndatacopy(p, 0, returndatasize())
      revert(p, returndatasize())
    }

请注意，如果没有后续分配，则无需更新空闲内存指针，但只能使用从空闲内存指针给出的当前偏移量开始的内存。

如果内存操作使用长度为零，也可以只使用任何偏移量（不仅是当它落入暂存空间时）：
.. code-block:: solidity

    assembly ("memory-safe") {
      revert(0, 0)
    }

请注意，不仅内联汇编中的内存操作本身可能是内存不安全的，而且对内存中引用类型的 Solidity 变量的赋值也是如此。例如以下不是内存安全的:

.. code-block:: solidity

    bytes memory x;
    assembly {
      x := 0x40
    }
    x[0x20] = 0x42;

既不涉及任何访问内存的操作也不分配给内存中的任何 Solidity 变量的内联汇编自动被认为是内存安全的并且不需要注释。

.. 提示::
    您有责任确保程序集确实满足内存模型。如果您将一个程序集块注释为内存安全的，
    但违反了其中一个内存设定，这将导致不正确和未定义的行为，而这些行为无法通过测试轻易发现。

    如果您正在开发一个旨在兼容多个版本的 Solidity 的库，您可以使用特殊注释将程序集块注释为内存安全的:

.. code-block:: solidity

    /// @solidity memory-safe-assembly
    assembly {
        ...
    }

请注意，我们将在未来的突破性版本中禁止通过评论进行注释；因此，如果您不关心与旧编译器版本的向后兼容性，则继续使用偏好的方言字符串。
