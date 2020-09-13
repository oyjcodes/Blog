---
title: lombok之Builder注解和建造者模式
date: 2020-09-10 15:07:45
tags: 
- lombok
categories: "设计模式"  
---
## @Builder注解的使用

### 使用Lombok的@Builder注解修饰类
```java
@Data
@Builder
public class Product {

    private int id;

    private String name;

}
```
### 在IDEA中使用建造者模式创建User对象
```java
 @Test
    public void test(){
        Product product = Product.builder()
                .id(1)
                .name("oyj").build();
    }
```

### 编译之后的Product.class文件
```java
public class Product {
    private int id;
    private String name;

    Product(int id, String name) {
        this.id = id;
        this.name = name;
    }

    public int getId() {
        return this.id;
    }

    public String getName() {
        return this.name;
    }

    public void setId(int id) {
        this.id = id;
    }

    public void setName(String name) {
        this.name = name;
    }

    public boolean equals(Object o) {
        if (o == this) {
            return true;
        } else if (!(o instanceof Product)) {
            return false;
        } else {
            Product other = (Product)o;
            if (!other.canEqual(this)) {
                return false;
            } else if (this.getId() != other.getId()) {
                return false;
            } else {
                Object this$name = this.getName();
                Object other$name = other.getName();
                if (this$name == null) {
                    if (other$name != null) {
                        return false;
                    }
                } else if (!this$name.equals(other$name)) {
                    return false;
                }

                return true;
            }
        }
    }

    protected boolean canEqual(Object other) {
        return other instanceof Product;
    }

    public int hashCode() {
        int PRIME = true;
        int result = 1;
        int result = result * 59 + this.getId();
        Object $name = this.getName();
        result = result * 59 + ($name == null ? 43 : $name.hashCode());
        return result;
    }

    public String toString() {
        return "Product(id=" + this.getId() + ", name=" + this.getName() + ")";
    }

    //下列代码是根据@Builder注解生成的
    public static Product.ProductBuilder builder() {
        return new Product.ProductBuilder();
    }

    //静态内部类
    public static class ProductBuilder {
        private int id;
        private String name;

        ProductBuilder() {
        }

        public Product.ProductBuilder id(int id) {
            this.id = id;
            return this;
        }

        public Product.ProductBuilder name(String name) {
            this.name = name;
            return this;
        }

        public Product build() {
            return new Product(this.id, this.name);
        }

        public String toString() {
            return "Product.ProductBuilder(id=" + this.id + ", name=" + this.name + ")";
        }
    }
}

```

### 构建的步骤
```java
1. 通过Product中的静态方法创建ProductBuilder对象（用于后续构建Product）：builder   //Product.ProductBuilder builder  = Product.builder()

2. 设置Product需要的属性值（链式返回ProductBuilder对象）  // builder = builder.id().name()
             
3. 通过ProductBuilder对象的 build()方法，构建Product对象并返回（其实就是将builder中与Product同样的属性值去构造Product对象）  // Product product = builder.build()
```

## 传统Builder模式

### UML类图

![avatar](1.png)<br>

```txt
Director（导演角色），调用具体构造者创建产品对象，他是负责从客户端传来指令交给具体干活的类。
Builder （抽象建造者），没有具体的业务意义，就是抽象出具体构造者的方法，简单说就是为了多态。
ConcreteBuilder（具体构造者），苦力，实打实的把零件造好，组装好
Product（产品）
```

### 产品类
```java
/**
 * @title: User
 * @description: 产品类
 */
public class Product {

    private int id;

    private String name;

    public Product(int id, String name) {
        this.id = id;
        this.name = name;
    }
}
```

### 抽象建造者

```java
/**
 * @title: Builder
 * @description: 抽象建造者
 */
public abstract class Builder {
    protected int id;

    protected String name;

    public void setId(int id) {
        this.id = id;
    }

    public void setName(String name) {
        this.name = name;
    }

    /**
    　* @Description: 构建产品
    　* @param
    　* @return
     */
    public abstract Product build();
}
```

### 具体建造者

```java
/**
 * @title: ConcreteBuilder
 * @description: 具体建造者
 */
public class ConcreteBuilder extends Builder{
    @Override
    public void setId(int id) {
        super.setId(id);
    }

    @Override
    public void setName(String name) {
        super.setName(name);
    }

    @Override
    public Product build() {
        return new Product(id,name);
    }
}
```

### 导演类
```java
/**
 * @title: Director
 * @description: 导演类
 */
public class Director {
    public Product getProduct(){
        Builder builder = new ConcreteBuilder();
        builder.setId(1);
        builder.setName("apple");
        return  builder.build();
    }
}
```




