#设计一个有getMin功能的栈
---

实现一个特殊的栈，在实现栈的基本功能的基础上，在实现返回栈中最小元素的操作。

要求：

1. pop、push、getMin操作的时间复杂度都是O(1)
2. 设计的栈类型可以使用现成的栈结构

解题：


```
package chapter01_stackandqueue;

import java.util.Stack;

/**
 * 
 * 实现一个特殊的栈，在实现栈的基本功能的基础上，在实现返回栈中最小元素的操作。 要求： 1. pop、push、getMin操作的时间复杂度都是O(1)
 * 2. 设计的栈类型可以使用现成的栈结构
 * 
 * @author dream
 *
 */
public class Problem01_GetMinStack {

	public static class MyStack1 {

		/**
		 * 两个栈，其中stacMin负责将最小值放在栈顶，stackData通过获取stackMin的peek()函数来获取到栈中的最小值
		 */
		private Stack<Integer> stackData;
		private Stack<Integer> stackMin;

		/**
		 * 在构造函数里面初始化两个栈
		 */
		public MyStack1() {
			stackData = new Stack<Integer>();
			stackMin = new Stack<Integer>();
		}

		/**
		 * 该函数是stackData弹出栈顶数据，如果弹出的数据恰好等于stackMin的数据，那么stackMin也弹出
		 * @return
		 */
		public Integer pop() {
			Integer num = (Integer) stackData.pop();
			if (num == getmin()) {
				return (Integer) stackMin.pop();
			}
			return null;
		}

		/**
		 * 该函数是先判断stackMin是否为空，如果为空，就push新的数据，如果这个数小于stackMin中的栈顶元素，那么stackMin需要push新的数，不管怎么样
		 * stackData都需要push新的数据
		 * @param value
		 */
		public void push(Integer value) {
			if (stackMin.isEmpty()) {
				stackMin.push(value);
			}

			else if (value < getmin()) {
				stackMin.push(value);
			}
			stackData.push(value);
		}

		/**
		 * 该函数是当stackMin为空的话第一次也得push到stackMin的栈中，返回stackMin的栈顶元素
		 * @return
		 */
		public Integer getmin() {
			if (stackMin == null) {
				throw new RuntimeException("stackMin is empty");
			}
			return (Integer) stackMin.peek();

		}
	}

	public static void main(String[] args) throws Exception {
		/**
		 * 要注意要将MyStack1声明成静态的，静态内部类不持有外部类的引用
		 */
		MyStack1 stack1 = new MyStack1();
		stack1.push(3);
		System.out.println(stack1.getmin());
		stack1.push(4);
		System.out.println(stack1.getmin());
		stack1.push(1);
		System.out.println(stack1.getmin());
		System.out.println(stack1.pop());
		System.out.println(stack1.getmin());

		System.out.println("=============");
	}

}

```