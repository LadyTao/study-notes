#### 概念
枚举是一种特殊的数据类型，它既是一种类，又有一些特殊的约束。我们可以把它看作是一个特殊的类，这个类是一个继承自Enum类的被final修饰的类，也就是它不会有其他子类出现。其定义的成员都是被public static final修饰的。它可以有自己的构造器，但是权限不能是public权限的。

#### 常见方法

在 enum 中，提供了一些基本方法：
* values()：返回 enum 实例的数组，而且该数组中的元素严格保持在 enum 中声明时的顺序。
* name()：返回实例名。
* ordinal()：返回实例声明时的次序，从 0 开始。getDeclaringClass()：返回实例所属的 enum 类型。
* equals() ：判断是否为同一个对象。

#### 枚举的应用
1. switch转发太极
```java

public class StateMachineDemo {
    public enum Signal {
        GREEN, YELLOW, RED
    }

    public static String getTrafficInstruct(Signal signal) {
        String instruct = "信号灯故障";
        switch (signal) {
            case RED:
                instruct = "红灯停";
                break;
            case YELLOW:
                instruct = "黄灯请注意";
                break;
            case GREEN:
                instruct = "绿灯行";
                break;
            default:
                break;
        }
        return instruct;
    }

    public static void main(String[] args) {
        System.out.println(getTrafficInstruct(Signal.RED));
    }
}
// Output:// 红灯停
```
2. 错误码
```java

public class ErrorCodeEnumDemo {
    enum ErrorCodeEn {
        OK(0, "成功"),
        ERROR_A(100, "错误A"),
        ERROR_B(200, "错误B");

        ErrorCodeEn(int number, String msg) {
            this.code = number;
            this.msg = msg;
        }

        private int code;
        private String msg;

        public int getCode() {
            return code;
        }

        public String getMsg() {
            return msg;
        }

        @Override
        public String toString() {
            return "ErrorCodeEn{" + "code=" + code + ", msg='" + msg + '\'' + '}';
        }

        public static String toStringAll() {
            StringBuilder sb = new StringBuilder();
            sb.append("ErrorCodeEn All Elements: [");
            for (ErrorCodeEn code : ErrorCodeEn.values()) {
                sb.append(code.getCode()).append(", ");
            }
            sb.append("]");
            return sb.toString();
        }
    }

    public static void main(String[] args) {
        System.out.println(ErrorCodeEn.toStringAll());
        for (ErrorCodeEn s : ErrorCodeEn.values()) {
            System.out.println(s);
        }
    }
}
// Output:
// ErrorCodeEn All Elements: [0, 100, 200, ]
// ErrorCodeEn{code=0, msg='成功'}
// ErrorCodeEn{code=100, msg='错误A'}
// ErrorCodeEn{code=200, msg='错误B'}
```
3. 单例模式（推荐）
```java
enum SingleEn {
    INSTANCE;
    private Dogs dd;

    SingleEn() {
        dd = new Dogs("aa", 11);
    }

    Dogs getInstance() {
        return dd;
    }
}

class Dogs {
    String name;
    int age;

    Dogs() {
    }

    Dogs(String n, int a) {
        this.name = n;
        this.age = a;
    }
}

@Test
public void test2() {
	Dogs i1 = SingleEn.INSTANCE.getInstance();
	Dogs i3 = SingleEn.INSTANCE.getInstance();
	Dogs i2 = SingleEn.INSTANCE.getInstance();
	System.out.println(i1);
	System.out.println(i2);
	System.out.println(i3);
}
```
#### 枚举工具类

java的枚举类提供两种高效的enum工具类：EnumSet和EnumMap。

* EnumSet

numSet 是枚举类型的高性能 Set 实现。它要求放入它的枚举常量必须属于同一枚举类型。主要接口：
* noneOf - 创建一个具有指定元素类型的空EnumSet
* allOf - 创建一个指定元素类型并包含所有枚举值的EnumSet
* range - 创建一个包括枚举值中指定范围元素的 EnumSet 
* complementOf - 初始集合包括指定集合的补集of - 创建一个包括参数中所有元素的 EnumSet
* copyOf - 创建一个包含参数容器中的所有元素的 EnumSet
```java
@Test
public void test3() {
	EnumSet<Day> errSet1 = EnumSet.allOf(Day.class);
	for (Day e : errSet1) {
		System.out.println(e.name() + " : " + e.ordinal());
	}
	System.out.println("---------------");
	EnumSet<Day> errSet2 = EnumSet.noneOf(Day.class);
	for (Day e : errSet2) {
		System.out.println(e.name() + " : " + e.ordinal());
	}
	System.out.println("---------------");
	EnumSet<Day> errSet3 = EnumSet.range(Day.MONDAY, Day.WEDNESDAY);
	for (Day e : errSet3) {
		System.out.println(e.name() + " : " + e.ordinal());
	}
	System.out.println("---------------");
	EnumSet<Day> errSet4 = EnumSet.complementOf(errSet3);
	for (Day e : errSet4) {
		System.out.println(e.name() + " : " + e.ordinal());
	}
	System.out.println("---------------");
	EnumSet<Day> errSet5 = EnumSet.copyOf(errSet4);
	for (Day e : errSet5) {
		System.out.println(e.name() + " : " + e.ordinal());
	}

}
```

* EnumMap：EnumMap 是专门为枚举类型量身定做的 Map 实现。虽然使用其它的 Map 实现（如 HashMap）也能完成枚举类型实例到值得映射，但是使用 EnumMap 会更加高效。

主要接口：
* size - 返回键值对数
* containsValue - 是否存在指定的 value
* containsKey - 是否存在指定的 key
* get - 根据指定 key 获取 value
* put - 取出指定的键值对
* remove - 删除指定 key
* putAll - 批量取出键值对
* clear - 清除数据
* keySet - 获取 key 集合
* values - 返回所有
发现其方法与常规的map基本一致。


