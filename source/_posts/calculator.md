---
title: 利用栈结构对中缀表达式进行求值运算
tags: [算法,中缀表达式]
author: Mingshan
categories: [算法]
date: 2020-06-14
---

最近看到邓公讲利用栈来求解中缀表达式的课程，讲的十分清楚，由于课程是用c++写的代码，我这里用Java简单实现下。

中缀表达式是一个通用的算术或逻辑公式表示方法。我们平时做的四则运算将数字与运算符拼接起来就是中缀表达式。算法思想比较清晰明了，下面我列下算法过程：

<!-- more -->

1. 首先创建两个栈，一个是操作符栈，用于保存等待参与计算的运算符，一个是数值栈，用来保存待参与计算的数值；
2. 确定各个运算符的优先级，以当前要计算的运算符，与操作符栈顶运算符为比较维度，这里用二维表格表示；
3. 依次正序遍历原字符，根据字符类型有以下计算：
   1. 如果是数字，直接入数值栈；
   2. 如果是操作符，根据当前操作符优先级与操作栈栈顶操作符优先级比较结果，需要进行一下区分：
      1. 如果当前操作符优先级高于操作栈栈顶操作符，比如当前计算符为 `*`，栈顶操作符为`+`，那么当前操作符`*`的计算需要被推迟，所以当前操作符入操作符栈；
      2. 如果优先级相等，说明当前操作符为右括号`)`，此时把操作符栈顶左括号`(`出栈；
      3. 如果当前操作符优先级低于操作栈栈顶操作符，那么此时实施计算（数值栈依次弹出两个值，与当前运算符计算），将计算结果入数值栈，同时考虑到当前运算符可能比操作栈内部其他的操作符优先级低，所以需要继续计算栈内可计算的数据；
4. 遍历完一遍后，此时栈内可能还有计算剩余，此时从数值栈直接弹出数据进行计算，直至操作符栈为空，计算完毕。

具体细节还是挺多的，比较麻烦是需要进行向前探测是否还有运算符需要计算，这里直接用递归就可以了。实现代码如下：

