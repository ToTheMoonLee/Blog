### Recursion & Division 相关算法

1. [Majority Element](https://leetcode.com/problems/majority-element/)

	> 最初的想法就是使用循环n次，然后乘以x值n次，如果n是负值乘以1/x n次，但是用例跑不过去，会Time Limit Exceeded，最终还是看了大佬们的版本，使用递归策略
	
	1 **递归版本** 
	
	```
	static double myPow(double x, int n) {
        if (n == 0) return 1;
        if(n == Integer.MIN_VALUE){
            x = x * x;
            n = n/2;
        }
        if (n < 0) {
            x = 1/x;
            n = -n;
        }
        return n%2 == 0 ? myPow(x*x,n/2) : x * myPow(x*x,n/2);
    }
	```
		
2. [Majority Element](https://leetcode.com/problems/majority-element/)

	> 找众数，这个思路比较多，首先就是brute force，双重for循环，依次统计每个数出现的次数，当然时间复杂度为O(n^2)；第二种是使用一个HashMap来统计，时间复杂度为O(n);第三种是直接进行排序，返回中间的那个数即可，时间复杂度为O(nlogn)；这是我想到的三种，剩余两种是Solution里的，一个是Divide and Conquer，采用分治的策略，时间复杂度为O(nlogn)；最后一种叫做Boyer-Moore Voting Algorithm，这个感觉真的是很厉害，完全想不到，下面是每种的代码
	
	1 **HashMap版本**
	
	```
	  private Map<Integer, Integer> countNums(int[] nums) {
        Map<Integer, Integer> counts = new HashMap<Integer, Integer>();
        for (int num : nums) {
            if (!counts.containsKey(num)) {
                counts.put(num, 1);
            }
            else {
                counts.put(num, counts.get(num)+1);
            }
        }
        return counts;
    }

    public int majorityElement(int[] nums) {
        Map<Integer, Integer> counts = countNums(nums);

        Map.Entry<Integer, Integer> majorityEntry = null;
        for (Map.Entry<Integer, Integer> entry : counts.entrySet()) {
            if (majorityEntry == null || entry.getValue() > majorityEntry.getValue()) {
                majorityEntry = entry;
            }
        }

        return majorityEntry.getKey();
    }
	```
	
	2 **sort版本**
	
	```
	public int majorityElement(int[] nums) {
        Arrays.sort(nums);
        return nums[nums.length/2];
    }
	```
	
	3 **divide and conquer版本**
	
	```
	 private int countInRange(int[] nums, int num, int lo, int hi) {
        int count = 0;
        for (int i = lo; i <= hi; i++) {
            if (nums[i] == num) {
                count++;
            }
        }
        return count;
    }

    private int majorityElementRec(int[] nums, int lo, int hi) {
        // base case; the only element in an array of size 1 is the majority
        // element.
        if (lo == hi) {
            return nums[lo];
        }

        // recurse on left and right halves of this slice.
        int mid = (hi-lo)/2 + lo;
        int left = majorityElementRec(nums, lo, mid);
        int right = majorityElementRec(nums, mid+1, hi);

        // if the two halves agree on the majority element, return it.
        if (left == right) {
            return left;
        }

        // otherwise, count each element and return the "winner".
        int leftCount = countInRange(nums, left, lo, hi);
        int rightCount = countInRange(nums, right, lo, hi);

        return leftCount > rightCount ? left : right;
    }

    public int majorityElement(int[] nums) {
        return majorityElementRec(nums, 0, nums.length-1);
    }
	```
	
	4 **Boyer-Moore Voting Algorithm版本**
	
	```
	    public int majorityElement(int[] nums) {
        int count = 0;
        Integer candidate = null;

        for (int num : nums) {
            if (count == 0) {
                candidate = num;
            }
            count += (num == candidate) ? 1 : -1;
        }

        return candidate;
    }
	```
	




