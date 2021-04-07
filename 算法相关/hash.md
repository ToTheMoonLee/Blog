### Hash 相关算法

1. [Valid Anagram](https://leetcode.com/problems/valid-anagram/)

	> 找出两个字符串是否是异位的字符串
	
	首先肯定可以想到遍历第一个字符串使用HashMap，使用每个字符为key，出现的次数为value来存储，没出现一次，就加一；然后遍历第二个字符串进行，对每出现的一个字符进行减一的操作；最终遍历HashMap中，每个字符对应的出现的次数是否为0，即可解决。当然更简单一点，直接使用一个数组即可，数组下标作为key，用来存储出现的次数，代码如下
	
	1 **array版本**：
	
	```
	public boolean isAnagram(String s, String t) {
        int[] arr = new int[26];
        for (int i=0;i<s.length();i++) {
            arr[s.charAt(i) - 'a']++;
        }
        for (int i=0;i<t.length();i++) {
            arr[t.charAt(i) - 'a']--;
            if (arr[t.charAt(i) - 'a'] < 0) return false;
        }
        for (int i=0;i<arr.length;i++) {
            if (arr[i]!=0) return false;
        }
        return true;
    }
	```

2. [Two Sum](https://leetcode.com/problems/two-sum/)

	> 作为LeetCode的第一题，最开始的时候，我使用的是Brute Force算法，直接两次循环，来找出答案，然后看了Discuss中的答案之后，才想到使用HashMap
	
	```
	public int[] twoSum(int[] nums, int target) {
        HashMap<Integer,Integer> hash = new HashMap<>();
        int[] result = new int[2];
        for (int i=0;i<nums.length;i++) {
            if (hash.get(nums[i]) == null) {
                hash.put(target - nums[i],i);
            } else {
                result[0] = hash.get(nums[i]);
                result[1] = i;
                return result;
            }
        }
        return null;
    }
	```	

3. [3Sum](https://leetcode.com/problems/3sum/)

	> 这个题最开始想到的是Brute-Force解法，使用三重for来层层遍历，但是明显时间复杂度为O(n^3)，这个题其实使用两边向中间夹的方式，先对array进行排序，然后再解决，具体解法见代码
	
	```
	static List<List<Integer>> threeSum(int[] nums) {
        List<List<Integer>> result = new ArrayList<>();
        Arrays.sort(nums);
        for (int i=0;i<nums.length;i++) {
            if (i==0 || (i>0 && nums[i]!=nums[i-1])) {
                int low = i+1, high = nums.length-1, sum = 0 - nums[i];
                while (low < high) {
                    if (nums[low] + nums[high] == sum) {
                        result.add(Arrays.asList(nums[i],nums[low],nums[high]));
                        while (low < high && nums[low] == nums[low+1]) low++;
                        while (low < high && nums[high] == nums[high-1]) high--;
                        low++;
                        high--;
                    } else if (nums[low] + nums[high] < sum) low++;
                    else high--;
                }
            }
        }
        return result;
    }
	```





