
## 1
- 《Go 1.18 源码剖析》1. 初始化：https://www.yuque.com/docs/share/0f4adfee-e9f6-4306-bc15-1653e0bb1d06?#
- 《1.1 引导》；https://www.yuque.com/docs/share/58fbf5d6-3932-4e73-a1f6-849566fbd45e?# 《1.2 初始化》
- 《Go 1.18 源码剖析》内存分配器：https://www.yuque.com/docs/share/77c62e26-05fa-4f4d-8fdd-cc110d641987?#

## 2《Go 1.18 源码剖析》内存分配器：

- 2.1 定义
    - https://www.yuque.com/docs/share/77c62e26-05fa-4f4d-8fdd-cc110d641987?# 《2.1 定义》

- 2.2 结构
    - https://www.yuque.com/docs/share/696ae233-0a3d-4ad0-83e4-921d4be1a9ce?# 《2.2.1 地址空间》
    - https://www.yuque.com/docs/share/66ea2ddd-8ef6-4ede-8e8c-5045b691aec1?# 《2.2.2 堆内存管理》
    - https://www.yuque.com/docs/share/fa7d6149-ada8-4453-bdd5-31685a86a096?# 《2.2.3 系统内存申请》

- 2.3 分配
    - https://www.yuque.com/docs/share/6b0482c2-882b-4475-9d0e-f8ad083fb50d?# 《2.3 分配》
    - https://www.yuque.com/docs/share/83a951d3-eb32-427f-b1ad-4aec1ae4f210?# 《2.3.1 零长度对象》
    - https://www.yuque.com/docs/share/3c209de6-d025-47ce-86f2-0b064d20ce79?# 《2.3.2 大对象》
    - https://www.yuque.com/docs/share/c1e60157-d1ba-4762-9326-0856e182c4a4?# 《2.3.3 小对象》
    - https://www.yuque.com/docs/share/33dcce35-3e03-413b-a4c4-6b0f1c741379?# 《2.3.4 微小对象》

- 2.4 回收
    - https://www.yuque.com/docs/share/03a7ccf4-4ee6-4cf8-8cdb-57fdd9ff4c17?# 《2.4 回收》

- 2.5 释放
    - https://www.yuque.com/docs/share/a366a78e-d2cc-4c8c-ad65-402c91d6f0ad?# 《2.5.1 同步释放》
    - https://www.yuque.com/docs/share/964e6e7b-2d02-4fd1-891b-1591cae2f9f4?# 《2.5.2 异步释放》
    - https://www.yuque.com/docs/share/2f100d25-c885-4d19-981a-4e57381c595b?# 《2.5.3 物理内存》

- 2.6 其他
    - https://www.yuque.com/docs/share/e0fdae4e-90cf-4541-abe8-b5f8640ddf8b?# 《2.6.1 固定分配器》
    - https://www.yuque.com/docs/share/9ebb5a0c-bc53-4e4e-944b-f149b72a685e?# 《2.6.2 内存块记录》


## 3《Go 1.18 源码剖析》垃圾回收器
- GC
    - https://www.yuque.com/docs/share/9fcad442-fa5f-4d22-b00e-51bba3b10268?# 《3.1 概述》
    - https://www.yuque.com/docs/share/a8fb7bd1-753e-4393-ad99-33c4378a1731?# 《3.2 初始化》
    - https://www.yuque.com/docs/share/d1d6e7b2-3c8a-4481-b7ce-294843e2de24?# 《3.3 启动》
    - https://www.yuque.com/docs/share/91a86b51-0ee7-4ae2-9078-3580dc998174?# 《3.3.1 触发》
    - https://www.yuque.com/docs/share/6e1e6759-a8be-4c8a-936d-713cf5714a8e?# 《3.3.2 启动》
    - https://www.yuque.com/docs/share/5683019e-2234-4307-b57b-76399239f593?# 《3.4 标记》
    - https://www.yuque.com/docs/share/956ee268-a1ef-4175-89c3-3b4e450e02a7?# 《3.4.1 流程》
    - https://www.yuque.com/docs/share/7b1cf954-4309-4f2e-835d-8645f78f1abe?# 《3.4.2 标记》
    - https://www.yuque.com/docs/share/f72f3d4c-7619-45f6-8345-77aa9abe449a?# 《3.4.3 辅助》
    - https://www.yuque.com/docs/share/ba534e15-50e4-40ac-a4a3-682d478fa55f?# 《3.5 清理》
    - https://www.yuque.com/docs/share/98bc2047-f54c-4702-b612-5d7ad1f5dd0e?# 《3.6.1 写屏障》
    - https://www.yuque.com/docs/share/86b8c527-86ef-443b-bde6-b056e795bbc3?# 《3.6.2 标记队列》

## 4《Go 1.18 源码剖析》调度器

- 初始化

    - https://www.yuque.com/docs/share/7f560daf-fb3d-4da9-ba1d-e662dca30e6b?# 《4.2.1 P》
    - https://www.yuque.com/docs/share/980033e0-48ee-4ba9-a8ba-87d11e951aac?# 《4.2.2 STW》
    - https://www.yuque.com/docs/share/a485c51d-214f-4648-9aef-158b3791c494?# 《4.2.3 GOMAXPROC》'

- 任务

    - https://www.yuque.com/docs/share/d6bec2dc-b0df-4ac7-a198-9160731cca24?# 《4.3.1 G》
    - https://www.yuque.com/docs/share/2a11ac8d-60ab-4a54-9899-3da31569f644?# 《4.3.2 新建》
    - https://www.yuque.com/docs/share/4dc3286a-ad22-402b-af2d-b31584f051da?# 《4.3.3 复用》
    - https://www.yuque.com/docs/share/51a3bdb1-f06a-4d96-8317-8a4eb48b79be?# 《4.3.4 队列》

