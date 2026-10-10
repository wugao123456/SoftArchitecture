# 带领域服务的电商下单转账示例

下面这版把「跨实体/跨聚合的业务规则」抽成 **领域服务**，对比上一版能明显看出区别。

```
src/
├── domain/                    # 领域层（核心，不依赖框架）
│   ├── Money.java                    值对象
│   ├── Order.java                    聚合根
│   ├── Account.java                  聚合根
│   ├── OrderRepository.java          端口
│   ├── AccountRepository.java        端口
│   ├── PaymentGateway.java           端口
│   └── PaymentDomainService.java     ★ 领域服务（跨聚合业务规则）
├── application/               # 应用层
│   └── OrderApplicationService.java  应用服务（用例编排）
├── infrastructure/            # 基础设施层
│   ├── InMemoryOrderRepository.java
│   ├── InMemoryAccountRepository.java
│   └── FakePaymentGateway.java
└── presentation/              # 用户界面层
    ├── OrderController.java
    └── Main.java
```

---

## 1. 领域层 Domain

### 1.1 值对象 Money

```java
package domain;

public record Money(long cents) {
    public Money {
        if (cents < 0) throw new IllegalArgumentException("金额不能为负");
    }
    public Money add(Money o) { return new Money(this.cents + o.cents); }
    public Money subtract(Money o) {
        if (o.cents > this.cents) throw new IllegalStateException("余额不足");
        return new Money(this.cents - o.cents);
    }
    public boolean isGreaterThan(Money o) { return this.cents > o.cents; }
    @Override public String toString() { return "¥" + String.format("%.2f", cents / 100.0); }
}
```

### 1.2 聚合根 Order

```java
package domain;

public class Order {
    public enum Status { CREATED, PAID, CANCELLED }

    private final String id;
    private final String buyerId;
    private final String merchantId;
    private final Money amount;
    private Status status = Status.CREATED;

    public Order(String id, String buyerId, String merchantId, Money amount) {
        this.id = id;
        this.buyerId = buyerId;
        this.merchantId = merchantId;
        this.amount = amount;
    }

    /** 实体内的规则：状态机 */
    public void markPaid() {
        if (status != Status.CREATED) throw new IllegalStateException("订单状态不允许支付：" + status);
        this.status = Status.PAID;
    }
    public void cancel() {
        if (status != Status.CREATED) throw new IllegalStateException("订单状态不允许取消：" + status);
        this.status = Status.CANCELLED;
    }

    public String id()          { return id; }
    public String buyerId()     { return buyerId; }
    public String merchantId()  { return merchantId; }
    public Money amount()       { return amount; }
    public Status status()      { return status; }
}
```

### 1.3 聚合根 Account

```java
package domain;

public class Account {
    private final String id;
    private Money balance;

    public Account(String id, Money balance) {
        this.id = id;
        this.balance = balance;
    }

    /** 实体内规则：扣款不能透支 */
    public void debit(Money amount) {
        if (balance.isGreaterThan(amount) == false && !balance.equals(amount) && amount.isGreaterThan(balance)) {
            // 简化：直接让 Money.subtract 报错
        }
        this.balance = balance.subtract(amount);
    }
    public void credit(Money amount) { this.balance = balance.add(amount); }

    public String id()      { return id; }
    public Money balance()  { return balance; }
}
```

### 1.4 端口（仓储 / 网关接口）

```java
package domain;
import java.util.Optional;

public interface OrderRepository {
    void save(Order order);
    Optional<Order> findById(String id);
}

public interface AccountRepository {
    Optional<Account> findById(String id);
    void save(Account account);
}

public interface PaymentGateway {
    boolean transfer(String fromAccountId, String toAccountId, Money amount);
}
```
（三个接口各占一个文件，这里合并展示。）

### 1.5 ★ 领域服务 PaymentDomainService

这是本次重点。为什么需要它？因为「一笔订单能不能付款」这条规则要同时看 **Order 的状态、买家的余额、买家≠商户、手续费计算**——横跨多个聚合，放在 Order 里访问不到 Account，放在 Account 里访问不到 Order，都不合适。这时就该领域服务出场。

