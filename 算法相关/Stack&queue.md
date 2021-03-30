### Stack & Queue 相关算法

1. [Valid Parentheses](https://leetcode.com/problems/valid-parentheses/)

	> 栈中很经典的一道题，问题详见上面链接
	
	最直观的解法就是将所碰到的左括号入栈，如果碰到右括号而且栈顶是对应的左括号，则将栈顶元素出栈；如果碰到右括号时，栈顶元素不是对应的左括号，则直接返回false（情况一）；同时如果所有操作都完成之后，栈不为空也返回false（情况二）：
	情况一： "( ]"
	情况二： "( ) ("
	
	1 **直观版本**：
	
	```
	public boolean isValid(String s) {
        Stack<Character> stack = new Stack<Character>();
        for(int i = 0; i<s.length(); i++) {
            if(s.charAt(i) == '(' || s.charAt(i) == '[' || s.charAt(i) == '{')
                stack.push(s.charAt(i));
            else if(s.charAt(i) == ')' && !stack.empty() && stack.peek() == '(')
                stack.pop();
            else if(s.charAt(i) == ']' && !stack.empty() && stack.peek() == '[')
                stack.pop();
            else if(s.charAt(i) == '}' && !stack.empty() && stack.peek() == '{')
                stack.pop();
            else
                return false;
        }
        return stack.empty();
    }
	```
	
	2 **简洁版本**：下面是leetcode top votes 版本，想必逻辑大家肯定能读懂，这种思维方式还是值得学习的
	
	```
	public boolean isValid(String s) {
       Stack<Character> stack = new Stack<>();
        for (Character c : s.toCharArray()) {
            if (c == '(') {
                stack.push(')');
            } else if (c == '[') {
                stack.push(']');
            } else if (c == '{') {
                stack.push('}');
            } else if (stack.isEmpty() || stack.pop() != c) {
                return false;
            }
        }
        return stack.isEmpty();
    }
	```
	
2. [Implement Queue using Stacks](https://leetcode.com/problems/implement-queue-using-stacks/)

	> 用栈实现队列，这里肯定得需要两个栈，其中可以选择push操作的时间复杂度为O(n),或者选择peek操作的时间复杂度为O(n)
	
	**push O(n)版本**,另外的版本，可以自行查看leetcode，代码如下：
	
	```
	class MyQueue {
    
	    Stack<Integer> s1;
	    Stack<Integer> s2;
	
	    /** Initialize your data structure here. */
	    public MyQueue() {
	        s1 = new Stack<>();
	        s2 = new Stack<>();
	    }
	    
	    /** Push element x to the back of queue. */
	    /** 在每次push的时候，先将s1中的所有元素放到s2中，然后将新元素push到s1中，之后再将s2中的元素放入到s1中
	    public void push(int x) {
	        int size = s1.size();
	        for (int i=0;i<size;i++) {
	            s2.push(s1.pop()); 
	        } 
	        s1.push(x);
	        size = s2.size();
	        for (int i=0;i<size;i++) {
	            s1.push(s2.pop());
	        }
	    }
	    
	    /** Removes the element from in front of queue and returns that element. */
	    public int pop() {   
	        return s1.pop();
	    }
	    
	    /** Get the front element. */
	    public int peek() {
	        return s1.peek();
	    }
	    
	    /** Returns whether the queue is empty. */
	    public boolean empty() {
	        return s1.isEmpty();
	    }
	}
	```
	
3. [Kth Largest Element in a Stream](https://leetcode.com/problems/kth-largest-element-in-a-stream/)

	> 前k大的元素，这种问题使用优先级队列即可解决，优先级队列是使用heap这个数据结构实现的，相关概念可以查看[堆和堆排序](https://time.geekbang.org/column/article/69913)
	
	代码就相对简单了，初始化一个大小为k的优先级队列，遍历目标的stream，并与堆中最小的元素做比较，进行对应的插入操作即可
	
	```	
	class KthLargest {
	    private int k;
		private PriorityQueue<Integer> priorityQueue;
		public KthLargest(int k, int[] nums) {
			this.k = k;
			priorityQueue = new PriorityQueue<>(k);
			for(int num : nums) {
				add(num);
			}
		}
	
		public int add(int val) {
			
			if(priorityQueue.size() == k) {
				
				val = Math.max(val, priorityQueue.poll());
			}
			priorityQueue.add(val);
			
			return priorityQueue.peek();
		}
	}
	```	

4. [Sliding Window Maximum](https://leetcode.com/problems/sliding-window-maximum/)

	> hard难度的滑动窗口的问题，想出了两种解法，一个是优先级队列，一个是Brute Force，然而因为时间复杂度太高，没有通过，最终看了各位大佬们的解法
	
	1 **Dequeue双端队列解法**  算法思想：使用dequeue用来保存遍历到的下标i，从i=0开始遍历nums，每次遍历先判断dequeue的最右端元素是否小于nums[i]，如果小于，则将dequeue右端开始所有小与nums[i]的元素都从右端移除；之后判断dequeue最右端元素保存的下标是否小与i-k，即查看是否最左端元素还在窗口范围内，如果不在则从左端移除；最后在i>k-1时，将最左端元素加入到result数组中。
	
	```
		static int[] maxSlidingWindow(int[] nums, int k) {
		   int[] result = new int[nums.length-k + 1];
		   LinkedList<Integer> window = new LinkedList<>();
		   for (int i=0;i<nums.length;i++) {
		       while (!window.isEmpty() && window.peekFirst() <= i-k) {
		           window.pollFirst();
		       }
		       while (!window.isEmpty() && nums[window.peekLast()] < nums[i]) {
		           window.pollLast();
		       }
		       window.addLast(i);
		       if (i>=k-1) {
		           result[i-k+1] = nums[window.peekFirst()];
		       }
		   }
		   return result;
		}
	```
	
	2 **DP解法** 算法思想还是自己看leetcode的解释吧，看了好多遍才勉强看懂[leetcode dp版本](https://leetcode.com/problems/sliding-window-maximum/discuss/65881/O(n)-solution-in-Java-with-two-simple-pass-in-the-array)
	
	```
	public static int[] slidingWindowMax(final int[] in, final int w) {
	    final int[] max_left = new int[in.length];
	    final int[] max_right = new int[in.length];
	
	    max_left[0] = in[0];
	    max_right[in.length - 1] = in[in.length - 1];
	
	    for (int i = 1; i < in.length; i++) {
	        max_left[i] = (i % w == 0) ? in[i] : Math.max(max_left[i - 1], in[i]);
	
	        final int j = in.length - i - 1;
	        max_right[j] = (j % w == 0) ? in[j] : Math.max(max_right[j + 1], in[j]);
	    }
	
	    final int[] sliding_max = new int[in.length - w + 1];
	    for (int i = 0, j = 0; i + w <= in.length; i++) {
	        sliding_max[j++] = Math.max(max_right[i], max_left[i + w - 1]);
	    }
	
	    return sliding_max;
	}
	```









