hello， 这是有态度马甲的第xxx篇原创口水文。有趣指数5颗星，有用指数5颗星。

😠😠本文是国外技术网站medium上点赞超过200+的翻译/笔记文，有关[规避/解决幂等请求](https://github.com "幂等")的编程指南。

## 1. 软件领域二次请求无法避免

我们生活的每时每刻都是独一无二的，事情/动作可能不会相同的形式再次发生。

在软件领域，同一动作请求并不总会只产生一次，这可能会带来一些问题： 想象你月底发薪，公司的转账指令错误的触发了2次，这岂不是双倍快乐。

## 为什么幂等性很重要？

* 网络不可靠：客户端超时后，可以放心地重试幂等的请求（如PUT, DELETE），而不用担心产生意外后果。
* 分布式系统：在微服务架构中，服务间的重试机制依赖于幂等性来保证数据一致性。

| 二次请求的来源 | 能避免出现吗？ | 怎么避免出现？ |
| --- | --- | --- |
| 前端的频繁点击提交 | 能 | 提交后置灰按钮/提交后切换页面/防误触来解决 |
| 客户端/中间服务器的重试动作 | 不能 | - |

![image](https://img2024.cnblogs.com/blog/587720/202510/587720-20251019135932833-1374211496.png)

根据双将军理论，即使A/B将军不断确认收到对方的上一条信息， 也没办法确保对方与自己达成(同一时间攻击的共识)。

两将军问题是无解的，间歇性重试是一种工程解。 （还有散弹打鸟）

：我们一直发送相同的服务请求，直到我们确定收到它（虽然可能会多次收到）， 这就叫至少一次交付。

但是我们不希望被扣款两次，那我们就必须**确保多次处理相同的请求不会改变最初的应用状态**， 这是幂等请求的重点。

> 除此之外，重试还可能带来 重试风暴、资源雪崩等衍生问题。

## 2. 某些请求天然幂等，你不需要做什么

想象你正在银行开户。

```
|  |  |
| --- | --- |
|  | public sealed class Account |
|  | { |
|  | public Guid Id { get; } |
|  | public decimal Balance { get; private set; } |
|  |  |
|  | public Account(Guid id, decimal balance) |
|  | { |
|  | if (id == default) |
|  | throw new InvalidOperationException("Account id must be provided"); |
|  |  |
|  | if (balance < 0) |
|  | throw new InvalidOperationException("Balance cannot be negative"); |
|  |  |
|  | Id = id; |
|  | Balance = balance; |
|  | } |
|  |  |
|  | // 取钱 |
|  | public void Withdraw(decimal amount) |
|  | { |
|  | if (amount < 0) |
|  | throw new InvalidOperationException("Cannot withdraw negative amount"); |
|  |  |
|  | if (amount > Balance) |
|  | throw new InvalidOperationException("Cannot withdraw more than existing balance"); |
|  |  |
|  | Balance -= amount; |
|  | } |
|  |  |
|  | // 存钱 |
|  | public void Deposit(decimal amount) |
|  | { |
|  | if (amount < 0) |
|  | throw new InvalidOperationException("Cannot deposit negative amount"); |
|  |  |
|  | Balance += amount; |
|  | } |
|  | } |
```

前端发起的开户请求`OpenAccountRequest`是幂等的， 只需要在开户逻辑里面检查 数据表是不是存在这个`AccountId`。

你甚至可在数据库**设置AccountId为唯一索引**，让重试动作爆出异常。

```
|  |  |
| --- | --- |
|  | public async Task HandleAsync(OpenAccountRequest request, CancellationToken token = default) |
|  | { |
|  | var account = new Account(request.AccountId, request.Balance); |
|  |  |
|  | try |
|  | { |
|  | await _repository.InsertAsync(account, token); |
|  | } |
|  | catch (DuplicateKeyException) |
|  | { |
|  | //Ignore |
|  | } |
|  | } |
```

---

对于存钱(WithDraw）取钱（Deposit）就不行了，如果因为网络原因而重试了2次存钱请求(deposit)，岂不就是双倍快乐。

## 3. 乐观锁的介入一定合理吗？

一种处理重复请求的方式是**质询实体的状态**，严格意义来讲， 这个方案是来解决更大叙事背景（乐观锁）下的方案。

首先我们知道高并发场景下，有一个叫乐观锁的并发控制机制，乐观地认为数据在操作时不会冲突， 因此在操作前不加锁，在提交时检查数据是否被修改。

文中一开始： 让前端在请求时带上需要保护的`Balance`,
在更新时利用`AccountId+原Balance`来定位并更新账户。

```
|  |  |
| --- | --- |
|  | // 下面的前端DTO需要带上账户余额，（二次请求也是这个值）。 |
|  | public sealed class DepositToAccountRequest |
|  | { |
|  | public Guid AccountId { get; } |
|  | public decimal Amount { get; }   // 操作金额 |
|  | public decimal AccountBalance { get; } |
|  |  |
|  | public DepositToAccountRequest(Guid accountId, decimal amount, decimal accountBalance) |
|  | { |
|  | AccountId = accountId; |
|  | Amount = amount; |
|  | AccountBalance = accountBalance; |
|  | } |
|  | } |
|  |  |
```

```
|  |  |
| --- | --- |
|  | public async Task HandleAsync(DepositToAccountRequest request, CancellationToken token = default) |
|  | { |
|  | var account = await _repository.GetAsync(request.AccountId, token) ?? |
|  | throw new EntityNotFoundException(); |
|  |  |
|  | account.Deposit(request.Amount); |
|  |  |
|  | await _repository.UpdateAsync(account, request.AccountBalance, token); |
|  |  |
|  |  |
|  | public sealed class AccountRepository : IAccountRepository |
|  | { |
|  | //.... |
|  |  |
|  | public async Task UpdateAsync(Account account, decimal expectedBalance, CancellationToken token = default) |
|  | { |
|  | var sql = "UPDATE Accounts SET Balance = @Balance WHERE Id = @Id AND Balance = @ExpectedBalance"; |
|  | var sqlParams = new |
|  | { |
|  | Id = account.Id, |
|  | Balance = account.Balance,  // 新余额 |
|  | ExpectedBalance = expectedBalance  // 原余额 |
|  | }; |
|  |  |
|  | await using var connection = new SqlConnection(_connectionString); |
|  | await connection.OpenAsync(token); |
|  |  |
|  | var rowsAffected = await connection.ExecuteAsync(sql, sqlParams); |
|  | if (rowsAffected == 0) |
|  | throw new InvalidStateException(); |
|  | } |
|  |  |
|  | //.... |
|  | } |
```

读者肯定也发现了：

① 这个方式不灵活，如果不是Balance，或者不只是Balance, 那么这个sql逻辑就得变化；

② 另一方面，这个方式归根到底不识别重复请求，不知道这是重复请求，还是底层的数据真的发生了变化。

想象你被触发了第二次取钱请求， 若此时刚好有人给你存了一笔钱（刚好等于你第一次取钱金额），促使你的第二次取钱请求成功了，这岂不是新的双倍悲伤。

所以文中提出了基于宏达叙事的正经方案： **前端介入 + 状态版本**
**在前端DTO请求带上`AccountVersion`，每次更新时用`AccoundId+原AccountVersion`去定位、更新状态版本， 如果where条件失败说明实体状态已经变化，需要报错给到前端，让前端重新拉取数据， 如果where条件成功，则说明状态版本无变更，递增version，并给到前端。**

```
|  |  |
| --- | --- |
|  | public async Task UpdateAsync(Account account, int expectedVersion, CancellationToken token = default) |
|  | { |
|  | var sql = "UPDATE Accounts SET Balance = @Balance, Version = @Version WHERE Id = @Id AND Version = @ExpectedVersion"; |
|  | var sqlParams = new |
|  | { |
|  | Id = account.Id, |
|  | Balance = account.Balance, |
|  | Version = account.Version, |
|  | ExpectedVersion = expectedVersion |
|  | }; |
|  |  |
|  | await using var connection = new SqlConnection(_connectionString); |
|  | await connection.OpenAsync(token); |
|  |  |
|  | var rowsAffected = await connection.ExecuteAsync(sql, sqlParams); |
|  | if (rowsAffected == 0) |
|  | throw new InvalidStateException(); |
|  | } |
```

---

![image](https://img2024.cnblogs.com/blog/587720/202510/587720-20251019144446991-1841103696.png)

给一个grafana 修改数据源的示例。

```
|  |  |
| --- | --- |
|  | curl 'https://grafana-chinese.observe.dev.eks.gainetics.io/api/datasources/uid/tempo' \ |
|  | -X 'PUT' \ |
|  | -H 'content-type: application/json' \ |
|  | --data-raw '{"id":2,"uid":"tempo","orgId":1,"name":"Tempo","type":"tempo","typeLogoUrl":"public/plugins/tempo/img/tempo_logo.svg","access":"proxy","url":"http://tempo:3200","user":"","database":"","basicAuth":false,"basicAuthUser":"","withCredentials":false,"isDefault":true,"jsonData":{"pdcInjected":false,"tracesToLogsV2":{"customQuery":false,"datasourceUid":"opensearch","filterBySpanID":true,"filterByTraceID":true,"spanEndTimeShift":"1m","spanStartTimeShift":"-1m","tags":[{"key":"beast","value":""}]}},"secureJsonFields":{},"version":18,"readOnly":false,"accessControl":{"alert.instances.external:read":true,"alert.instances.external:write":true,"alert.notifications.external:read":true,"alert.notifications.external:write":true,"alert.rules.external:read":true,"alert.rules.external:write":true,"datasources.id:read":true,"datasources:delete":true,"datasources:query":true,"datasources:read":true,"datasources:write":true},"apiVersion":""}' |
```

里面有一个`version`就是状态版本，每次前端尝试去更细时， 会带上version，去后端定位。

```
|  |  |
| --- | --- |
|  | ds = &datasources.DataSource{ |
|  | ID:              cmd.ID, |
|  | OrgID:           cmd.OrgID, |
|  | ..... |
|  | Version:         cmd.Version + 1, |
|  | ..... |
|  | } |
|  |  |
|  | var updateSession *xorm.Session |
|  | if cmd.Version != 0 { |
|  | // the reason we allow cmd.version > db.version is make it possible for people to force |
|  | // updates to datasources using the datasource.yaml file without knowing exactly what version |
|  | // a datasource have in the db. |
|  | updateSession = sess.Where("id=? and org_id=? and version < ?", ds.ID, ds.OrgID, ds.Version) |
|  | } else { |
|  | updateSession = sess.Where("id=? and org_id=?", ds.ID, ds.OrgID) |
|  | } |
|  |  |
|  | affected, err := updateSession.Update(ds) |
|  | if err != nil { |
|  | return err |
|  | } |
```

这种乐观锁的思想去解决幂等问题有一个小弊端， 因为乐观锁的思想本是针对并发控制，它解决了并发请求中的重复请求这一子集场景，但是带来的副作用就是高并发时，很多请求会被拒绝(重试请求会被拒绝，并发请求也会被拒绝)，效率变低，但数据不一致问题没有了，双倍悲伤也不会有。

## 4. 用数据库事务包围 更简单、常规

你有一张表来存储 requestId的历史记录， 这个表保证requestId唯一。

那么通过事务： requestId先插入历史记录表、 实际的请求动作，便可以真实解决幂等问题， 这是真的幂等， 因为这个事务真正识别出了重复请求。

```
|  |  |
| --- | --- |
|  | public sealed class AccountRepository : IAccountRepository |
|  | { |
|  | //.... |
|  |  |
|  | public async Task UpdateAsync(Account account, Guid requestId, CancellationToken token = default) |
|  | { |
|  | var requestSql = "INSERT INTO RequestIds VALUES (@Id)"; |
|  | var requestSqlParams = new |
|  | { |
|  | Id = requestId.ToString() |
|  | }; |
|  |  |
|  | var accountSql = "UPDATE Accounts SET Balance = @Balance WHERE Id = @Id"; |
|  | var accountSqlParams = new |
|  | { |
|  | Id = account.Id, |
|  | Balance = account.Balance |
|  | }; |
|  |  |
|  | await using var connection = new SqlConnection(_connectionString); |
|  | await connection.OpenAsync(token); |
|  | await using var transaction = await connection.BeginTransactionAsync(token); |
|  |  |
|  | try |
|  | { |
|  | await connection.ExecuteAsync(requestSql, requestSqlParams); |
|  | } |
|  | catch (Exception e) when (IsDuplicateKeyException(e)) |
|  | { |
|  | throw new DuplicateKeyException(); |
|  | } |
|  |  |
|  | await connection.ExecuteAsync(accountSql, accountSqlParams); |
|  | await transaction.CommitAsync(token); |
|  | } |
|  |  |
|  | //.... |
|  | } |
```

还可对上面的requestId历史记录表做优化，不用一直记录该id，弄一个进程周期性清理这个表。

## 总结

1. 没有最佳的方式去处理幂等，只有最合适的。
2. 有些业务天然幂等， 使用简单的全局唯一id就可以定位出二次请求。
3. 如果你的实体更新的不频繁， 可以考虑使用基于乐观锁的版本状态来解决（总体上乐观锁是更宏达叙事的一个思路，在频繁更新场景下能处理幂等问题，但体验不佳，是一味猛药）。
4. 更常见的幂等解决方式是：基于数据库的ACID事务理论，利用事务识别出二次请求，整个动作直接面向数据库， 是真正的实现了幂等语义。

🙏🏻🙏🏻🙏🏻

[https://medium.com/swlh/retry-requests-fearlessly-with-idempotence-f6bc23f1c721](https://github.com):[楚门加速器下载](https://chuman.org)
