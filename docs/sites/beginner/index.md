<h1>Opensea-seaport源码解析</h1>
<h2>Opensea主要流程</h2>
7个交易方法：
fulfillBasicOrder
发送基础交易，只能报价单个offer，同时只能执行单个consiration
fulfillOrder  买家只能是msg.sender，并且只能全部交易
fulfillAdvancedOrder  可以通过传参指定买家，可以部分交易
fulfillAvailableOrders 批量交易，并且可以通过orderIndex指定执行哪些订单，没有指定的不进行交易，买家只能是msg.sender
fulfillAvailableAdvancedOrders 批量交易，并且可以通过orderIndex指定执行哪些订单，没有指定的不进行交易，买家可以通过传参指定，并且可以验证交易中的nft是否默克尔树中
matchOrders 批量交易，
matchAdvancedOrders 批量交易，并且可以验证交易中的nft是否默克尔树中

主要函数的作用：
_validateAndFulfillAdvancedOrder：

    _validateOrderAndUpdateStatus：（总之，这一步是为了验证订单是否正确，并且部分交易信息是否准确）
        1、验证时间；
        2、验证分子必须<=分母；如果分子<分母，那么验证订单必须支持部分交易_doesNotSupportPartialFills；
        3、验证consideraion中的数组的数量必须>=totalOriginalConsiderationItems，也就是说执行的数组的数量必须大于最开始的执行总数
        4、生成orderHash；
        5、验证zone是否授权，如果传入了extraData，那么将extraData一起传入进行验证；
        6、验证订单状态和签名；
        7、检查订单是否进行了交易（根据orderStatus中的denominator！=0表示已经进行过交易）
            7.1、如果进行了交易，并且本次提交的分母是1，那么本次将剩余部分全部交易(因为分子必须比分母小且大于0，所以此时分子只能是1，那么分子/分母 = 1，表示全部)；
            7.2、如果进行了交易，并且本次提交的分母 != 已经交易的分母，那么：（和前一次提交的分母不一致，那么已经交易的分子和分母都 * 本次提交的分母）
                7.2.1、已经交易的分子 = 已经交易的分子 * 本次提交的分母；
                7.2.2、本次提交的分子 = 本次提交的分子 * 已经交易的分母；
                7.2.3、本次提交的分母 = 本次提交的分母 * 已经交易的分母；
            7.3、如果部分未交易，并且已经交易的分子 + 本次提交的分子 > 本次提交的分母，那么本次提交的分子 = 本次提交的分母 - 已经交易的分子（如果已经交易的和本次提交的分子综合超过了分母，那么本次提交的就要减少）
            7.4、如果部分未交易，已经交易的分子 = 已经交易的分子 + 本次提交的分子
            （7.3和7.4是为了保证，本次执行完以后的分子<=分母）
            7.5、如果部分未交易，已经交易的分子、本次提交的分母必须 < type(uint120)所能表示的最大值;
            7.6、如果部分未交易，更新当前已经交易的分子，已经交易的分母 = 本次提交的分母
        8、如果订单没有任何交易，那么设置订单的分母是当前提交的分母，设置订单的分子是当前提交的分子；

    _applyCriteriaResolversAdvanced ： （验证criteriaResolvers,这一步应该是通过默克尔树验证订单中的nft是否在报价或者执行的nft列表中）
        1、验证criteriaResolvers的orderIndex必须是0；
        2、如果criteriaResolver.side == Side.OFFER,那么验证orderParameters.offer中的index是criteriaResolver.index的nft，是否在criteriaResolver.criteriaProof 默克尔树
        3、如果criteriaResolver.side == Side.CONSIDERATION,那么验证orderParameters.consideration中的index是criteriaResolver.index的nft，是否在criteriaResolver.criteriaProof 默克尔树

    _applyFractionsAndTransferEach：（）
        1、报价不能是eth报价，只能是nft；
        2、如果startAmount==endAmount，那么最终的交易数量是endAmount * 分子/分母；否则，按照start_time和end_time的差值做一个比例，根据startAmount和endAmount的在这段时间内的权重，计算这段时间内的平均数量；
        3、转移erc721:
            3.1、如果没有指定conduitkey，那么就是offerer提前进行了授权，直接调用nft合约的transferFrom转账；
            3.2、如果指定了conduitkey，那么调用conduit合约中的excute进行转账；

_fulfillAvailableAdvancedOrders：
    _validateOrdersAndPrepareToFulfill
        1、_validateOrderAndUpdateStatus
        2、如果offer中存在ETH报价，但是调用的方法不是matchAdvancedOrders 或者 matchOrders，那么报错，终止运行；
        3、maximumFulfilled 表示最大的填充数量，如果是0，表示不填充，也就是不进行任何交易；
        4、重新验证并且计算orderToExecute的数量；
        5、校验每一个订单的填充的分子；


    _executeAvailableFulfillments
        解析offeritem和considrationitem，然后获得最终转账的结构体数组