```Java
/**
 * 四则运算，支持优先级
 * <p>
 * 例如计算：1 + 2 * 3 - ( 1 + 2 / 3 * ( 2 + 3))
 *
 * @author mingshan
 */
public class Calculator {
  /**
   * 支持的运算符
   */
  private static final String[] SUPPORT_OPERATOR = {"+", "-", "*", "/", "(", ")", "\0"};
  /**
   * 操作符索引map
   */
  private static final Map<String, Integer> OPERATOR_INDEX_MAP = new HashMap<>();

  static {
    for (int i = 0; i < SUPPORT_OPERATOR.length; i++) {
      OPERATOR_INDEX_MAP.put(SUPPORT_OPERATOR[i], i);
    }
  }

  /**
   * 运算符优先级表 栈顶运算符索引，当前运算符索引
   */
  private static final String[][] PRIORITY_TABLE = {
      // 当前运算符 +    -    *    /    (    )    \0
      // 栈顶的运算符
      {">", ">", "<", "<", "<", ">", ">"},  // +
      {">", ">", "<", "<", "<", ">", ">"},  // -
      {">", ">", ">", ">", "<", ">", ">"},  // *
      {">", ">", ">", ">", "<", ">", ">"},  // /
      {"<", "<", "<", "<", "<", "=", ">"},  // (
      {" ", " ", " ", " ", " ", " ", ">"},  // )
      {"<", "<", "<", "<", "<", "<", "="},  // \0
  };

  public static void main(String[] args) {
    //solution("1 + 2 * 3 - ( 1 + 2 / 3 * ( 2 + 3))");
    System.out.println(solution("1 + 2"));
    System.out.println(solution("1 * 2 + 3"));
    System.out.println(solution("1 + 2 * 3"));
    System.out.println(solution("1 + 2 * 3 - 4 / 2 + 6 / 2 - 7"));

    System.out.println(solution("1 + 2 * (3 * (1 + 2))"));
  }

  public static int solution(String expression) {
    if (expression == null || expression.length() == 0) {
      return 0;
    }

    // 操作栈
    Stack<Character> operatorStack = new Stack<>();
    // 数值栈
    Stack<Integer> numberStack = new Stack<>();
    char[] chars = expression.toCharArray();

    for (int i = 0; i < chars.length; i++) {
      calculate(operatorStack, numberStack, String.valueOf(chars[i]), i == chars.length - 1);
    }

    while (!operatorStack.isEmpty()) {
      calculate(operatorStack, numberStack, String.valueOf(numberStack.pop()), true);
    }

    return numberStack.pop();
  }

  /**
   * 根据传入的 {@code item} 元素，进行计算
   *
   * @param operatorStack 操作数栈
   * @param numberStack   数值栈
   * @param item          传入的元素
   * @param isLast        当前元素是否为实际数值栈中栈顶元素
   */
  private static void calculate(Stack<Character> operatorStack, Stack<Integer> numberStack, String item,
                                boolean isLast) {
    if (" ".equals(item)) {
      return;
    }

    char[] items = item.toCharArray();

    // 字符是数字
    if (Character.isDigit(items[0])) {
      int itemInt = Integer.parseInt(item);

      // 判断是不是最后一位
      if (isLast) {
        int result = calculate(itemInt, Integer.parseInt(String.valueOf(numberStack.pop())), operatorStack.pop());
        numberStack.push(result);
      } else {
        numberStack.push(itemInt);
      }
    } else if (OPERATOR_INDEX_MAP.containsKey(item)) {
      handleOperator(operatorStack, numberStack, items[0]);
    } else {
      throw new IllegalArgumentException("不支持的运算符：" + item);
    }
  }

  /**
   * 处理操作符计算
   *
   * @param operatorStack 操作栈
   * @param numberStack   数值栈
   * @param currOperator  当前操作符
   */
  private static void handleOperator(Stack<Character> operatorStack, Stack<Integer> numberStack, char currOperator) {
    if (operatorStack.isEmpty()) {
      operatorStack.push(currOperator);
    } else {
      // 字符是操作符
      Character stackChar = operatorStack.peek();
      Integer stackIndex = OPERATOR_INDEX_MAP.get(String.valueOf(stackChar));
      Integer currIndex = OPERATOR_INDEX_MAP.get(String.valueOf(currOperator));
      String order = PRIORITY_TABLE[stackIndex][currIndex];

      switch (order) {
        // 当前操作符优先级高于操作栈栈顶操作符
        case "<":
          // 计算推迟，当前操作符入栈
          operatorStack.push(currOperator);
          break;
        case "=":
          // 优先级级相等，说明当前操作符为右括号，此时把括号去掉
          operatorStack.pop();
          break;
        case ">":
          // 当前操作符优先级低于操作栈栈顶操作符，此时实施计算
          int num1 = numberStack.pop();
          int num2 = numberStack.pop();
          numberStack.push(calculate(num1, num2, operatorStack.pop()));
          // 继续计算栈内可计算的数据
          handleOperator(operatorStack, numberStack, currOperator);
          break;
        default:
          throw new IllegalArgumentException("无效的比较结果：" + order);
      }
    }
  }

  /**
   * 计算结果
   *
   * @param num1 操作数1
   * @param num2 操作数2
   * @param item 运算符
   * @return 计算结果
   */
  private static int calculate(int num1, int num2, char item) {
    switch (String.valueOf(item)) {
      case "+":
        return num2 + num1;
      case "-":
        return num2 - num1;
      case "*":
        return num2 * num1;
      case "/":
        return num2 / num1;
      default:
        throw new UnsupportedOperationException("无效的的运算符：" + item);
    }
  }
}
```

