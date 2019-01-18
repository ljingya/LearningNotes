##### Java基础之注解篇

1. ###### 元数据

元数据定义：元数据是添加到元素程序中例如类，属性，方法上的额外信息。

元数据的作用：根据程序元素上的注释创建文档，声明方法重载，是否编译期间检测，代码分析等

java平台的元数据：注解Annotation。

2. ###### 注解（Annotation）

   概念：jdk1.5之后新增，将元数据嵌入到程序，可以通过解析工具和编译工具解析。

   内建注解：java中提供了很多内建注解，例如：@Override，@Deprecated,@SuppressWarning,@FunctionalInterface等。

   内建注解作用：主要提供编译期检测。

   ###### @Override

   告知编译器，复写父类的某个方法，保留在源文件中。

   ###### @Deprecated

   告知编译器，方法过时。适用于除注解类型除外的所有元素。保留时长运行时。

   ###### @SuppressWarning

   告知编译器忽略特定的警告信息,适用于除注解类型和包名之外的所有元素，保留在java源文件中。

   该注解有value()方法，支持多个字符串。

   ```
   public @interface SuppressWarnings {
       String[] value();
   }
   //使用
   @SupressWarning(value={"uncheck","deprecation"})
   ```

   ###### @FunctionalInterface

   检查接口，保证该接口是函数式接口，可适用于注解类型生命，保留时长运行时。

   3. ###### 元Annotation

      java除了提供了内建注解，还提供了6个元Annotation。下面介绍常用的4个注解。

      ######  @Documented

      使用该元Annotation修饰时，会通过javadoc生成注释文档，若定义Annotation时使用该元Annotation，使用定义的Annotation修饰的程序元素，会在APi文档上有对应的Annotation注释。

      ###### @Inherited

      用该元注解修饰的注解具有继承性，解释：某个类使用@Inherited定义的注解，则其子类会自动被定义的注解修饰

      ###### @Retention

      表示注解保留的时长，注解类型未生命@Retention，默认使用RetentionPolicy.CLASS，RetentionPolicy是一个枚举类。

      RetentionPolicy.CLASS:保留在源文件中，Class字节码文件，但在运行时JVM不在保留注释

      RetentionPolicy.RUNTIME:保留在源文件，Class字节码文件，以及运行时JVM中，可以通过反射读取。

      RetentionPolicy.SOURCE：仅保留在源文件中。

      ###### @Target

      指定注解类型使用的程序元素范围，解释：程序会根据指定的@Target限制注解类型使用的范围，若没有指定，则没有限制。通过ElementType设置。

      ANNOTATION_TYPE:注解类型声明

      CONSTRUCTOR:构造方法声明。

      FIELD:字段类型声明。

      LOCAL_VARIABLE:局部变量声明

      METHOD:方法声明。

      PACKAGE:包声明。

      PARAMETER:参数声明。

      TYPE:类和接口或者枚举类

      4. ###### 自定义注解

         声明一个注解类型，自定义注解类型的成员变量的规则：

         其定义是以无形参的方法声明的，**注解方法中不带参数，例如：complexity()**，注解方法的返回类型：基本类型，String，enum，Annotation及前面这些类型的数组类型,注解方法中可以有默认值，例如： int complexity() default 1，默认的返回值是1。

         ```
         @Documented
         @Retention(RetentionPolicy.RUNTIME)
         @Target({ElementType.METHOD, ElementType.FIELD})
         public @interface ProgramLanguage {
             /**
              * 复杂度
              *
              * @return
              */
             int complexity() default 1;
         
             /**
              * 学习周期
              *
              * @return
              */
             String learningCycle() default "oneDay";
         
             /**
              * 枚举类
              */
             enum LgName {
                 JAVA, ANDROID, IOS
             }
         
         
             LgName plgName() default LgName.JAVA;
         }
         ```

         ###### 注解的使用

         在该类中使用上面定义的注解，为Program这个编程类的成员变量和方法添加注解。另外当注解类型没有成员变量时，以是否包含该注解作为解析的依据，当有成员变量时，如果成员变量没有设置默认值，必须在使用时指定默认的值。

         ```
         public class Program {
         
             @ProgramLanguage(plgName = LgName.ANDROID)
             private String name;
         
             @ProgramLanguage(complexity = 10)
             private int complexity;
         
             @ProgramLanguage(learningCycle = "tenDay")
             private String learningCycle;
         
             public String getName() {
                 return name;
             }
         
             public void setName(String name) {
                 this.name = name;
             }
         
             public int getComplexity() {
                 return complexity;
             }
         
             public void setComplexity(int complexity) {
                 this.complexity = complexity;
             }
         
             public String getLearningCycle() {
                 return learningCycle;
             }
         
             public void setLearningCycle(String learningCycle) {
                 this.learningCycle = learningCycle;
             }
         
             @ProgramLanguage(plgName = LgName.IOS, complexity = 2, learningCycle = "threeDay")
             public void getProgram() {
                 System.out.println("method：getProgram");
             }
         }
         ```

         ###### 注解的解析

         由于ProgramLanguage注解类型的Retention定义的是运行时，因此可以通过反射解析自定义注解，实现代码分析功能。反射包中有一个接口AnnotatedElement，

         | 返回值       | 方法                                                  | 解释                           |
         | ------------ | ----------------------------------------------------- | ------------------------------ |
         | T            | getAnnotation(Class<T> var1)                          | 返回存在的类型注解             |
         | Annotation[] | getAnnotations()                                      | 返回此元素上的注解             |
         | Annotation[] | getDeclaredAnnotations()                              | 返回直接存在此元素上的所有注解 |
         | boolean      | isAnnotationPresent(Class<? extends Annotation> var1) | 是否存在该元素的指定类型注解。 |

         定义了一个注解解析类，获取编程的Class对象，获取它的属性和方法。获取注解的详细信息。

         ```
         public class AnnotationParserUtil {
         
             public static void getPrograminfo() {
                 String clazz = "com.lijingya.annotation.Program";
                 try {
                     Class<?> c = AnnotationParserUtil.class.getClassLoader().loadClass(clazz);
                     Field[] fields = c.getDeclaredFields();
                     Method[] methods = c.getMethods();
         
                     for (Field field : fields) {
         
                         if (field.isAnnotationPresent(ProgramLanguage.class)) {
                             ProgramLanguage annotation = field.getAnnotation(ProgramLanguage.class);
                             System.out.println("名称：" + annotation.plgName() + "复杂度：" + annotation.complexity() + "学习周期：" + annotation.learningCycle());
                         }
                     }
         
                     for (Method method : methods) {
                         if (method.isAnnotationPresent(ProgramLanguage.class)) {
                             ProgramLanguage annotation = method.getAnnotation(ProgramLanguage.class);
                             System.out.println("名称：" + annotation.plgName() + "复杂度：" + annotation.complexity() + "学习周期：" + annotation.learningCycle());
                         }
                     }
         
                 } catch (ClassNotFoundException e) {
                     e.printStackTrace();
                 }
             }
         
         }
         ```

         测试：

         ```
         public class TestAnnotation {
             public static void main(String[] args) {
                 AnnotationParserUtil.getPrograminfo();
             }
         }
         ```

         日志：

         通过日志发现，我设置的四个注解，三个在成员变量上，一个在方法上的注解信息都获取到了。

         ```
         名称：ANDROID复杂度：1学习周期：oneDay
         名称：JAVA复杂度：10学习周期：oneDay
         名称：JAVA复杂度：1学习周期：tenDay
         名称：IOS复杂度：2学习周期：threeDay
         ```




