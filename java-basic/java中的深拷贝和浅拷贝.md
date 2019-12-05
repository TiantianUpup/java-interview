# java中的深拷贝和浅拷贝

### 定义
**深拷贝：**  
- 基本数据类型 => 拷贝数值  
- 引用类型 => 拷贝对象引用

**浅拷贝：**  
- 基本数据类型 => 拷贝数值  
- 引用类型 => 拷贝引用对象的所有数值

### 如何拷贝对象
如果对象需要提供拷贝的功能，必须实现Cloneable接口重写clone方法  

**示例：**
- Student对象：Student.java
  ```
   public class Student implements Cloneable {
      private int id;
      private String name;
      private Score score;

      //get and set

      @Override
      protected Object clone() throws CloneNotSupportedException {
          return super.clone();
      }
  }
  ```
  clone方法中直接调用了父类的clone，父类的clone方法是一个native方法
- Score对象：
  ```
  public class Score {
      private int math;
    
      private int chinese;

      //get and set
  }
  ```
- 测试类：CloneTest.java
  ```
  public class CloneTest {
      public static void main(String[] args) throws CloneNotSupportedException {
          Student student = new Student();
          student.setId(1);
          student.setName("h2t");
          Score score = new Score();
          score.setChinese(100);
          score.setMath(100);
          student.setScore(score);

          Student cloneStudent = (Student)student.clone();
          cloneStudent.getScore().setChinese(59);
          System.out.println(student);
      }
  }
  ```
- 执行结果：
  ```
   Student{id=1, name='h2t', score=Score{math=100, chinese=59}}
  ```
  被拷贝对象的值也被修改了。默认父类的clone方法是浅拷贝

- 修改clone方法
  - 修改Student的clone方法 
    ```
    @Override
    protected Object clone() throws CloneNotSupportedException {
        Student student = (Student) super.clone();
        Score score = student.getScore();
        student.setScore((Score)score.clone());
        return student;
    }
    ```
  - Score添加clone方法
    ```
    @Override
    protected Object clone() throws CloneNotSupportedException {
        return super.clone();
    }
    ```

- 重新测试结果
  ```
  Student{id=1, name='h2t', score=Score{math=100, chinese=100}}
  ```
  实现了深拷贝，拷贝对象的修改没有影响被拷贝对象，两者是独立的

### 扩展
重写clone方法实现深拷贝如果在对象引用对象特别多的时候代码量将特别大，可以使用对象的序列化与反序列化实现对象的深拷贝

**示例：**
```
ByteArrayOutputStream bos = new ByteArrayOutputStream();
ObjectOutputStream oos = new ObjectOutputStream(bos);
oos.writeObject(student);
oos.flush();

ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(bos.toByteArray()));
Student cloneStudent = (Student) ois.readObject();
cloneStudent.getScore().setChinese(59);
System.out.println(student);
```
**执行结果：**
```
Student{id=1, name='h2t', score=Score{math=100, chinese=100}}
```
成功实现了深拷贝

最后附：[示例代码](https://github.com/TiantianUpup/java-learning/tree/master/src/main/java/com/h2t/study/clone)，欢迎**fork**与**star**