```java
package domain;

/** 领域服务：承载跨聚合的业务规则，无状态、不碰数据库/HTTP */
public class PaymentDomainService {

    /** 每笔交易固定手续费 1 元 */
    private static final Money FEE = new Money(100);

    /** 规则 1：校验这笔支付是否合法（横跨 Order + Account） */
    public void validate(Order order, Account buyer, Account merchant) {
        if (order.status() != Order.Status.CREATED) {
            throw new IllegalStateException("订单状态不允许支付：" + order.status());
        }
        if (buyer.id().equals(merchant.id())) {
            throw new IllegalStateException("买家与商户不能是同一账户");
        }
        if (!order.buyerId().equals(buyer.id())) {
            throw new IllegalStateException("账户与订单买家不匹配");
        }
        if (!order.merchantId().equals(merchant.id())) {
            throw new IllegalStateException("账户与订单商户不匹配");
        }
        Money total = order.amount().add(FEE);
        if (!buyer.balance().equals(total) && buyer.balance().isGreaterThan(total) == false) {
            throw new IllegalStateException(
                "余额不足，需要 " + total + "，当前 " + buyer.balance());
        }
    }

    /** 规则 2：计算买家实付金额（含手续费） */
    public Money totalPayable(Order order) {
        return order.amount().add(FEE);
    }

    /** 规则 3：真正执行账户间划转（先扣买家、再给商户，原子性由应用层事务保证） */
    public void executeTransfer(Order order, Account buyer, Account merchant) {
        Money total = totalPayable(order);
        buyer.debit(total);
        merchant.credit(order.amount());   // 商户只收订单金额，手续费归平台
    }
}
```

注意：这个领域服务**只依赖领域对象**，不认识数据库、不认识网关、不认识 HTTP。所以它能在纯 JUnit 里被单测。

---

## 2. 应用层 Application

应用服务只做编排：查、校验、调网关、更新状态、保存。**它不写“余额够不够”“买家和商户能不能相同”这类规则**——那些都在领域服务里。

```java
package application;

import domain.*;
import java.util.UUID;

public class OrderApplicationService {

    private final OrderRepository orderRepository;
    private final AccountRepository accountRepository;
    private final PaymentGateway paymentGateway;
    private final PaymentDomainService paymentDomainService;   // 注入领域服务

    public OrderApplicationService(OrderRepository orderRepository,
                                   AccountRepository accountRepository,
                                   PaymentGateway paymentGateway,
                                   PaymentDomainService paymentDomainService) {
        this.orderRepository = orderRepository;
        this.accountRepository = accountRepository;
        this.paymentGateway = paymentGateway;
        this.paymentDomainService = paymentDomainService;
    }

    /** 用例：下单 */
    public String placeOrder(String buyerId, String merchantId, long amountCents) {
        Order order = new Order(UUID.randomUUID().toString(),
                                buyerId, merchantId, new Money(amountCents));
        orderRepository.save(order);
        return order.id();
    }

    /** 用例：支付订单（编排） */
    public void payOrder(String orderId) {
        // 1. 取聚合
        Order order = orderRepository.findById(orderId)
                .orElseThrow(() -> new IllegalArgumentException("订单不存在：" + orderId));
        Account buyer = accountRepository.findById(order.buyerId())
                .orElseThrow(() -> new IllegalArgumentException("买家账户不存在"));
        Account merchant = accountRepository.findById(order.merchantId())
                .orElseThrow(() -> new IllegalArgumentException("商户账户不存在"));

        // 2. 交给领域服务校验（业务规则）
        paymentDomainService.validate(order, buyer, merchant);

        // 3. 调外部支付网关（技术动作）
        boolean ok = paymentGateway.transfer(buyer.id(), merchant.id(),
                                             paymentDomainService.totalPayable(order));
        if (!ok) throw new IllegalStateException("支付失败，请重试");

        // 4. 领域服务执行账户划转
        paymentDomainService.executeTransfer(order, buyer, merchant);

        // 5. 更新订单状态 + 落库（生产环境用 @Transactional 包住 4~5）
        order.markPaid();
        orderRepository.save(order);
        accountRepository.save(buyer);
        accountRepository.save(merchant);
    }

    public Order queryOrder(String orderId) {
        return orderRepository.findById(orderId)
                .orElseThrow(() -> new IllegalArgumentException("订单不存在：" + orderId));
    }

    public Account queryAccount(String accountId) {
        return accountRepository.findById(accountId)
                .orElseThrow(() -> new IllegalArgumentException("账户不存在：" + accountId));
    }
}
```

---

## 3. 基础设施层 Infrastructure

```java
package infrastructure;

import domain.*;
import java.util.Map;
import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;

public class InMemoryOrderRepository implements OrderRepository {
    private final Map<String, Order> store = new ConcurrentHashMap<>();
    @Override public void save(Order o)              { store.put(o.id(), o); }
    @Override public Optional<Order> findById(String id) { return Optional.ofNullable(store.get(id)); }
}
```

```java
package infrastructure;

import domain.*;
import java.util.Map;
import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;

public class InMemoryAccountRepository implements AccountRepository {
    private final Map<String, Account> store = new ConcurrentHashMap<>();
    @Override public Optional<Account> findById(String id) { return Optional.ofNullable(store.get(id)); }
    @Override public void save(Account a)                  { store.put(a.id(), a); }
}
```

```java
package infrastructure;

import domain.Money;
import domain.PaymentGateway;

public class FakePaymentGateway implements PaymentGateway {
    @Override
    public boolean transfer(String from, String to, Money amount) {
        System.out.printf("[支付网关] %s → %s 转账 %s ... 成功%n", from, to, amount);
        return true;
    }
}
```

