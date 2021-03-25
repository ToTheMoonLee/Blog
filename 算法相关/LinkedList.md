### Linked List 相关算法

1. [Reverse Linked List](https://leetcode.com/problems/reverse-linked-list/)

	> 单链表中很经典的一道题，问题详见上面链接
	
	首先可以使用重新建立一个链表，遍历给定的链表，然后使用头插法插入到新的链表中，但是这种方法显然不是这道题想要的，这道题要使用原地反转的方法。有以下两种解法：
	
	1 **非递归版本**：一次遍历，在遍历的过程中记录下前一个节点，跟下一个节点，防止断链
	
	```
	public ListNode reverseList(ListNode head) {
	   if (head == null || head.next == null) {
	       return head;
	   }
	      
	   ListNode cur = head;
	   ListNode pre = null;
	   ListNode next = null;
	   while (cur != null) {
	       next = cur.next;
	       cur.next = pre;
	       pre = cur;
	       cur = next;
	   }
	   return pre;
	}
	```
	
	2 **递归版本**：递归对于我来说实在是不好想，下面是leetcode top votes 版本
	
	```
	public ListNode reverseList(ListNode head) {
	    /* recursive solution */
	    return reverseListInt(head, null);
	}
	
	private ListNode reverseListInt(ListNode head, ListNode newHead) {
	    if (head == null)
	        return newHead;
	    ListNode next = head.next;
	    head.next = newHead;
	    return reverseListInt(next, head);
	}
	```
	
2. [Swap Nodes in Pairs](https://leetcode.com/problems/swap-nodes-in-pairs/)

	> 问题乍一看不是很难，但是真正实现的时候却并不是想象中的那么简单，详细问题见链接
	
	1 **我自己的解法**：创建一个空节点的头指针（单链表算法一般都会用到这个技巧，会减少空判断代码），然后具体的想法如下图
	
	```
	null->1->2->3->4
	pre     cur   
	
	  _______
	 |       |
	 |       v
	null  1->2->3->4
	 |    |     |
	pre  cur   next
	 
	  _______ 
	 |       |
	 |       v
	null  1->2  3->4
	 |    |     |
	pre  cur   next
	      ^     |
	      |_____|
	      
	       ______
	  ____|___  |    
	 |    |  |  | 
	 |    |  v  v 
	null  1  2  3->4
	 |    |     |    
	pre  cur   next
	      ^     |
	      |_____|

	        
	``` 
	
	```
	public ListNode swapPairs(ListNode head) {
        ListNode pre = new ListNode();
        pre.next = head;
        ListNode next = null;
        ListNode cur = head;
        head = pre;
        while (cur != null && cur.next != null) {
            pre.next = cur.next;
            next = cur.next.next;
            cur.next.next = cur;
            cur.next = next;
            pre = cur;
            cur = next;
        }
        return head.next;
   }
	```
	
	2 **递归版本**，同样，递归版本还是没想出来，同样看的是top votes版本
	
	```
	public ListNode swapPairs(ListNode head) {
        if (head == null || head.next == null) {
            return head;
        }
        ListNode n = head.next;
        head.next = swapPairs(head.next.next);
        n.next = head;
        return n;
    }
	```

3. [Linked List Cycle](https://leetcode.com/problems/linked-list-cycle/)
	
	> 该问题最开始看的时候没啥思路，后来想到使用HashMap来保存遍历过的节点，然后判断之后遍历到的节点是否在HashMap中，如果存在则返回true，如果没有，则返回false
	
	1 **HashMap版本** ：
	
	```
	public boolean hasCycle(ListNode head) {
       if (head == null) {
            return false;
        }

        HashSet<ListNode> hashSet = new HashSet<>();
        while (head != null) {
            if (hashSet.contains(head)) {
                return true;
            }
            hashSet.add(head);
            head = head.next;
        }
        return false;
    }
	```
	
	之后看了大神的思路，确实是惊叹。使用快慢指针来查看是否有cycle，快指针每次往前走两步，慢指针每次走一步，如果有环，则他们最终就会相遇，如果没环，则循环结束。
	
	2 **快慢指针版本**
	
	```
	public boolean hasCycle(ListNode head) {
        if (head == null) {
            return false;
        }

        ListNode walker = head;
        ListNode runner = head;
        while (runner != null && runner.next != null) {
            runner = runner.next.next;
            walker = walker.next;
            if (runner == walker) {
                return true;
            }
        }
        return false;
    }
	```
	
4. [Linked List Cycle II](https://leetcode.com/problems/linked-list-cycle-ii/)
	
	> 同样是在单链表里找环，不同的是，这次要找到是从哪个位置开始的环，比前一个难很多，先来个最直观的HashMap版本
	
	1 **HashMap版本**
	
	```
	public ListNode detectCycle(ListNode head) {
        int i = 0;
        HashMap<ListNode,Integer> hashMap = new HashMap<>();
        while (head != null) {
            if (hashMap.get(head) != null) {
                return head;
            }    
            hashMap.put(head,i++);
            head = head.next;
        }
        return null;
    }
	```
	
	2 **空间复杂度O(1)版本** 这个同样是top votes的大佬版本，刚开始没理解是为啥，然后看了几篇解释
	[floyds-cycle-detection-algorithm-determining-the-starting-point-of-cycle](https://cs.stackexchange.com/questions/10360/floyds-cycle-detection-algorithm-determining-the-starting-point-of-cycle)和
	[Detecting start of a loop in singly Linked List](https://web.archive.org/web/20160401024212/http://learningarsenal.info:80/index.php/2015/08/24/detecting-start-of-a-loop-in-singly-linked-list/)
	同样是使用的快慢指针，但是之后的逻辑稍有不同，代码如下：
	
	```
	public ListNode detectCycle(ListNode head) {
        ListNode slow = head;
        ListNode fast = head;

        while (fast!=null && fast.next!=null){
            fast = fast.next.next;
            slow = slow.next;

            if (fast == slow){
                ListNode slow2 = head; 
                while (slow2 != slow){
                    slow = slow.next;
                    slow2 = slow2.next;
                }
                return slow;
            }
        }
        return null;
	 }
	```
	
	简化的解释如下图，假设环是这个样子的话，慢指针跟快指针将在节点7的位置相遇。抽象一下，考虑更普遍的一种情况，假设从头节点到开始circle的节点距离为A（对应上图就是2），而慢指针跟快指针发生相遇的节点到开始circle的节点距离为B（对应上图就是4），从两指针相遇的地方再到circle开始的节点距离为C（对应上图为从7到3，即距离为2），那么慢指针走过的距离为 A+B ，快指针走过的距离为 A+B+C+B，又快指针走过的距离一定是慢指针的两倍，即2A+2B = A+B+C+B 则 A = C，也就是说从开始节点到circle开始的节点和两指针相遇节点到circle开始的节点距离相同，即上面的算法成立。

	```
	       1-->2-->3-->4-->5     
	               ^       |		
	               |       v
	               8<--7<--6
	                   |
	                  slow
	                 faster
		
	```
5. [Reverse Nodes in k-Group](https://leetcode.com/problems/reverse-nodes-in-k-group/)

	> hard级别的题，	最初想要用递归的方式解决，尝试了一下之后还是决定用最基本的遍历方式了，思路就是：记录一下要每段要反转的k个单链表元素的开始节点的前一个节点和结束节点的后一个节点，将k个单链表反转之后，在交换头尾的指针，如下图：
	
	```
	直接拿的leetcode中解法的注释，每三个节点反转一次，加了一个值为0的头节点，将begin跟end之间的单链表反转，然后将1的next指向end，0的next指向3，以此往复即可
     * 0->1->2->3->4->5->6
     * |           |   
     * begin       end
     * after call begin = reverse(begin, end)
     * 
     * 0->3->2->1->4->5->6
     *          |  |
     *      begin end
	```
	
	1 **遍历版本**代码如下：
	
	```
	static ListNode reverseKGroup(ListNode head, int k) {
        int i = 0;
        ListNode preLink = new ListNode();
        preLink.next = head;
        ListNode returnHead = preLink;
        ListNode nextLink = null;
        while (head != null) {
            i++;
            head = head.next;
            if (i == k) {
                nextLink = head;
                ListNode pre = preLink;
                ListNode cur = preLink.next;
                ListNode next = null;
                while (cur.next != nextLink) {
                    next = cur.next;
                    cur.next = pre;
                    pre = cur;
                    cur = next;
                }
                cur.next = pre;
                preLink.next.next = nextLink;
                next = preLink.next;
                preLink.next = cur;
                preLink = next;
                i = 0;
            }
        }
        return returnHead.next;
    }
	```
	
	2 **递归版本**，同样是top votes版, 详细见注释
	
	```
	static ListNode reverseKGroup2(ListNode head, int k) {
        ListNode curr = head;
        int count = 0;
        while (curr != null && count != k) { // find the k+1 node
            curr = curr.next;
            count++;
        }
        if (count == k) { // if k+1 node is found
            curr = reverseKGroup(curr, k); // reverse list with k+1 node as head
            // head - head-pointer to direct part,
            // curr - head-pointer to reversed part;
            while (count-- > 0) { // reverse current k-group:
                ListNode tmp = head.next; // tmp - next head in direct part
                head.next = curr; // preappending "direct" head to the reversed list
                curr = head; // move head of reversed part to a new node
                head = tmp; // move "direct" head to the next node in direct part
            }
            head = curr;
        }
        return head;
    }
	```
	
	







