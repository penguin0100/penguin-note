
## Java 代码文档

```markdown
# 转账业务校验框架 · 完整代码（每行注释）

> 基于 Spring Boot，可直接复制使用。每行注释说明“做什么、有什么用”。

---

## 1. 校验器统一接口

```java
package com.example.business.validator;

/**
 * 所有业务校验器必须实现的接口。
 * 方法内校验不通过时直接抛出 BusinessException，实现快速失败。
 */
public interface Validator {
    /**
     * 执行校验
     * @param request 请求上下文对象（可根据业务替换为具体类型）
     * @throws BusinessException 校验失败时抛出
     */
    void validate(Object request);
}
```

---

## 2. 账户存在性校验器

```java
package com.example.business.validator.impl;

import com.example.business.exception.BusinessException;   // 自定义业务异常
import com.example.business.constant.ErrorCode;           // 错误码常量
import com.example.business.mapper.AccountMapper;         // 账户数据访问层
import com.example.business.validator.Validator;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

/**
 * 校验付款账户是否存在。
 * 执行顺序为最高优先级（1），先保证账户有效再检查其他规则。
 */
@Component  // 交给Spring管理，自动扫描为Bean
@Order(1)   // 数字越小越先执行
public class AccountExistValidator implements Validator {

    @Autowired  // 自动注入账户Mapper，避免手动new
    private AccountMapper accountMapper;

    @Override
    public void validate(Object request) {
        // 将请求对象转为具体类型（实际项目可改为自定义类型）
        TransferRequest req = (TransferRequest) request;
        // 查询付款账户
        Account account = accountMapper.selectById(req.getFromAccountId());
        if (account == null) {
            // 账户不存在，直接抛出业务异常，中断后续所有校验
            throw new BusinessException(ErrorCode.ACCOUNT_NOT_FOUND);
        }
    }
}
```

---

## 3. 金额合法性校验器

```java
package com.example.business.validator.impl;

import com.example.business.exception.BusinessException;
import com.example.business.constant.ErrorCode;
import com.example.business.validator.Validator;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;
import java.math.BigDecimal;

/**
 * 校验转账金额是否合法（必须大于0，小数位数符合要求）。
 */
@Component
@Order(2)  // 在存在性校验之后
public class AmountValidator implements Validator {

    @Override
    public void validate(Object request) {
        TransferRequest req = (TransferRequest) request;
        // 金额必须大于0
        if (req.getAmount().compareTo(BigDecimal.ZERO) <= 0) {
            throw new BusinessException(ErrorCode.AMOUNT_INVALID);
        }
        // 可选：校验小数位数（例如只允许两位小数）
        // if (req.getAmount().scale() > 2) { ... }
    }
}
```

---

## 4. 余额充足性校验器

```java
package com.example.business.validator.impl;

import com.example.business.exception.BusinessException;
import com.example.business.constant.ErrorCode;
import com.example.business.mapper.AccountMapper;
import com.example.business.validator.Validator;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;
import java.math.BigDecimal;

/**
 * 校验付款账户余额是否足够支付转账金额。
 */
@Component
@Order(3)  // 在金额合法和账户存在之后
public class BalanceValidator implements Validator {

    @Autowired
    private AccountMapper accountMapper;

    @Override
    public void validate(Object request) {
        TransferRequest req = (TransferRequest) request;
        // 查询当前余额（一般使用BigDecimal处理金额）
        BigDecimal balance = accountMapper.getBalance(req.getFromAccountId());
        // 比较余额和转账金额
        if (balance.compareTo(req.getAmount()) < 0) {
            throw new BusinessException(ErrorCode.INSUFFICIENT_BALANCE);
        }
    }
}
```

---

## 5. 校验链管理器（TransferValidator）

```java
package com.example.business.validator;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;
import java.util.List;

/**
 * 转账校验链，负责按顺序执行所有 Validator。
 */
@Component
public class TransferValidator {

