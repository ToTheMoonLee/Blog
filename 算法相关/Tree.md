### Stack & Queue 相关算法

1. [Validate Binary Search Tree](https://leetcode.com/problems/validate-binary-search-tree/)

	> 我自己想出的办法就是进行一次inOrder遍历，使用一个List来存储遍历的结果，然后判断是否是升序；之后看了most votes的方法，发现了两个，一个是iterative的，一个是recursive的，但是还是没想明白为什么recursive的会更快
	
	1 **我自己的中序遍历版本** 
	
	```
	static List<Integer> list = new ArrayList<>();
    static boolean isValidBST(TreeNode root) {
        traverse(root);
        if (list.size() == 0) return false;
        if (list.size() == 1) return true;
        for (int i=1;i<list.size();i++) {
            if (list.get(i) <= list.get(i-1)) return false;
        }
        return true;
    }

    static void traverse(TreeNode root) {
        if (root == null) {
            return;
        }
        traverse(root.left);
        list.add(root.val);
        traverse(root.right);
    }
	```
	
	2 **直接遍历版本**
	
	```
	static boolean isValidBST2(TreeNode root) {
        if (root == null) return true;
        Stack<TreeNode> stack = new Stack<>();
        TreeNode pre = null;
        while (root!=null || !stack.isEmpty()) {
            while (root!=null) {
                stack.push(root);
                root = root.left;
            }
            root = stack.pop();
            if (pre !=null && pre.val >= root.val) return false;
            pre = root;
            root = root.right;
        }
        return true;
    }
	```
	
	3 **递归版本**
	
	```
	static boolean isValidBST3(TreeNode root) {
        return helper(root,null,null);
    }

    static boolean helper(TreeNode root, Integer minVal, Integer maxVal) {
        if (root == null) return true;
        if (minVal!=null && root.val <= minVal) return false;
        if (maxVal!=null && root.val >= maxVal) return false;
        return helper(root.left,minVal,root.val) && helper(root.right,root.val,maxVal);
    }
	```
	
2. [Lowest Common Ancestor of a Binary Search Tree](https://leetcode.com/problems/lowest-common-ancestor-of-a-binary-search-tree/)

	> BST中找最小的祖先，直接使用递归即可解决
	
	```
	static TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
			// 如果root的值比p和q都小，那么就从root的右节点继续找
        if (p.val > root.val && q.val > root.val) return lowestCommonAncestor(root.right,p,q);
        // 如果root的值比p和q都大，那么就从root的左节点继续找
        else if (p.val < root.val && q.val < root.val) return lowestCommonAncestor(root.left,p,q);
        // 否则则找到了对应的节点
        else return root;
    }
	```
	
3. [Lowest Common Ancestor of a Binary Tree](https://leetcode.com/problems/lowest-common-ancestor-of-a-binary-tree/)

	> 比起在BST中找最小祖先，在普通树中找最小祖先有点难度，最先想到的是使用recursive的方式解决，但是看了most votes版本，发现还有更简洁的写法
	
	1 **我的递归版本**
	
	```
	 static TreeNode result;

    static TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
        helper(root, p, q);
        return result;
    }

    static boolean helper(TreeNode root, TreeNode p, TreeNode q) {
        if (root == null) return false;
        // 搜索左子树是否包含p或者q
        boolean leftHasNode = helper(root.left, p, q);
        // 搜索右子树是否包含p或者q
        boolean rightHasNode = helper(root.right, p, q);
        // 判断当前的节点是否是p或者q
        boolean curNode = root.val == p.val || root.val == q.val;
        boolean isDone = (root.val == p.val && (leftHasNode || rightHasNode))
                || (root.val == q.val && (leftHasNode || rightHasNode)) ||
                (leftHasNode && rightHasNode);
        if (isDone) result = root;
        return leftHasNode || rightHasNode || curNode;
    }

	```
	
	2 **递归简洁版本** 
	
	```
	    static TreeNode lowestCommonAncestor3(TreeNode root, TreeNode p, TreeNode q) {
        if(root == null || root == p || root == q)  return root;
        TreeNode left = lowestCommonAncestor(root.left, p, q);
        TreeNode right = lowestCommonAncestor(root.right, p, q);
        if(left != null && right != null)   return root;
        return left != null ? left : right;
    }
	```
	
	3 **非递归版本**，核心思想就是记录下每个节点的父节点，然后通过搜索二叉树，找到p和q，之后再从p和q开始，找到第一个共同的祖先，即为结果
	
	```
	static TreeNode lowestCommonAncestor4(TreeNode root, TreeNode p, TreeNode q) {
        Stack<TreeNode> stack = new Stack<>();
        HashMap<TreeNode,TreeNode> parents = new HashMap<>();
        parents.put(root,null);
        stack.push(root);

        while (!parents.containsKey(p) || !parents.containsKey(q)) {
            TreeNode node = stack.pop();
            if (node.left!=null) {
                stack.push(node.left);
                parents.put(node.left,node);
            }
            if (node.right!=null) {
                stack.push(node.right);
                parents.put(node.right,node);
            }
        }
        HashSet<TreeNode> set = new HashSet<>();
        while (p!=null) {
            set.add(p);
            p = parents.get(p);
        }
        while (!set.contains(q)) {
            q = parents.get(q);
        }
        return q;
    }
	```




