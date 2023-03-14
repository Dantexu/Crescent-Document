# 技术架构

Crescent是一款开源钱包，本章为钱包基础架构的解析，供研究和学习使用。 注，SDK接入者无需阅读本章也可接入Crescent。

### **Bundler**&#x20;

调用EntryPoint实现打包用户交易，链下验证UserOperation(后续简称UO)，剔除不符合要求和有问题的UO。打包合规的UO，提交上链。&#x20;

主要包含以下功能：&#x20;

* eth\_sendUserOperation：发送交易。
* eth\_estimateUserOperationGas：gas评估。
* eth\_getUserOperationReceipt：获取交易历史。

```solidity
async handleMethod(method, params) {
        let result;
        switch (method) {
            case 'eth_chainId':
                // eslint-disable-next-line no-case-declarations
                const { chainId } = await this.provider.getNetwork();
                result = chainId;
                break;
            case 'eth_supportedEntryPoints':
                result = await this.methodHandler.getSupportedEntryPoints();
                break;
            case 'eth_sendUserOperation':
                result = await this.methodHandler.sendUserOperation(params[0], params[1]);
                break;
            case 'eth_estimateUserOperationGas':
                result = await this.methodHandler.estimateUserOperationGas(params[0], params[1]);
                break;
            case 'eth_getUserOperationReceipt':
                result = await this.methodHandler.getUserOperationReceipt(params[0]);
                break;
            case 'eth_getUserOperationByHash':
                result = await this.methodHandler.getUserOperationByHash(params[0]);
                break;
            case 'web3_clientVersion':
                result = this.methodHandler.clientVersion();
                break;
            default:
                throw new utils_2.RpcError(`Method ${method} is not supported`, -32601);
        }
        return result;
    }
```

### EntryPoint合约

EntryPoint是所有功能的核心入口，每个项目自行部署自己的EntryPoint。Bundler，Wallet和Paymaster都需要围绕EntryPoint工作。

主要包含以下功能：

* simulateValidation：模拟用户交易，链下验证UO。

```solidity
    function simulateValidation(UserOperation calldata userOp) external returns (uint256 preOpGas, uint256 prefund) {
        uint256 preGas = gasleft();


        bytes32 requestId = getRequestId(userOp);
        (prefund,,) = _validatePrepayment(0, userOp, requestId);
        preOpGas = preGas - gasleft() + userOp.preVerificationGas;


        require(msg.sender == address(0), "must be called off-chain with from=zero-addr");
    }
```

