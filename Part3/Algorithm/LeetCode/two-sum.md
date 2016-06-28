#two-sum
---

Question

```
Given an array of integers, find two numbers such that they add up to a specific target number.
The function twoSum should return indices of the two numbers such that they add up to the target, where index1 must be less than index2. Please note that your returned answers (both index1 and index2) are not zero-based.
You may assume that each input would have exactly one solution.
Input: numbers={2, 7, 11, 15}, target=9
Output: index1=1, index2=2
```


题目大意：

```
给定一个整数数组，找到2个数字，这样他们就可以添加到一个特定的目标号。功能twosum应该返回两个数字，他们总计达目标数，其中index1必须小于index2。请注意，你的答案返回（包括指数和指数）不为零的基础。你可以假设每个输入都有一个解决方案。
输入数字numbers= { 2，7，11，15 }，目标= 9输出：index1 = 1，index2= 2
```

解题思路：

```
可以申请额外空间来存储目标数减去从头遍历的数，记为key，如果hashMap中存在该key，就可以返回两个索引了。
```


代码；

```
import java.util.HashMap;

public class Solution {

	public int[] twoSum(int[] numbers, int target) {
		HashMap<Integer, Integer> map = new HashMap<>();
		for(int i = 0; i < numbers.length; i++){
			if(map.get(numbers[i]) != null){
				int[] result = {map.get(numbers[i]) + 1, i+1};
				return result;
			}else {
				map.put(target - numbers[i], i);
			}
		}
		int[] result = {};
		return result;
	}
}
```