---

## 4. 用户界面层 Presentation

```java
package presentation;

import application.OrderApplicationService;
import domain.Account;
import domain.Order;

public class OrderController {

    private final OrderApplicationService orderService;

    public OrderController(OrderApplicationService orderService) {
        this.orderService = orderService;
    }

    /** POST /orders */
    public String placeOrder(String buyerId, String merchantId, long amountCents) {
        if (buyerId == null || buyerId.isBlank()) throw new IllegalArgumentException("buyerId 不能为空");
        return orderService.placeOrder(buyerId, merchantId, amountCents);
    }

    /** POST /orders/{id}/pay */
    public void pay(String orderId) { orderService.payOrder(orderId); }

    /** GET /orders/{id} */
    public String queryOrder(String orderId) {
        Order o = orderService.queryOrder(orderId);
        return "Order{id=" + o.id() + ", buyer=" + o.buyerId()
             + ", amount=" + o.amount() + ", status=" + o.status() + "}";
    }

    /** GET /accounts/{id} */
    public String queryAccount(String id) {
        Account a = orderService.queryAccount(id);
        return "Account{id=" + a.id() + ", balance=" + a.balance() + "}";
    }
}
```

```java
package presentation;

import application.OrderApplicationService;
import domain.*;
import infrastructure.*;

public class Main {
    public static void main(String[] args) {
        // 1. 装配（组合根）
        OrderRepository   orderRepo   = new InMemoryOrderRepository();
        AccountRepository accountRepo = new InMemoryAccountRepository();
        PaymentGateway    gateway     = new FakePaymentGateway();

        // 预置两个账户：买家 500 元，商户 0 元
        accountRepo.save(new Account("buyer-001",    new Money(50000)));
        accountRepo.save(new Account("merchant-001", new Money(0)));

        PaymentDomainService domainService = new PaymentDomainService();  // 领域服务
        OrderApplicationService app = new OrderApplicationService(
                orderRepo, accountRepo, gateway, domainService);
        OrderController controller = new OrderController(app);

        // 2. 下单（199 元）
        String orderId = controller.placeOrder("buyer-001", "merchant-001", 19900);
        System.out.println("下单成功：" + controller.queryOrder(orderId));

        // 3. 支付（199 + 1 手续费 = 200 元）
        controller.pay(orderId);
        System.out.println("支付后：" + controller.queryOrder(orderId));
        System.out.println("买家：" + controller.queryAccount("buyer-001"));
        System.out.println("商户：" + controller.queryAccount("merchant-001"));

        // 4. 重复支付 → 被领域服务拦截
        try { controller.pay(orderId); }
        catch (IllegalStateException e) { System.out.println("被领域拦截：" + e.getMessage()); }

        // 5. 余额不足场景
        String order2 = controller.placeOrder("buyer-001", "merchant-001", 99900);
        try { controller.pay(order2); }
        catch (IllegalStateException e) { System.out.println("被领域拦截：" + e.getMessage()); }
    }
}
```

运行输出：

```
下单成功：Order{id=..., buyer=buyer-001, amount=¥199.00, status=CREATED}
[支付网关] buyer-001 → merchant-001 转账 ¥200.00 ... 成功
支付后：Order{id=..., buyer=buyer-001, amount=¥199.00, status=PAID}
买家：Account{id=buyer-001, balance=¥300.00}
商户：Account{id=merchant-001, balance=¥199.00}
被领域拦截：订单状态不允许支付：PAID
被领域拦截：余额不足，需要 ¥1000.00，当前 ¥300.00
```

---

## 一图看清“应用服务 vs 领域服务”

```
Presentation  →  ApplicationService  →  DomainService  →  Entity
   (HTTP)          (编排用例)              (跨聚合规则)      (聚合内规则)
                                                
 参数校验       查仓储、调网关            余额足不足？      状态能不能变？
 转 DTO        开事务、发事件            买=商户吗？      扣款别透支
               不写业务规则              手续费怎么算？
```

| | 应用服务 `OrderApplicationService` | 领域服务 `PaymentDomainService` |
|---|---|---|
| 所在层 | application | domain |
| 依赖谁 | 端口（仓储、网关）+ 领域服务 + 实体 | 只依赖实体/值对象 |
| 是否知道数据库/HTTP | 通过端口知道“要用”，但不知道实现 | 完全不知道 |
| 是否可脱离框架单测 | 需要 mock 端口 | **可以直接 new 出来测** |
| 写业务规则吗 | 不写 | 写（跨聚合的那部分） |
| 本示例中的规则 | “先校验 → 再转账 → 再改状态 → 再落库” | “余额够不够、买家≠商户、手续费多少、划转顺序” |

**判断该不该用领域服务**：一条规则**只涉及一个聚合** → 放实体方法；**横跨多个聚合或需要外部数据但仍是纯业务规则** → 放领域服务；**只是流程编排或调外部系统** → 放应用服务。