.. _metadata:

#################
合约的元数据
#################

.. index:: metadata, contract verification

Solidity编译器自动生成JSON文件，即合约的元数据，其中包含了当前合约的相关信息。
它可以用于查询编译器版本，所使用的源代码，|ABI| 和 |natspec| 文档，以便更安全地与合约进行交互并验证其源代码。

编译器会将元数据文件的 Swarm 哈希值附加到每个合约的字节码末尾（详情请参阅下文），
以便你可以以认证的方式获取该文件，而不必求助于中心化的数据提供者。

当然，你必须将元数据文件发布到 Swarm （或其他服务），以便其他人可以访问它。
该文件可以通过使用 ``solc --metadata`` 来生成，并被命名为 ``ContractName_meta.json`` 。
它将包含源代码的在 Swarm 上的引用，因此你必须上传所有源文件和元数据文件。

元数据文件具有以下格式。 下面的例子将以人类可读的方式呈现。
正确格式化的元数据应正确使用引号，将空白减少到最小，并对所有对象的键值进行排序以得到唯一的格式。
代码注释当然也是不允许的，这里仅用于解释目的。

.. code-block:: none

    {
      // 必选：元数据格式的版本
      version: "1",
      // 必选：源代码的编程语言，一般会选择规范的“子版本”
      language: "Solidity",
      // 必选：编译器的细节，内容视语言而定。
      compiler: {
        // 对 Solidity 来说是必须的：编译器的版本
        version: "0.4.6+commit.2dabbdf0.Emscripten.clang",
        // 可选： 生成此输出的编译器二进制文件的哈希值
        keccak256: "0x123..."
      },
      // 必选：编译的源文件／源单位，键值为文件名
      sources:
      {
        "myFile.sol": {
          // 必选：源文件的 keccak256 哈希值
          "keccak256": "0x123...",
          // 必选（除非定义了“content”，详见下文）：
          // 已排序的源文件的URL，URL的协议可以是任意的，但建议使用 Swarm 的URL
          "urls": [ "bzzr://56ab..." ]
          // Optional: 在源文件中定义的 SPDX license 标识
          "license": "MIT"
        },
        "mortal": {
          // 必选：源文件的 keccak256 哈希值
          "keccak256": "0x234...",
          // 必选（除非定义了“urls”）： 源文件的字面内容
          "content": "contract mortal is owned { function kill() { if (msg.sender == owner) selfdestruct(owner); } }"
        }
      },
      // 必选：编译器的设置
      settings:
      {
        // 对 Solidity 来说是必须的： 已排序的重定向列表
        remappings: [ ":g/dir" ],
        // 可选： 优化器的设置（ enabled 默认设为 false ）
        optimizer: {
          enabled: true,
          runs: 500,
          details: {
            // peephole defaults to "true"
            peephole: true,
            // jumpdestRemover defaults to "true"
            jumpdestRemover: true,
            orderLiterals: false,
            deduplicate: false,
            cse: false,
            constantOptimizer: false,
            yul: true,
            // Optional: Only present if "yul" is "true"
            yulDetails: {
              stackAllocation: false,
              optimizerSteps: "dhfoDgvulfnTUtnIf..."
            }
          }
        }
      },
      metadata: {
          // Reflects the setting used in the input json, defaults to false
          useLiteralContent: true,
          // Reflects the setting used in the input json, defaults to "ipfs"
          bytecodeHash: "ipfs"
        }
        // Required for Solidity: File and name of the contract or library this
        // metadata is created for.
        compilationTarget: {
          "myFile.sol": "MyContract"
        },
        // Required for Solidity: Addresses for libraries used
        libraries: {
          "MyLib": "0x123123..."
        }
      },
      // 必选：合约的生成信息
      output:
      {
        // 必选：合约的 ABI 定义
        abi: [ ... ],
        // 必选：合约的 NatSpec 用户文档
        userdoc: [ ... ],
        // 必选：合约的 NatSpec 开发者文档
        devdoc: [ ... ],
      }
    }

.. warning::
    由于生成的合约的字节码包含元数据的哈希值，因此对元数据的任何更改都会导致字节码的更改。
    此外，由于元数据包含所有使用的源代码的哈希值，所以任何源代码中的，
    哪怕是一个空格的变化都将导致不同的元数据，并随后产生不同的字节代码。

.. note::
    需注意，上面的 ABI 没有固定的顺序，随编译器的版本而不同。尽管从 Solidity 0.5.12 开始，数组保持了一定的顺序。

.. _encoding-of-the-metadata-hash-in-the-bytecode:

字节码中元数据哈希的编码
=============================================

由于在将来我们可能会支持其他方式来获取元数据文件，
类似 ``{"bzzr0"：<Swarm hash>}`` 的键值对，将会以 `CBOR <https://tools.ietf.org/html/rfc7049>`_ 编码来存储。
由于这种编码的起始位不容易找到，因此添加两个字节来表述其长度，以大端方式编码。
所以，当前版本的Solidity编译器，将以下内容添加到部署的字节码的末尾::

    0xa1 0x65 'b' 'z' 'z' 'r' '0' 0x58 0x20 <32 bytes swarm hash> 0x00 0x29

因此，为了获取数据，可以检查部署的字节码的末尾以匹配该模式，并使用 Swarm 哈希来获取元数据文件。

自动化接口生成和 |natspec| 的使用方法
====================================================

元数据以下列方式被使用：想要与合约交互的组件（例如，Mist）读取合约的字节码，
从中获取元数据文件的 Swarm 哈希，然后从 Swarm 获取该文件。该文件被解码为上面的 JSON 结构。

然后该组件可以使用ABI自动生成合约的基本用户接口。

此外，Mist可以使用 userdoc 在用户与合约进行交互时向用户显示确认消息。

有关 |natspec| 的其他信息可以在 `这里 <https://github.com/ethereum/wiki/wiki/Ethereum-Natural-Specification-Format>`_ 找到。

源代码验证的使用方法
==================================

为了验证编译，可以通过元数据文件中的链接从 Swarm 中获取源代码。
获取到的源码，会根据元数据中指定的设置，被正确版本的编译器（应该为“官方”编译器之一）所处理。
处理得到的字节码会与创建交易的数据或者 ``CREATE`` 操作码使用的数据进行比较。
这会自动验证元数据，因为它的哈希值是字节码的一部分。
而额外的数据，则是与基于接口进行编码并展示给用户的构造输入数据相符的。

In the repository `sourcify <https://github.com/ethereum/sourcify>`_
(`npm package <https://www.npmjs.com/package/source-verify>`_) you can see
example code that shows how to use this feature.