- 线程

    - https://www.yuque.com/docs/share/b1ec1eaf-353a-491b-8202-2f26109e9121?# 《4.4.1 新建》
    - https://www.yuque.com/docs/share/6810a4e4-d968-4efe-b66a-b9f01eeb6a62?# 《4.4.2 闲置》
    - https://www.yuque.com/docs/share/705b9494-5505-460a-b77f-ddde75fecc3b?# 《4.4.3 唤醒》

-  执行

    - https://www.yuque.com/docs/share/79b3bfd9-d3a6-4d81-ad0b-a627d0d5d21d?# 《4.5.1 调度》
    - https://www.yuque.com/docs/share/954698d2-0f39-4ca2-8cb9-f5dee44eca85?# 《4.5.2 查找》
    - https://www.yuque.com/docs/share/3968ca1c-2eb2-4599-9f2e-4a1ba6a40816?# 《4.5.3 执行》
    - https://www.yuque.com/docs/share/edc8ed80-f4f0-439c-93bf-670a24ea2e7b?# 《4.5.4 内核调用》
    - https://www.yuque.com/docs/share/56f80ea2-b3d9-4f39-9edf-7bd4b30ab8f3?# 《4.5.5 终止线程》
    - https://www.yuque.com/docs/share/bd780518-baf3-4135-9a41-10c54b42332e?# 《4.5.6 内核函数》
    - https://www.yuque.com/docs/share/ff6fd4bc-af90-45ad-abd4-4f2a7615ff1f?# 《4.5.7 标准库函数》

- 栈

    - https://www.yuque.com/docs/share/a57e787a-9b8a-421b-ae12-3e1ab2a3c4d2?# 《4.6.1 布局》
    - https://www.yuque.com/docs/share/12d2e092-cfc5-4ce3-a017-87b7b7e27b5c?# 《4.6.2 分配》
    - https://www.yuque.com/docs/share/e62837fc-d158-4f83-b892-a320dbfff5a9?# 《4.6.3 释放》
    - https://www.yuque.com/docs/share/83acb400-e921-4d6f-973a-5ad2da68cf0c?# 《4.6.4 池》
    - https://www.yuque.com/docs/share/a56d5b6f-8951-4e4e-87b4-36d28ed99f0e?# 《4.6.5 扩容》
    - https://www.yuque.com/docs/share/0e89325d-6742-4694-80a0-b3acc70c7bdb?# 《4.6.6 垃圾回收》

- 其他
    - https://www.yuque.com/docs/share/958ac1b8-0728-4732-bce4-902f604b606a?# 《4.7.1 系统调用》
    - https://www.yuque.com/docs/share/f81c49f7-b410-4383-8ce9-cae18e290c8c?# 《4.7.2 系统监控》
    - https://www.yuque.com/docs/share/bccb10ee-d5a2-4152-8e54-0a25b1667cee?# 《4.7.3 抢占调度》
    - https://www.yuque.com/docs/share/2d07b8a9-02be-4e1e-a29c-c276f245a097?# 《4.7.4 网络轮询》
    - https://www.yuque.com/docs/share/5a0631a2-2126-4b65-9a8d-918203733d4f?# 《4.7.5 定时器》

## 5 Go 1.18 源码剖析》并发通道
- concurrency
    - https://www.yuque.com/docs/share/e6621239-7e91-4286-9abc-322af0a26924?# 《5.1 创建》
    - https://www.yuque.com/docs/share/a8c76f5f-2784-4491-9a72-441b34a5d471?# 《5.2 收发》
    - https://www.yuque.com/docs/share/5ef74d3e-52a3-449d-a583-c766d2a5ac65?# 《5.3 选择》

## 6 defer panic conventions
- 关键字
    - https://www.yuque.com/docs/share/44dfe7ad-c745-4a46-a663-23d07c38ddc7?# 《6.1 defer》；
    - https://www.yuque.com/docs/share/31d8a036-8e12-4631-bed4-387299f35377?# 《6.2 panic》；
    - https://www.yuque.com/docs/share/d3ec4cbe-038c-47d1-b8a1-39aa5ad69643?# 《8.1 call conventions》

## 7《Go 1.18 源码剖析》

- 终结器

    - https://www.yuque.com/docs/share/6f89694e-3972-4a4b-978d-68e4368731e8?# 《7.1 设置》
    - https://www.yuque.com/docs/share/8fa5f6ff-8080-4bac-af8f-b5ce1b47d520?# 《7.2 队列》
    - https://www.yuque.com/docs/share/702cd675-98bf-4591-b775-f01a935bbe90?# 《7.3 执行》

## 8其他
- others
    - https://www.yuque.com/docs/share/d3ec4cbe-038c-47d1-b8a1-39aa5ad69643?# 《8.1 调用约定》
    - https://www.yuque.com/docs/share/e125a9cc-88c4-41b4-997d-5f9a60f7cc2e?# 《8.2 接口》
    - https://www.yuque.com/docs/share/384a56d5-eab9-4856-84cc-d02d61f39ef7?# 《8.3 切片》
    - https://www.yuque.com/docs/share/f5e4b391-4ef9-4a93-964e-9d994e2740d8?# 《8.4 字典》
    - https://www.yuque.com/docs/share/859e22b4-c8e4-4edf-b3c6-4a341af8cee9?# 《8.5 锁》