    /**
     * Spring 会自动将所有实现了 Validator 接口的 Bean 注入此列表，
     * 并根据 @Order 排序。
     */
    @Autowired
    private List<Validator> validators;

    /**
     * 顺序执行全部校验，任一失败抛出异常终止。
     * @param request 业务请求对象
     */
    public void validateAll(Object request) {
        for (Validator validator : validators) {
            // 调用校验，不通过时异常会跳出循环
            validator.validate(request);
        }
    }
}
```

---

## 6. 业务异常类

```java
package com.example.business.exception;

/**
 * 统一业务异常，携带错误码。
 */
public class BusinessException extends RuntimeException {
    private String code;  // 错误码，如 ACCOUNT_NOT_FOUND

    public BusinessException(String code) {
        super(code);
        this.code = code;
    }

    public String getCode() {
        return code;
    }
}
```

---

## 7. 错误码常量类

```java
package com.example.business.constant;

/**
 * 集中管理所有业务错误码，避免硬编码字符串。
 */
public class ErrorCode {
    private ErrorCode() {} // 私有构造，禁止实例化

    public static final String ACCOUNT_NOT_FOUND = "ACCOUNT_NOT_FOUND";
    public static final String PAYEE_NOT_FOUND = "PAYEE_NOT_FOUND";
    public static final String AMOUNT_INVALID = "AMOUNT_INVALID";
    public static final String INSUFFICIENT_BALANCE = "INSUFFICIENT_BALANCE";
    public static final String ACCOUNT_FROZEN = "ACCOUNT_FROZEN";
    public static final String DAILY_LIMIT_EXCEEDED = "DAILY_LIMIT_EXCEEDED";
}
```

---

## 8. 全局异常处理器

```java
package com.example.business.handler;

import com.example.business.exception.BusinessException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

/**
 * 全局异常拦截，将业务异常转为统一 JSON 响应。
 */
@RestControllerAdvice  // 相当于 @ControllerAdvice + @ResponseBody
public class GlobalExceptionHandler {

    /**
     * 处理所有 BusinessException
     */
    @ExceptionHandler(BusinessException.class)
    public Result handleBusinessException(BusinessException e) {
        // 根据错误码构造统一返回对象（可进一步从资源文件获取可读消息）
        return Result.fail(e.getCode());
    }

    // 可添加其他异常处理，如参数校验异常、系统异常等
}

/**
 * 统一返回结果类示例（项目通常已有此封装）
 */
class Result {
    private String code;
    private String message;
    private Object data;

    public static Result fail(String code) {
        Result r = new Result();
        r.code = code;
        r.message = "操作失败"; // 实际可用MessageSource根据code获取国际化消息
        return r;
    }
    // 省略 getter/setter...
}
```

---

## 9. 转账业务服务（核心 Service）

```java
package com.example.business.service;

import com.example.business.validator.TransferValidator;
import com.example.business.mapper.AccountMapper;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

/**
 * 转账核心服务，整合校验与业务逻辑。
 */
@Service
public class TransferService {

    @Autowired
    private TransferValidator transferValidator;  // 注入校验链

    @Autowired
    private AccountMapper accountMapper;          // 数据库操作

    /**
     * 执行转账
     * @param request 转账请求
     */
    @Transactional  // 开启事务，保证扣款与入账要么全部成功要么全部回滚
    public void transfer(TransferRequest request) {
        // 1. 执行前置校验链，失败时抛出异常由全局处理器捕获
        transferValidator.validateAll(request);

        // 2. 原子扣款：UPDATE account SET balance = balance - ? WHERE id=? AND balance >= ?
        int debitRows = accountMapper.debit(request.getFromAccountId(), request.getAmount());
        if (debitRows == 0) {
            // 理论上校验通过后不会出现，但作为兜底
            throw new BusinessException(ErrorCode.INSUFFICIENT_BALANCE);
        }

        // 3. 收款方入账
        accountMapper.credit(request.getToAccountId(), request.getAmount());

        // 4. 记录交易流水（可选）
        // transactionLogMapper.insert(...);
    }
}