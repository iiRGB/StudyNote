[TOC]

# 前言

从数据持久层到表现层，校验数据是贯穿在所有应用层的常见任务。通常在每层实现相同的验证逻辑，耗时且易出错。为了避免这些重复的校验，开发者们通常直接将验证逻辑绑定到业务模型，将模型类与验证代码(实际上是关于类本身的元数据)混合在一起。

![application layers](C:\Users\Administrator\Documents\本地笔记\积累\Hibernate Validator\application-layers.png)

JSR 380 - Bean Validation 2.0 为实体和方法校验定义了元数据模型和API。 默认的元数据源都是注释，并且具有通过使用XML覆盖和扩展元数据的能力。API不绑定到特定的应用层或编程模型。它不是与web或持久化层特别相关的，它既可用于服务器端应用程序编程，也可用于富客户端Swing应用程序开发人员。

![application layers2](C:\Users\Administrator\Documents\本地笔记\积累\Hibernate Validator\application-layers2.png)

Hibernate Validator 是 JSR 380 的实现。实现的本身和 Bean Validation API 、TCK的提供和分发基于 [Apache Software License 2.0](http://www.apache.org/licenses/LICENSE-2.0)。

Hibernate Validator 6 和 Bean Validation 2.0 需要 Java 8 或更高版本.

# 1. 入门

这个章节将会向你展示如何入门 Hibernate Validator, 以及 Bean Validation的实现。接下来的`快速开始`需要：

- JDK 8
- [Apache Maven](http://maven.apache.org/)
- 网络连接（Maven需要下载依赖库）

## 1.1 项目设置

为了在 Maven 项目中使用 Hibernate Validator，添加如下依赖到你的`pom.xml`文件中：

##### *例 1.1: Hibernate Validator 的 Maven 依赖*

```xml
<dependency>
    <groupId>org.hibernate.validator</groupId>
    <artifactId>hibernate-validator</artifactId>
    <version>6.0.8.Final</version>
</dependency>
```

该依赖会将 Bean Validation API （`javax.validation:validation-api:2.0.1.Final`）一同下载。

### 1.1.1 统一EL

Hibernate Validator 需要统一的EL表达式语言([JSR 341](http://jcp.org/en/jsr/detail?id=341)) 用于得到约束变量信息（[见章节 4.1, “Default message interpolation”](http://docs.jboss.org/hibernate/stable/validator/reference/en-US/html_single/#section-message-interpolation)）。如果你的应用运行在类如 JBoss 的 Java EE 容器中，容器就已经提供了EL表达式的实现。但是如果在 Java SE 环境中，就必须添加该实现的依赖到你的 POM 文件中。例如可以添加如下 JSR 341的  [参考实现](https://javaee.github.io/uel-ri/) 依赖。

##### *例 1.2：统一EL的参考实现 MAVEN 依赖*

```xml
<dependency>
    <groupId>org.glassfish</groupId>
    <artifactId>javax.el</artifactId>
    <version>3.0.1-b09</version>
</dependency>
```

> 对环境中不能提供EL实现的， Hibernate验证器给出了 [12.9节"ParameterMessageInterpolator "](http://docs.jboss.org/hibernate/stable/validator/reference/en-US/html_single/#non-el-message-interpolator)。 然而，这种插值器并不符合 Bean验证规范。

### 1.1.2 CDI

Bean Validation 通过 CDI （Contexts and Dependency Injection for Java TM EE 上下文和依赖注入, [JSR 346](http://jcp.org/en/jsr/detail?id=346)）定义了集成点。如果您的应用程序运行在没有提供这种集成容器环境，你可以通过添加以下Maven依赖你的POM使用 Hibernate Validator CDI 便携扩展：

##### *例1.3:Hibernate Validator CDI便携扩展Maven的依赖*

``` xml
<dependency>
    <groupId>org.hibernate.validator</groupId>
    <artifactId>hibernate-validator-cdi</artifactId>
    <version>6.0.8.Final</version>
</dependency>
```

需要注意的是，添加所需的这种依赖性通常是一个无需运行在 Java EE 环境的应用程序。 您可以了解更多关于Bean验证和CDI的集成在 [11.3节 “CDI” ](http://docs.jboss.org/hibernate/stable/validator/reference/en-US/html_single/#section-integration-with-cdi)。

### 1.1.3 运行在安全管理器中

Hibernate Validator支持运行在[安全管理器](http://docs.oracle.com/javase/8/docs/technotes/guides/security/index.html)中。要做到这一点，必须指定几个权限给 Hibernate验证框架的基础代码，相关类，Bean验证API，和JBoss日志和调用Bean验证的代码。 下面显示了如何通过这 [策略文件 ](http://docs.oracle.com/javase/8/docs/technotes/guides/security/PolicyFiles.html)实现，就像Java默认策略实现处理。

##### *例1.4:在安全管理器中使用Hibernate验证框架的策略文件*

``` shell
grant codeBase "file:path/to/hibernate-validator-6.0.8.Final.jar" {
    permission java.lang.reflect.ReflectPermission "suppressAccessChecks";
    permission java.lang.RuntimePermission "accessDeclaredMembers";
    permission java.lang.RuntimePermission "setContextClassLoader";

    permission org.hibernate.validator.HibernateValidatorPermission "accessPrivateMembers";

    // Only needed when working with XML descriptors (validation.xml or XML constraint mappings)
    permission java.util.PropertyPermission "mapAnyUriToUri", "read";
};

grant codeBase "file:path/to/validation-api-2.0.1.Final.jar" {
    permission java.io.FilePermission "path/to/hibernate-validator-6.0.8.Final.jar", "read";
};

grant codeBase "file:path/to/jboss-logging-3.3.2.Final.jar" {
    permission java.util.PropertyPermission "org.jboss.logging.provider", "read";
    permission java.util.PropertyPermission "org.jboss.logging.locale", "read";
};

grant codeBase "file:path/to/classmate-1.3.4.jar" {
    permission java.lang.RuntimePermission "accessDeclaredMembers";
};

grant codeBase "file:path/to/validation-caller-x.y.z.jar" {
    permission org.hibernate.validator.HibernateValidatorPermission "accessPrivateMembers";
};
```

### 1.1.4 在WildFly Hibernate验证框架

 [WildFly应用服务器 ](http://wildfly.org/) 中Hibernate验证框架开箱即用。 为了更新服务器模块中 最新最好的Bean Validation API和Hibernate验证框架的，可以使用WildFly补丁机制。

您可以从 [SourceForge ](http://sourceforge.net/projects/hibernate/files/hibernate-validator)下载补丁文件或从Maven中央使用以下依赖：

*Example 1.5: 用于 WildFly 12.0.0.Final 补丁文件的Maven依赖f*

```xml
<dependency>
    <groupId>org.hibernate.validator</groupId>
    <artifactId>hibernate-validator-modules</artifactId>
    <version>6.0.8.Final</version>
    <classifier>wildfly-12.0.0.Final-patch</classifier>
    <type>zip</type>
</dependency>
```

我们同样提供了用于 WildFly 11.0.0.FInal 的补丁：

*例1.6：用于WildFly 11.0.0.Final补丁文件的依赖*

```xml
<dependency>
    <groupId>org.hibernate.validator</groupId>
    <artifactId>hibernate-validator-modules</artifactId>
    <version>6.0.8.Final</version>
    <classifier>wildfly-11.0.0.Final-patch</classifier>
    <type>zip</type>
</dependency>
```

下载补丁文件后，你可以通过运行这条命令把它应用到WildFly：

*例1.7:应用WildFly补丁*

```shell
$JBOSS_HOME/bin/jboss-cli.sh patch apply hibernate-validator-modules-6.0.8.Final-wildfly-12.0.0.Final-patch.zip
```

如果您想在服务器中取消补丁或回滚到Hibernate Validator最初的版本，运行以下命令:

*例1.8:回滚WildFly补丁*

```shell
$JBOSS_HOME/bin/jboss-cli.sh patch rollback --reset-configuration=true
```

你可以了解更多关于WildFly补丁结构一般 [在这里 ](https://developer.jboss.org/wiki/SingleInstallationPatching/)和 [在这里 ](http://www.mastertheboss.com/jboss-server/jboss-configuration/managing-wildfly-and-eap-patches)。

### 1.1.5 运行在Java 9

在Hibernate Validator 6.0.8.Final中，对Java 9和Java平台模块化系统(jpms)的支持是实验性的。现在还没有提供模块描述，但Hibernate验证器自动模块是可用的。

这些是使用`Automatic-Module-Name`头声明的模块名称：

- Bean Validation API: `java.validation`
- Hibernate Validator 核心: `org.hibernate.validator`
- Hibernate Validator CDI 扩展: `org.hibernate.validator.cdi`
- Hibernate Validator 测试单元: `org.hibernate.validator.testutils`
- Hibernate Validator 注解处理器: `org.hibernate.validator.annotationprocessor`

这些模块名称是初步的，在将来的版本中提供真正的模块描述符时可能会更改。

如果使用 Bean Validation XML描述（*META-INF/validation.xml* 或者约束匹配文件），`java.xml.bind` 模块必须被启动。通过添加 `--add-modules java.xml.bind`到你的 java 调用中。

> 如果使用Hibernate Validator CDI，主要不要启用JDK 的 `java.xml.ws.annotation `模块的。 这个模块包含了JSR 250 API的一个子集(“Commons Annotations通用”)，一些注解如 `javax.annotation.Priority `等会丢失。 这会导致Hibernate验证框架的方法验证拦截器不会被注册，即方法验证行不通。
>
> 替换方案，将完整的JSR 250 API添加到未命名的模块(即类路径)，如通过 *javax.annotation:javax.annotation-api* 依赖 (当依赖*org.hibernate.validator:hibernate-validator-cdi*时已经有一个传递依赖 JSR 250 API )。
>
> 如果由于某种原因您需要启用 `java.xml.ws.annotation `模块，你应该将完整的API 内容通过 `——patch-module java.xml.ws.annotation = /path/to/complete-jsr250-api.jar `添加到你的 *java* 调用。



## 1.2 应用约束

让我们直接深入到一个例子如何应用约束。

*例1.9：带约束注解的汽车类*

```java 
package org.hibernate.validator.referenceguide.chapter01;

import javax.validation.constraints.Min;
import javax.validation.constraints.NotNull;
import javax.validation.constraints.Size;

public class Car {

    @NotNull
    private String manufacturer;

    @NotNull
    @Size(min = 2, max = 14)
    private String licensePlate;

    @Min(2)
    private int seatCount;

    public Car(String manufacturer, String licencePlate, int seatCount) {
        this.manufacturer = manufacturer;
        this.licensePlate = licencePlate;
        this.seatCount = seatCount;
    }

    //getters and setters ...
}
```

`@NotNull `, `@Size `和 `@Min `注解声明了应该应用在汽车实例属性上的约束 :

- `manufacturer` 必须永远不能为 `null`
- `licensePlate` 必须永远不能为 `null` 同时必须在 2-14 字符范围中
- `seatCount` 必须至少为2 

> 你可以找在GitHub [Hibernate Validator 源码库 ](https://github.com/hibernate/hibernate-validator/tree/master/documentation/src/test)上找到所有指南中完整的源代码。

## 1.3 验证约束

执行验证这些约束，可以使用 `Validator `实例。 让我们看一看  `Car `的单元测试:

##### *例1.10：Car类显示验证的例子*

``` JAVA
package org.hibernate.validator.referenceguide.chapter01;

import java.util.Set;
import javax.validation.ConstraintViolation;
import javax.validation.Validation;
import javax.validation.Validator;
import javax.validation.ValidatorFactory;

import org.junit.BeforeClass;
import org.junit.Test;

import static org.junit.Assert.assertEquals;

public class CarTest {

    private static Validator validator;

    @BeforeClass
    public static void setUpValidator() {
        ValidatorFactory factory = Validation.buildDefaultValidatorFactory();
        validator = factory.getValidator();
    }

    @Test
    public void manufacturerIsNull() {
        Car car = new Car( null, "DD-AB-123", 4 );

        Set<ConstraintViolation<Car>> constraintViolations =
                validator.validate( car );

        assertEquals( 1, constraintViolations.size() );
        assertEquals( "must not be null", constraintViolations.iterator().next().getMessage() );
    }

    @Test
    public void licensePlateTooShort() {
        Car car = new Car( "Morris", "D", 4 );

        Set<ConstraintViolation<Car>> constraintViolations =
                validator.validate( car );

        assertEquals( 1, constraintViolations.size() );
        assertEquals(
                "size must be between 2 and 14",
                constraintViolations.iterator().next().getMessage()
        );
    }

    @Test
    public void seatCountTooLow() {
        Car car = new Car( "Morris", "DD-AB-123", 1 );

        Set<ConstraintViolation<Car>> constraintViolations =
                validator.validate( car );

        assertEquals( 1, constraintViolations.size() );
        assertEquals(
                "must be greater than or equal to 2",
                constraintViolations.iterator().next().getMessage()
        );
    }

    @Test
    public void carIsValid() {
        Car car = new Car( "Morris", "DD-AB-123", 2 );

        Set<ConstraintViolation<Car>> constraintViolations =
                validator.validate( car );

        assertEquals( 0, constraintViolations.size() );
    }
}
```

在`setUp()`方法中，从`ValidatorFactory`中得到 `Validator`验证器   。 一个 `Validator` 实例是线程安全的，可以多次重复使用。 因此可以安全地存储在一个静态属性中，用于验证测试方法中不同的 `Car `实例。

 `validate() `方法返回一组 `ConstraintViolation `实例，您可以遍历 为了看到哪些验证错误发生。 前三个测试方法显示了一些预期约束违反:

-  `manufacturerIsNull()`中违反了`manufacturer`的 `@NotNull`约束
- `licensePlateTooShort()` 违反了 `licensePlate`的 `@Size`约束
-  `seatCountTooLow()`违反了`seatCount`的 `@Min`约束 

在`carIsValid() `可以看出，如果对象验证成功, `validate() `返回一个空集 。

注意，只有 `javax.validation `包的类被使用了。 这些类由Bean Validation API 提供。 没有从Hibernate Validator类应用，使得代码预备移植性。

## 1.4 接下来去哪？

总结了5分钟Hibernate验证器和Bean验证的教程。 继续探索引用的代码示例或看进一步的例子 [第14章, *进一步的阅读*](#3. 声明和验证方法约束)。

学习更多关于bean类的验证和属性，继续阅读[*第二章, 声明和验证bean约束*](#2. 声明和验证Bean约束)。 如果你有兴趣使用Bean验证来校验方法的前置和后置条件，[第三章, 声明和验证方法约束](#3. 声明和验证方法约束)。 如果您的应用程序 有特殊验证需求，看一看 [第六章, *创建自定义约束* ](http://docs.jboss.org/hibernate/stable/validator/reference/en-US/html_single/#validator-customconstraints)。

# 2. 声明和验证Bean约束

在本章中你将学习如何声明(见 [2.1节,“声明bean约束” ](#2.1 声明Bean约束)), 验证(见 [2.2节,“bean验证约束” ](http://docs.jboss.org/hibernate/stable/validator/reference/en-US/html_single/#section-validating-bean-constraints))bean约束。 [2.3节,“内置约束” ](http://docs.jboss.org/hibernate/stable/validator/reference/en-US/html_single/#section-builtin-constraints)概述了所有Hibernate验证器内置的约束 。

如果你有兴趣将约束应用于方法参数和返回值,请参考 [第三章, *声明和验证方法约束* ](#3. 声明和验证方法约束)

## 2.1 声明Bean约束

Bean约束是通过Java注解表示的。 在本节中，您将学习如何通过这些注解增强对象模型。 有四种类型的bean的约束:

- 字段的约束
- 属性约束
- 容器元素的约束
- 类的约束

> 并不是所有的约束都可以放置在这些层级上。 事实上,没有一个Bean验证器定义的默认约束可以放置在类级别。 `java.lang.annotation.Target `注释 在约束注解中决定了哪些元素可以放置一个约束。 更多的信息参考 [第六章, *创建自定义约束* ](http://docs.jboss.org/hibernate/stable/validator/reference/en-US/html_single/#validator-customconstraints)。

### 2.1.1 字段级别约束

约束可以通过给类字段添加注解来表达，[例2.1，“字段级别约束”](#*例2.1： 字段级别约束*)展示了一个字段级别的配置实例。

##### *例2.1： 字段级别约束*

```java
package org.hibernate.validator.referenceguide.chapter02.fieldlevel;

public class Car {

    @NotNull
    private String manufacturer;

    @AssertTrue
    private boolean isRegistered;

    public Car(String manufacturer, boolean isRegistered) {
        this.manufacturer = manufacturer;
        this.isRegistered = isRegistered;
    }

    //getters and setters...
}
```

在使用字段级约束时，使用字段访问策略访问要验证的值。这意味着验证引擎直接访问实例变量，即使存在属性访问器方法也不会调用。

约束可以应用于任何访问类型(公共、私有等)的字段。但是，不支持静态字段上的约束。

> 在验证字节码增强对象时，应该使用属性级别的约束，因为字节代码增强库将无法通过反射确定字段访问。

### 2.1.2 属性级约束

如果你的模型类遵循 [javabean ](http://www.oracle.com/technetwork/articles/javaee/spec-136004.html)标准，也可以注解bean类的属性而不是它的字段。 [2.2的例子,“属性级约束” ](#*例2.2：属性级别约束*)，使用与[示例2.1、“字段级约束” ](#*例2.1： 字段级别约束*)相同的实体，但是使用属性级约束。

##### *例2.2：属性级别约束*

```java
package org.hibernate.validator.referenceguide.chapter02.propertylevel;

public class Car {

    private String manufacturer;

    private boolean isRegistered;

    public Car(String manufacturer, boolean isRegistered) {
        this.manufacturer = manufacturer;
        this.isRegistered = isRegistered;
    }

    @NotNull
    public String getManufacturer() {
        return manufacturer;
    }

    public void setManufacturer(String manufacturer) {
        this.manufacturer = manufacturer;
    }

    @AssertTrue
    public boolean isRegistered() {
        return isRegistered;
    }

    public void setRegistered(boolean isRegistered) {
        this.isRegistered = isRegistered;
    }
}
```

> 属性的getter方法必须进行注释，而不是其setter。这也可以限制没有setter方法的只读属性。

使用属性约束属性访问策略对访问值进行验证，即验证引擎通过属性访问器方法访问状态。

> 建议在一个类中只对字段或属性注释。不建议对字段和属性方法同时添加注解，因为这样会导致字段被验证两次。

### 2.1.3 容器元素约束

有时候可能直接在参数化类型的类型参数上指定约束：这些约束称为容器元素约束。

这就要求在约束上定义 `@Target `为 `ElementType.TYPE_USE`。从Bean验证2.0开始，内置的 `Bean Validation` 以及`Hibernate Validator`等特定约束指定了`ElementType.YPE_USE`，并且可以在这个上下文中直接使用。

Hibernate Validator验证在以下标准Java容器上指定的容器元素约束：

- `java.util.Iterable` (如 `List`s, `Set`s)的实现
- `java.util.Map`的实现，支持key和value
- `java.util.Optional`, `java.util.OptionalInt`, `java.util.OptionalDouble`, `java.util.OptionalLong`
- JavaFX的`javafx.beans.observable.ObservableValue`各种实现

它还支持自定义容器类型的容器元素约束(见 [第七章, *值提取* ](# 7. 值提取))。

> 在版本6之前支持容器元素的子集约束，这需要在容器级添加注解 `@Valid `启用它们。 在Hibernate Validator 6这不是必需的。

下面是几个示例，演示各种Java类型上的容器元素约束。

在这些实例中，`@ValidPart`是一个指定了`TYPE_USE`的自定义约束。

#### 2.1.3.1 用于Iterable

当把约束应用在`Iterable`迭代器类型参数上时，Hibernate Validator 会校验每个元素。[例 2.3 “在Set上使用容器元素约束”](# *例 2.3: 在 Set 上使用容器元素约束*)演示了在Set上使用该约束。

##### *例 2.3: 在 Set 上使用容器元素约束*

```java
package org.hibernate.validator.referenceguide.chapter02.containerelement.set;

import java.util.HashSet;
import java.util.Set;

public class Car {

    private Set<@ValidPart String> parts = new HashSet<>();

    public void addPart(String part) {
        parts.add(part);
    }

    //...

}

Car car = new Car();
car.addPart( "Wheel" );
car.addPart( null );

Set<ConstraintViolation<Car>> constraintViolations = validator.validate( car );

assertEquals( 1, constraintViolations.size() );

ConstraintViolation<Car> constraintViolation =
        constraintViolations.iterator().next();
assertEquals(
        "'null' is not a valid car part.",
        constraintViolation.getMessage()
);
assertEquals( "parts[].<iterable element>",
        constraintViolation.getPropertyPath().toString() );
```

注意属性路径是如何清晰地声明违规来自可迭代器的某个元素。

#### 2.1.3.2 用于List

当把约束应用在一个`List`类型参数中，Hibernate 验证器会验证每个元素。[例2.4，“在List中使用容器元素约束”](#*例 2.4：在List中使用容器元素约束*)演示了一个在`List`中使用容器约束的示例。

##### *例 2.4：在List中使用容器元素约束*

```java
package org.hibernate.validator.referenceguide.chapter02.containerelement.list;

public class Car {

    private List<@ValidPart String> parts = new ArrayList<>();

    public void addPart(String part) {
        parts.add( part );
    }

    //...

}
Car car = new Car();
car.addPart( "Wheel" );
car.addPart( null );

Set<ConstraintViolation<Car>> constraintViolations = validator.validate( car );

assertEquals( 1, constraintViolations.size() );

ConstraintViolation<Car> constraintViolation =
        constraintViolations.iterator().next();
assertEquals(
        "'null' is not a valid car part.",
        constraintViolation.getMessage()
);
assertEquals( "parts[1].<list element>",
        constraintViolation.getPropertyPath().toString() );
```

这里，属性路径同样包含了没有通过校验元素的索引。

#### 2.1.3.3 用于Map

容器元素约束同样可以校验Map的key和value。[例 2.5，“在map的key和value中使用容器元素约束”](#*例 2.5：在map的key和value中使用容器元素验证*)演示了该示例。

##### *例 2.5：在map的key和value中使用容器元素验证*

``` java
package org.hibernate.validator.referenceguide.chapter02.containerelement.map;

import java.util.HashMap;
import java.util.Map;

import javax.validation.constraints.NotNull;

public class Car {

    public enum FuelConsumption {
        CITY,
        HIGHWAY
    }

    private Map<@NotNull FuelConsumption, @MaxAllowedFuelConsumption Integer> fuelConsumption = new HashMap<>();

    public void setFuelConsumption(FuelConsumption consumption, int value) {
        fuelConsumption.put( consumption, value );
    }

    //...

}
Car car = new Car();
car.setFuelConsumption( Car.FuelConsumption.HIGHWAY, 20 );

Set<ConstraintViolation<Car>> constraintViolations = validator.validate( car );

assertEquals( 1, constraintViolations.size() );

ConstraintViolation<Car> constraintViolation =
        constraintViolations.iterator().next();
assertEquals(
        "20 is outside the max fuel consumption.",
        constraintViolation.getMessage()
);
assertEquals(
        "fuelConsumption[HIGHWAY].<map value>",
        constraintViolation.getPropertyPath().toString()
);
Car car = new Car();
car.setFuelConsumption( null, 5 );

Set<ConstraintViolation<Car>> constraintViolations = validator.validate( car );

assertEquals( 1, constraintViolations.size() );

ConstraintViolation<Car> constraintViolation =
        constraintViolations.iterator().next();
assertEquals(
        "must not be null",
        constraintViolation.getMessage()
);
assertEquals(
        "fuelConsumption<K>[].<map key>",
        constraintViolation.getPropertyPath().toString()
);
```

违反约束的属性路径特别有趣：

- 未通过校验的元素的key包含在属性路径中（在第二个示例中，key为 `null`）
- 在第一个示例中，违反约束包含了`<map value>`, 第二个包含了`<map key>`。
- 在第二个实例中，你可能注意到了类型参数`<K>`，后面会有详细说明。

#### 2.1.3.4 在java.util.Optional中

如果将约束应用在`Optional`类型参数中，Hibernate验证器会自动包装类型并验证它的值。[例 2.6，“在jOptional中使用容器元素约束”](#*例2.6：在Optional中使用容器元素约束*)演示了该用法。

##### *例2.6：在Optional中使用容器元素约束*

``` java


package org.hibernate.validator.referenceguide.chapter02.containerelement.optional;

public class Car {

    private Optional<@MinTowingCapacity(1000) Integer> towingCapacity = Optional.empty();

    public void setTowingCapacity(Integer alias) {
        towingCapacity = Optional.of( alias );
    }

    //...

}
```

``` java
Car car = new Car();
car.setTowingCapacity( 100 );

Set<ConstraintViolation<Car>> constraintViolations = validator.validate( car );

assertEquals( 1, constraintViolations.size() );

ConstraintViolation<Car> constraintViolation = constraintViolations.iterator().next();
assertEquals(
        "Not enough towing capacity.",
        constraintViolation.getMessage()
);
assertEquals(
        "towingCapacity",
        constraintViolation.getPropertyPath().toString()
);
```

这里，属性路径只包含了属性名称，因为我们认为`Optional`是一个透明容器。

#### 2.1.3.5 在自定义容器类型中

容器元素约束同样可以使用在自定义容器中。

自定义类型必须注册为 `ValueExtractor `以允许检索值用来验证（关于如何实现自定义的 `ValueExtractor `以及如何注册参考 [第七章, *值提取* ](http://docs.jboss.org/hibernate/stable/validator/reference/en-US/html_single/#chapter-valueextraction)获取更多信息）。

[例 2.7， “在自定义容器中使用容器元素约束”](#*例2.7：在自定义容器中使用容器元素约束*)演示了一个使用参数类型约束的自定义参数类型实例。

##### *例2.7：在自定义容器中使用容器元素约束*

``` java


package org.hibernate.validator.referenceguide.chapter02.containerelement.custom;

public class Car {

    private GearBox<@MinTorque(100) Gear> gearBox;

    public void setGearBox(GearBox<Gear> gearBox) {
        this.gearBox = gearBox;
    }

    //...

}
```

``` java
public class GearBox<T extends Gear> {

    private final T gear;

    public GearBox(T gear) {
        this.gear = gear;
    }

    public Gear getGear() {
        return this.gear;
    }
}
```

``` java


package org.hibernate.validator.referenceguide.chapter02.containerelement.custom;

public class Gear {
    private final Integer torque;

    public Gear(Integer torque) {
        this.torque = torque;
    }

    public Integer getTorque() {
        return torque;
    }

    public static class AcmeGear extends Gear {
        public AcmeGear() {
            super( 60 );
        }
    }
}

```

```java


package org.hibernate.validator.referenceguide.chapter02.containerelement.custom;

public class GearBoxValueExtractor implements ValueExtractor<GearBox<@ExtractedValue ?>> {

    @Override
    public void extractValues(GearBox<@ExtractedValue ?> originalValue, ValueExtractor.ValueReceiver receiver) {
        receiver.value( null, originalValue.getGear() );
    }
}

```

``` java


Car car = new Car();
car.setGearBox( new GearBox<>( new Gear.AcmeGear() ) );

Set<ConstraintViolation<Car>> constraintViolations = validator.validate( car );
assertEquals( 1, constraintViolations.size() );

ConstraintViolation<Car> constraintViolation =
        constraintViolations.iterator().next();
assertEquals(
        "Gear is not providing enough torque.",
        constraintViolation.getMessage()
);
assertEquals(
        "gearBox",
        constraintViolation.getPropertyPath().toString()
);


```

#### 2.1.3.6 嵌套容器元素

约束也同样支持嵌套的容器元素。

 在[例 2.8，“在嵌套容器元素上约束”](#*例 2.8：在嵌套容器元素上约束*)中验证`Car`对象，`Part`和`Manufacturer`将会受`@NotNull`约束。

##### *例 2.8：在嵌套容器元素上约束*

``` java


package org.hibernate.validator.referenceguide.chapter02.containerelement.nested;

import java.util.HashMap;
import java.util.List;
import java.util.Map;

import javax.validation.constraints.NotNull;

public class Car {

    private Map<@NotNull Part, List<@NotNull Manufacturer>> partManufacturers =
            new HashMap<>();

    //...
}
```

### 2.1.4 类层级约束

最后但同样重要的一点是，约束也可以放在类级别上。在这种情况下，完整的对象是约束的主体，而不是任何一个单独属性。如果验证依赖于一个对象的多个属性之间的相关性，那么类级约束是有用的。

在[例 2.9，“类级别约束”](#)中的`Car`类有两个属性 座位数`seatCount` 和乘客`passengers` ，应确保乘客名单上的乘客人数不会超过现有座位。为了实现该目标，在类层级上添加了`@validairercount`约束。该约束的验证器可以访问完整的`Car`对象，从而可以比较座位和乘客的数量。

参考[章节 6.2，“类层级约束”](#)学习更多如何实现该自定义约束的细节。

##### *例2.9：类层级约束*

```java
package org.hibernate.validator.referenceguide.chapter02.classlevel;

@ValidPassengerCount
public class Car {

    private int seatCount;

    private List<Person> passengers;

    //...
}
```

### 2.1.5 约束继承

当一个类实现一个接口或扩展另一个类时，在父类型上声明的所有约束注解都以相同的方式应用在该类本身上。为了说清楚这个，来看一看下面的示例。

##### *例2.10：约束继承*

``` java
package org.hibernate.validator.referenceguide.chapter02.inheritance;

public class Car {

    private String manufacturer;

    @NotNull
    public String getManufacturer() {
        return manufacturer;
    }

    //...
}
```

``` java
package org.hibernate.validator.referenceguide.chapter02.inheritance;

public class RentalCar extends Car {

    private String rentalStation;

    @NotNull
    public String getRentalStation() {
        return rentalStation;
    }

    //...
}
```

类 `RentalCar`是`Car`类的子类，同时新增了属性`rentalStation`，如果 `RentalCar`的实例被校验了，不是只有`rentalStation`上的`@NotNull`约束生效了，还有来自父类的 `manufacturer` 上的约束。

如果`Car`不是父类而是被 `RentalCar`实现的接口，结果同上。

如果方法被重写，方法上的约束注解会被聚合。所以如果 `RentalCar`重写了`Car`类的`getManufacturer()` 方法，重写方法上的所有约束注解都会添加到来自父类的`@NotNull`上。

### 2.1.6 对象图

 Bean Validation API 不仅可以验证单个实例对象，也可以实现对象图（级联验证）。要做到这一点，只需在引用的对象属性上添加注解 `@Valid`，参考[例 2.11，“级联验证”](#*例2.11：级联验证*)。

##### *例2.11：级联验证*

``` java
package org.hibernate.validator.referenceguide.chapter02.objectgraph;

public class Car {

    @NotNull
    @Valid
    private Person driver;

    //...
}
```

``` java
package org.hibernate.validator.referenceguide.chapter02.objectgraph;

public class Person {

    @NotNull
    private String name;

    //...
}
```

如果对象 `Car`





# 3. 声明和验证方法约束





[id]: