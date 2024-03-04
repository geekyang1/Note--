# 手写Spring

## **循环依赖问题**：

**什么是循环依赖**：

![image-20240218163854584](C:\Users\geekyang\AppData\Roaming\Typora\typora-user-images\image-20240218163854584.png)

两个对象都为创建，但是你中有我，我中有你。

Spring是怎么在**set + singleton**（构造方法加单例、构造方法加多例、set加多例都不行）解决循环依赖问题的。

核心方法是**曝光**：

​		什么是曝光那，就是在创建对象的过程中<u>***先创建对象，但是不符值***</u>。

为什么可以这样，这就涉及到了spring创建Bean对象的方式了：**<u>三级缓存</u>**

![img](https://cdn.nlark.com/yuque/0/2022/png/21376908/1665456331018-18c45ae3-fa4c-4cd8-aabf-d9bace567693.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_41%2Ctext_5Yqo5Yqb6IqC54K5%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

在以上类中包含三个重要的属性（源码上第二个是第三级缓存，第三个是第二级缓存）：

***Cache of singleton objects: bean name to bean instance.\*** **单例对象的缓存：key存储bean名称，value存储Bean对象【一级缓存】**

***Cache of early singleton objects: bean name to bean instance.\*** **早期单例对象的缓存：key存储bean名称，value存储早期的Bean对象【二级缓存】**

***Cache of singleton factories: bean name to ObjectFactory.\*** **单例工厂缓存：key存储bean名称，value存储该Bean对应的ObjectFactory对象【三级缓存】**

从源码中看，Spring先从一级缓存中取Bean，取不到再从二级缓存中取，最后还是取不到再从三级缓存中取被提前**曝光**过的ObjectFactory对象，再从中取Bean实例

**从解决循环依赖问题的源码中看到，Spring底层存储Bean的方式是Map集合**

## 1.编写核心接口ApplicationContext及其实现类

![image-20240218161706728](C:\Users\geekyang\AppData\Roaming\Typora\typora-user-images\image-20240218161706728.png)

最开始的时候没有思路，就按照自己写单元测试的时候来入手。

**首先：** 

需要一个核心接口Application  该接口中需要有getBean方法

**其次：**

需要一个实现类ClassPathXMLApplicationContext，其中通过构造方法来解析配置文件，所以该类需要一个解析配置文件的构造方法。

**最后：**

解析配置文件后的Bean对象，该存储到哪？

从三级缓存知识部分了解到，Bean对象其实存在Map集合中去，Key就是Bean的ID，Value就是Bean对象。

```java
package org.myspringframework.core;

import java.util.HashMap;
import java.util.Map;

/**
 * @author 动力节点
 * @version 1.0
 * @className ClassPathXmlApplicationContext
 * @since 1.0
 **/
public class ClassPathXmlApplicationContext implements ApplicationContext{
    /**
     * 存储bean的Map集合
     */
    private Map<String,Object> beanMap = new HashMap<>();

    /**
     * 在该构造方法中，解析myspring.xml文件，创建所有的Bean实例，并将Bean实例存放到Map集合中。
     * @param resource 配置文件路径（要求在类路径当中）
     */
    public ClassPathXmlApplicationContext(String resource) {

    }

    @Override
    public Object getBean(String beanId) {
        return beanMap.get(beanId);
    }
}
```

## 2.实例化Bean

**首先：**

需要先读取XML文件内的内容，通过Dom4j解析文件。

通过类加载器获得输入流，然后SAXReader reader = new SAXReader(配置文件);

**然后：**

获取到对应节点之后，得到Bean的ID和Bean的类路径之后，通过反射机制实例化Bean

因为知道类路径后可以通过 Class.forName(类路径)获取对应类

获取构造方法，创建对象

**最后：**

将Bean的ID Bean的对象存到Map集合中，因为只创建了Bean对象，并没有赋值，这属于曝光。

```java
/**
* 在该构造方法中，解析myspring.xml文件，创建所有的Bean实例，并将Bean实例存放到Map集合中。
* @param resource 配置文件路径（要求在类路径当中）
*/
public ClassPathXmlApplicationContext(String resource) {
    try {
        SAXReader reader = new SAXReader();
        Document document = reader.read(ClassLoader.getSystemClassLoader().getResourceAsStream(resource));
        // 获取所有的bean标签
        List<Node> beanNodes = document.selectNodes("//bean");
        // 遍历集合
        beanNodes.forEach(beanNode -> {
            Element beanElt = (Element) beanNode;
            // 获取id
            String id = beanElt.attributeValue("id");
            // 获取className
            String className = beanElt.attributeValue("class");
            try {
                // 通过反射机制创建对象
                Class<?> clazz = Class.forName(className);
                Constructor<?> defaultConstructor = clazz.getDeclaredConstructor();
                Object bean = defaultConstructor.newInstance();
                // 存储到Map集合
                beanMap.put(id, bean);
            } catch (Exception e) {
                e.printStackTrace();
            }
        });
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

## 3.给实例化的Bean赋值

### 3.1获取并执行set方法

1.赋值时我们需要获得<bean>标签里面的<property>属性，通过property里面的属性获取对应的name属性。

2.通过name属性值 ，加”set“+propertyName.toUpperCase().charAt(0)+propertyName.subString(1)获取类中的set方法名。

3.有set方法名称了，获取set方法（先从Map中获取实例对象，在通过实例对象获取类，然后传递方法名，方法参数类型，返回值三要素就可以获取set方法。又因为set方法没有返回值可以不传）通过Method类中的invoke（）方法来执行set方法。

```java
						// 获取属性名
                        String propertyName = propertyElt.attributeValue("name");
                        // 获取属性类型
                        Class<?> propertyType = 			beanMap.get(beanId).getClass().getDeclaredField(propertyName).getType();
                        // 获取set方法名
                        String setMethodName = "set" + propertyName.toUpperCase().charAt(0) + 			 	propertyName.substring(1);
                        // 获取set方法
                        Method setMethod = beanMap.get(beanId).getClass().getDeclaredMethod(setMethodName, propertyType);
```

set方法的执行需要判断参数类型

如果<property>属性值是value，则是简单类型。如果<property>属性值是ref，则是引用类型。

这里需要两个if

```java
 						// 获取value
                        String propertyValue = propertyElt.attributeValue("value");
                        // 获取ref
                        String propertyRef = propertyElt.attributeValue("ref");
						if (propertyValue != null) 
     					if (propertyRef != null)
```



### 3.2给引用类型赋值

```java
   						// 获取ref
                        String propertyRef = propertyElt.attributeValue("ref");
```

如果是引用类型，那么ref的值是bean容器的id，然后通过容器id获取对象，这就是属性值。

set方法的两个值一个就是属性名，属性值。

```java
 						if (propertyRef != null) {
                            // 该属性不是简单属性
                            setMethod.invoke(beanMap.get(beanId), beanMap.get(propertyRef));
                        }
```



### 3.3给简单类型赋值

```java
 						// 获取value
                        String propertyValue = propertyElt.attributeValue("value");
```

给简单类型赋值的难点在于bean中Value属性的值都是String，我们需要把他转为对象本来属性的类型。

```java
 Class<?> propertyType = beanMap.get(beanId).getClass().getDeclaredField(propertyName).getType();
```

获取Value类型之后，获得简易的String类型的名字（简易：不带包名。转为String是为了跟基本类型进行对比，确定是那种类型。然后才能将Value值转为对应类型）。

如果是简单类型直接调用包装类的parseXXX进行拆箱，也可以使用包装类的ValueOf（Value）

这里使用Object接受数据，因为出现的情况很多。

```java
 						 Object propertyVal = null;	
						if (propertyValue != null) {
                            // 该属性是简单属性
                            String propertyTypeSimpleName = propertyType.getSimpleName();
                            switch (propertyTypeSimpleName) {
                                case "byte": case "Byte":
                                    propertyVal = Byte.valueOf(propertyValue);
                                    break;
                                case "short": case "Short":
                                    propertyVal = Short.valueOf(propertyValue);
                                    break;
                                case "int": case "Integer":
                                    propertyVal = Integer.valueOf(propertyValue);
                                    break;
                                case "long": case "Long":
                                    propertyVal = Long.valueOf(propertyValue);
                                    break;
                                case "float": case "Float":
                                    propertyVal = Float.valueOf(propertyValue);
                                    break;
                                case "double": case "Double":
                                    propertyVal = Double.valueOf(propertyValue);
                                    break;
                                case "boolean": case "Boolean":
                                    propertyVal = Boolean.valueOf(propertyValue);
                                    break;
                                case "char": case "Character":
                                    propertyVal = propertyValue.charAt(0);
                                    break;
                                case "String":
                                    propertyVal = propertyValue;
                                    break;
                            }
                            setMethod.invoke(beanMap.get(beanId), propertyVal);
```

### 3.4完整的Bean赋值代码

```java
             // 再重新遍历集合，这次遍历是为了给Bean的所有属性赋值。
            // 思考：为什么不在上面的循环中给Bean的属性赋值，而在这里再重新遍历一次呢？
            // 通过这里你是否能够想到Spring是如何解决循环依赖的：实例化和属性赋值分开。
            beanNodes.forEach(beanNode -> {
                Element beanElt = (Element) beanNode;
                // 获取bean的id
                String beanId = beanElt.attributeValue("id");
                // 获取所有property标签
                List<Element> propertyElts = beanElt.elements("property");
                // 遍历所有属性
                propertyElts.forEach(propertyElt -> {
                    try {
                        // 获取属性名
                        String propertyName = propertyElt.attributeValue("name");
                        // 获取属性类型
                        Class<?> propertyType = beanMap.get(beanId).getClass().getDeclaredField(propertyName).getType();
                        // 获取set方法名
                        String setMethodName = "set" + propertyName.toUpperCase().charAt(0) + 			 	propertyName.substring(1);
                        // 获取set方法
                        Method setMethod = beanMap.get(beanId).getClass().getDeclaredMethod(setMethodName, propertyType);
                        // 获取属性的值，值可能是value，也可能是ref。
                        // 获取value
                        String propertyValue = propertyElt.attributeValue("value");
                        // 获取ref
                        String propertyRef = propertyElt.attributeValue("ref");
                        Object propertyVal = null;
                        if (propertyValue != null) {
                            // 该属性是简单属性
                            String propertyTypeSimpleName = propertyType.getSimpleName();
                            switch (propertyTypeSimpleName) {
                                case "byte": case "Byte":
                                    propertyVal = Byte.valueOf(propertyValue);
                                    break;
                                case "short": case "Short":
                                    propertyVal = Short.valueOf(propertyValue);
                                    break;
                                case "int": case "Integer":
                                    propertyVal = Integer.valueOf(propertyValue);
                                    break;
                                case "long": case "Long":
                                    propertyVal = Long.valueOf(propertyValue);
                                    break;
                                case "float": case "Float":
                                    propertyVal = Float.valueOf(propertyValue);
                                    break;
                                case "double": case "Double":
                                    propertyVal = Double.valueOf(propertyValue);
                                    break;
                                case "boolean": case "Boolean":
                                    propertyVal = Boolean.valueOf(propertyValue);
                                    break;
                                case "char": case "Character":
                                    propertyVal = propertyValue.charAt(0);
                                    break;
                                case "String":
                                    propertyVal = propertyValue;
                                    break;
                            }
                            setMethod.invoke(beanMap.get(beanId), propertyVal);
                        }
                        if (propertyRef != null) {
                            // 该属性不是简单属性
                            setMethod.invoke(beanMap.get(beanId), beanMap.get(propertyRef));
                        }
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                });
            });

        } catch (Exception e) {
            e.printStackTrace();
        }
```