其中simulateValidation中如果发现用户钱包地址尚未创建，将依据[EIP-2470](https://eips.ethereum.org/EIPS/eip-2470)为用户创建钱包地址。

* handleOPs：打包合规的UO，提交上链。

```solidity
    function handleOps(UserOperation[] calldata ops, address payable beneficiary) public {


        uint256 opslen = ops.length;
        UserOpInfo[] memory opInfos = new UserOpInfo[](opslen);
}
```

### Payumaster合约

Paymaster具有以下功能和特性：

* 向EntryPoint支付gas费；
* 只响应来自EntryPoint的消息；
* 向EntryPoint确认自己为某UO服务的意愿；
* 在EntryPoint内质押才能成为paymaster；

注：只有通过validatePaymasterUserOp的验证后才会为用户代为支付gas。

```solidity
function validatePaymasterUserOp(UserOperation calldata userOp, bytes32 requestId, uint256 maxCost) external view returns (bytes memory context);
```

### PaymasterProxy合约

是Paymaster的主合约负责Paymaster合约的升级。

```solidity
    function upgradeDelegate(address newImplementation) public {
        require(msg.sender == _getAdmin());
        _upgradeTo(newImplementation);
    }
```

### Wallet合约

用户最终使用的钱包，合约地址既用户钱包地址。具有以下特性和功能：

* 向EntryPoint支付gas费；
* 只响应来自EntryPoint的消息；
* 执行来自EntryPoint的具体交易内容；
* addOwner：DKIM验证通过后添加用户设备。

```solidity
function addOwner(
        address owner,
        VerifierInfo calldata info
    ) external onlyEntryPoint {
        bytes memory modulus = dkimManager.dkim(info.ds);
        require(modulus.length != 0, "Not find modulus!");
        require(IVerifier(dkimVerifier).verifier(owner, modulus, info), "Verification failed!");
        require(allOwner.length < type(uint16).max, "Too many owners");
        uint16 index = uint16(allOwner.length + 1);
        allOwner.push(owner);
        owners[owner] = index;
    }
```

* \_validateSignature：验证签名是否一致，一致方可进行后续操作。

```solidity
 function _validateSignature(UserOperation calldata userOp, bytes32 requestId) internal view override {
        //0x350bddaa addOwner
        bool isAddOwner = bytes4(userOp.callData) == 0x350bddaa;
        if (userOp.initCode.length != 0 && !isAddOwner) {
            revert("wallet: not allow");
        }


        if (!isAddOwner) {
            bytes32 hash = requestId.toEthSignedMessageHash();
            address signatureAddress = hash.recover(userOp.signature);
            require(owners[signatureAddress] > 0, "wallet: wrong signature");
        }
    }
```

* 正常的接收发送功能。

### **WalletProxy**合约

是Wallet的主合约负责Wallet合约的升级，支持手动和自动升级。自动升级时需要WalletController合约配合。

注：默认为手动升级。

```solidity
    function upgradeDelegate(address newDelegateAddress) public {
        require(msg.sender == _getAdmin());
        _upgradeTo(newDelegateAddress);
    }


    function setAutoUpdateImplementation(bool value) public {
        require(msg.sender == _getAdmin());
        StorageSlot.getBooleanSlot(_AUTO_UPDATE_SLOT).value = value;
    }
```

### **WalletController合约**

配合WalletProxy升级钱包。

```solidity
    function setImplementation(address _implementation) public onlyOwner {
        implementation = _implementation;
    }
```

### DKIM模块

#### DKIM详述

在任意邮箱中点击查看原文/原始邮件，可直接查看该邮件所对应的.eml文件。详情参考[DKIM\_WIKI](https://en.wikipedia.org/wiki/DomainKeys\_Identified\_Mail)。其中包含DKIM-Signature字段：

> DKIM-Signature: v=1; a=rsa-sha256; c=relaxed/relaxed; d=foxmail.com;
>
> s=s201512; t=1670419235;
>
> bh=9RqYI6fxZOUZAYcxZV4SvznReZm2Mn7vMx5y5+asYAM=;
>
> h=From:To:Subject:Date;
>
> b=A3Kfzk0KcOfQhiEGJZ5KUpb3ItszuNBCSJ08hhgaGUIuglV4QaTm9BVH9pDmljKl+AIzS4nRZjYFLiRQWN8ZaYh7edwCp7BAV2l2ei27+mlP/7nsCapEFdbM1cyNBoR8lGwJMkMh3HGhCPMLH8c2GQVx5GxdOj+NLVQZGNVrHwk=

* v：DKIM版本
* a：签名所使用的密码学算法
* c：邮件header和body分别对应的标准化算法，分为simple和relaxed两种。标准化算法是一种处理空格与回车、换行等内容的算法
* d：发件方服务提供商域名
* s：selector，由发件方服务提供商自定义，一个域名下可对应多个selector。一个selector对应一对公私钥。domain+selector所对应的公钥可在DNS服务器上查询获得
* t：UNIX时间戳
* bh：body hash，base64编码后的正文哈希
* h：被签名的header字段，由发件方服务提供商选择
* b：base64编码后的签名

通过DKIM验证可以保证邮件发件人且内容未被篡改。

#### DKIM权威记录合约

用于记录DKIM相关数据。其中：

* upgradeDKIM 用户记录保存DKIM数据。
* dkim：为DKIM验证合约提供接口数据。

```solidity
contract DKIMManager is Ownable {


    mapping (bytes => bytes) private allDkim;


    constructor() {}


    function upgradeDKIM(bytes memory name, bytes memory _dkim) public onlyOwner {
        allDkim[name] = _dkim;
    }


    function dkim(bytes memory name) public view returns (bytes memory) {
        return allDkim[name];
    }


}
```

#### DKIM验证合约

所有加密、验证相关功能均在此合约。用户创建、更换设备等操作需要调用verifier功能验证，验证成功证明该动作为账户owner发起。

其中

* verifyProof：负责零知识证明验证（ZKP）。
* equalBase64：负责验证DKIM中bh内容是否正确。
* containsAddress：负责验证是否为用户发起行为。
* sha256Verify：验证DKIM中b参数是否正确。

```solidity
   function verifier(
        address publicKey,
        bytes memory modulus,
        VerifierInfo calldata info
    ) external view returns (bool)  {
        uint[] memory input = getInput(info.hmua, info.bh, info.base);
        //ZKP Verifier
        if (!verifyProof(info.a, info.b, info.c, input)) {
            return false;
        }
        //bh(bytes) == base64(sha1/sha256(Canon-body))
        if (!equalBase64(info.bh, info.body)) {
            return false;
        }
        //Operation ∈ Canon-body
        if (!containsAddress(publicKey, info.body)) {
            return false;
        }
        //b == RSA(base)
        if (!sha256Verify(info.base, info.rb, info.e, modulus)) {
            return false;
        }
        return true;
    }
```
