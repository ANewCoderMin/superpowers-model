---
name: modelView
description: Use when 需要在 Java 项目中按主键查询、View 构建与 Struct 转换的读侧开发模式完成需求时
---

# View 主导的开发模式

## 目标
通过 **主键查询 + View 构建 + Struct 转换** 的链路，统一读侧口径，减少多表拼装与重复逻辑。

## 核心原则
1. **查询尽量以主键为主**：优先用 id / taskId / xxxId 定位实体
2. **查询结果先构建为 View**：使用 `buildView(...)` 生成 View，再从 View 取关联信息。
3. **跨表访问不直接查库**：通过 `View#getXxxView()`/`getXxxList()` 获取关联数据。
4. **输出统一走 Struct**：Struct 的入参通常是 `View` / `List<View>` / `Page<View>`，不直接用 Model。
5. **转换时允许嵌套 View**：Struct 内部可以使用 View 的 getter 获取其他 View 或派生字段。

## 标准工作流（推荐顺序）
1. **主键查询**：`baseDao.getEntityByKey(id)` 或 `baseDao.listByKeys(ids)`。
2. **构建 View**：`buildView(entity)` / `buildView(list)` / `buildView(page)`。
3. **View 内取关联**：在 View 中调用 `buildOneToOne/buildOneToMany/build(...)`。
4. **Struct 转换**：`struct.convert(view)` / `struct.convert(list)` / `struct.convert(page)`。
5. **返回 VO/DTO**：对外只返回 VO/DTO，不直接返回 Model。

## View 的角色定位
- **读侧聚合容器**：把跨表数据集中到 View 中，避免 Service 里手工拼装。
- **业务口径承载**：时间区间、统计、展示字段都放在 View getter 中统一计算。
- **避免 N+1**：View 获取关联数据依赖模型注解与批量构建上下文，SQL 在构建阶段批量执行。

## Struct 的使用规则
- **只接收 View**：Struct 的入参应是 `View/列表/分页`，不直接接收 Model。
- **通过 View 获取关联**：在 Struct 映射时可以调用 `view.getXxxView()` 获取关联数据。
- **时间/金额格式化**：统一在 Struct 中处理格式化与拼接，不散落在 Service。
- **使用 INSTANCE 静态实例方式**：参考 `CoalSourceAbbreviationStruct`，优先采用 `Mappers.getMapper` 获取实例，不使用 Spring 注入。

### Struct 推荐写法（INSTANCE 静态实例）
```java
@Mapper(uses = DateStructUtil.class)
public interface XxxStruct {
    XxxStruct INSTANCE = Mappers.getMapper(XxxStruct.class);

    XxxVo toVo(XxxView view);

    List<XxxVo> toVoList(List<XxxView> views);
}
```

### 使用方式
```java
XxxVo vo = XxxStruct.INSTANCE.toVo(view);
```

## 示例（简化）
```java
// 1) 主键查询
Task task = baseDao.getEntityByKey(id).orElseThrow(...);
// 2) 构建 View
TaskView view = buildView(task);
// 3) View 内取关联
AddressView addressView = view.getAddressView();
// 4) Struct 转换输出
TaskDetailVo vo = taskStruct.toDetail(view);
return vo;
```

## 禁止/避免事项
- 不在 Service 中手写联表拼装或重复计算逻辑。
- 不直接用 Model 作为返回对象。
- 不在 Struct 中直接查询数据库。

## 适用场景
适用于列表、详情、导出、统计等读场景；写场景仍以 Model + DAO 更新为主。

## View/注解使用指南（新增实体时必读）
### 1）@ManyToOne：子表指向父表（批量一对多）
**场景**：B 表有 `a_id` 外键，想在 `AView` 里拿 `List<BView>`。

**写法（在子表字段上标注）**
```java
public class B {
    @TableField("a_id")
    @ManyToOne(A.class)
    private Integer aId;
}
```

**View 用法**
```java
public class AView extends BaseIntensiveView<A> {
    public List<BView> getBViews() {
        return buildOneToMany(B.class);
    }
}
```

**自动 SQL 形态**
```
select * from b where a_id in (aId1, aId2, ...)
```

### 2）@OneToOne：子表指向父表（单条一对一）
**场景**：B 表以 `a_id` 唯一指向 A，想在 `AView` 里拿 `BView`。

**写法（在子表字段上标注）**
```java
public class B {
    @TableField("a_id")
    @OneToOne(A.class)
    private Integer aId;
}
```

**View 用法**
```java
public class AView extends BaseIntensiveView<A> {
    public BView getBView() {
        return buildOneToOne(B.class);
    }
}
```

**自动 SQL 形态**
```
select * from b where a_id in (aId1, aId2, ...)
```

### 3）@OneMatch：只登记“外键 id”，不触发自动查询
**场景**：A 表里只有 `b_id`，只想**把 id 加入构建上下文**，不希望自动按关系批量查询。

**写法（在 A 的外键字段上标注）**
```java
public class A {
    @TableField("b_id")
    @OneMatch(B.class)
    private Integer bId;
}
```

**View 用法（按 id 获取单个 View）**
```java
public class AView extends BaseIntensiveView<A> {
    public BView getBView() {
        return build(B.class, source.getBId());
    }
}
```

**说明**
- `@OneMatch` 只做 `extractId`，不会注册 `lazyBuild`。
- 查询由 `build(B.class, id)` 触发，SQL 形态是：
```
select * from b where id in (...)
```

## 查询与 VO 转换模板（新增实体时直接套用）
```java
// 1) 主键查询
A a = baseDao.getEntityByKey(id).orElseThrow(...);

// 2) 构建 View
AView view = buildView(a);

// 3) View 内取关联（触发已注册的批量加载）
BView bView = view.getBView();          // OneToOne / OneMatch
List<CView> cViews = view.getCViews();  // ManyToOne

// 4) Struct 转换输出
AVo vo = aStruct.toVo(view);
return vo;
```
