============
leetcode
============

回溯套路
============

1. leetcode 39 组合总和 https://leetcode-cn.com/problems/combination-sum/

.. code::

    class Solution:
        def combinationSum(self, candidates: List[int], target: int) -> List[List[int]]:
            self.track = []
            self.results = []
            self.backtrack(candidates, target)
            return self.results

        def backtrack(self, candidates, target):
            if target == 0:
                self.results.append(self.track[:])
                return
            if target < 0:
                return
            for index, num in enumerate(candidates):
                self.track.append(num)
                self.backtrack(candidates[index:], target-num)
                self.track.pop()

2. leetcode 22 括号生成 https://leetcode-cn.com/problems/generate-parentheses/

.. code::

    class Solution:
        def generateParenthesis(self, n: int) -> List[str]:
            self.results = []
            self.backtrack(n, [], 0, 0)
            return self.results

        def backtrack(self, n, track, left, right):
            if left > n or right > n or left < right:
                return
            if left == right == n:
                self.results.append(''.join(track))
                return
            track.append('(')
            self.backtrack(n, track, left+1, right)
            track.pop()
            track.append(')')
            self.backtrack(n, track, left, right+1)
            track.pop()

计算器
=========

leetcode 224 基本计算器 https://leetcode-cn.com/problems/basic-calculator/

.. code::

    class Solution:
        def helper(self, s: list) -> int:
            sign = '+'
            num = 0
            stack = []
            while s:
                c = s.pop()
                if c.isdigit():
                    num = num * 10 + int(c)
                if c == '(':
                    # 递归计算num
                    num = self.helper(s)
                if (c.strip() and not c.isdigit()) or not s:
                    if sign == '+':
                        stack.append(num)
                    elif sign == '-':
                        stack.append(-num)
                    elif sign == '*':
                        stack[-1] *= num
                    elif sign == '/':
                        stack[-1] = int(stack[-1] / num)
                    num = 0
                    sign = c
                if c == ')':
                    break
            return sum(stack)

        def calculate(self, s: str) -> int:
            return self.helper(list(s)[::-1])

两数之和
============

1. leetcode 167 两数之和-输入有序数组 https://leetcode-cn.com/problems/two-sum-ii-input-array-is-sorted/

.. code::

    class Solution:
        def twoSum(self, numbers: List[int], target: int) -> List[int]:
            left, right = 0, len(numbers) - 1
            while left < right:
                left_val = numbers[left]
                right_val = numbers[right]
                if left_val + right_val == target:
                    return [left+1, right+1]
                elif left_val + right_val < target:
                    left += 1
                elif left_val + right_val > target:
                    right -= 1


1. leetcode 15 三数之和 https://leetcode-cn.com/problems/3sum/

.. code::

    class Solution:

        def twoSum(self, nums: List[int], start: int, target: int):
            left, right = start, len(nums) - 1
            res = []
            while left < right:
                left_val, right_val = nums[left], nums[right]
                tmp = left_val + right_val
                if tmp == target:
                    res.append([left_val, right_val])
                    while left < right and nums[left] == left_val:
                        left += 1
                    while left < right and nums[right] == right_val:
                        right -= 1
                elif tmp < target:
                    while left < right and nums[left] == left_val:
                        left += 1
                elif tmp > target:
                    while left < right and nums[right] == right_val:
                        right -= 1
            return res

        def threeSum(self, nums: List[int]) -> List[List[int]]:
            nums.sort()  # 先排序
            res = []
            for index, num in enumerate(nums):
                if index > 0 and num == nums[index-1]:
                    continue
                target = 0 - num
                two_sum = self.twoSum(nums, index+1, target)
                for lst in two_sum:
                    res.append([num, lst[0], lst[1]])
            return res

滑动窗口
===========

1. leetcode 76 最小覆盖子串 https://leetcode-cn.com/problems/minimum-window-substring/

.. code::

    class Solution:
        def minWindow(self, s: str, t: str) -> str:
            from collections import defaultdict

            need, window = defaultdict(int), defaultdict(int)
            for c in t:
                need[c] += 1
            # 初始化滑动窗口两端，初始不包含任何元素
            left, right = 0, 0
            # 窗口中满足need条件的字符数，如果valid==len(need)，说明窗口满足条件
            # 例如，t='ABC', window滑动到ADOBE时，valid=2，而need为3，滑动窗口还不满足条件
            valid = 0

            INT_MAX = len(s) + 1
            start = 0
            min_length = INT_MAX
            while right < len(s):
                c = s[right]
                right += 1
                if c in need:
                    window[c] += 1
                    # 窗口中c的个数==need中c的个数时，说明这个字符c已经全部出现了，valid+1
                    if window[c] == need[c]:
                        valid += 1
                # 目前右边界的滑动窗口已经满足need要求了，接下来缩小左边界
                while valid == len(need):
                    if (right - left) < min_length:
                        start = left
                        min_length = right - left
                    d = s[left]
                    left += 1
                    # d是要被移出窗口的字符
                    # 当d在need中时，要更新valid
                    if d in need:
                        if window[d] == need[d]:
                            valid -= 1
                        # window只会存need中的字符，所以需要判断，不管是否等于need中d的数量，最终都会-1
                        window[d] -= 1
            return '' if min_length == INT_MAX else s[start: start+min_length]

2. leetcode 567 字符串的排列 https://leetcode-cn.com/problems/permutation-in-string/

.. code::

    class Solution:
        def checkInclusion(self, s1: str, s2: str) -> bool:
            from collections import defaultdict
            need, window = defaultdict(int), defaultdict(int)
            for c in s1:
                need[c] += 1
            left, right = 0, 0
            valid = 0
            while right < len(s2):
                c = s2[right]
                right += 1
                if c in need:
                    window[c] += 1
                    if window[c] == need[c]:
                        valid += 1
                while right - left >= len(s1):
                    if valid == len(need):
                        return True
                    d = s2[left]
                    left += 1
                    if d in need:
                        if window[d] == need[d]:
                            valid -= 1
                        window[d] -= 1
            return False

3. leetcode 3 无重复字符的最长子串 https://leetcode-cn.com/problems/longest-substring-without-repeating-characters/

.. code::

    class Solution:
        def lengthOfLongestSubstring(self, s: str) -> int:
            window = set()
            left, right = 0, 0
            max_length = 0
            while right < len(s):
                c = s[right]
                right += 1
                while c in window:
                    d = s[left]
                    left += 1
                    window.remove(d)
                window.add(c)
                max_length = max(max_length, right-left)
            return max_length

LRU缓存
===========

1. leetcode 146 LRU缓存机制 https://leetcode-cn.com/problems/lru-cache/

.. code::

    class Node:
        def __init__(self, key: int, val: int) -> None:
            """双向链表的节点
            存key更方便，HashLinkedList的方法可以少传一个参数key

            Args:
                key (int): key
                val (int): value
            """
            self.key = key
            self.val = val
            self.prev = None
            self.next = None


    class HashLinkedList:
        def __init__(self) -> None:
            self.datas = {}
            # 使用虚节点，实际操作在head和tail之间
            # 下面的头节点和尾节点分别是head之后的第一个节点和tail之前的第一个节点
            self.head = Node(None, None)
            self.tail = Node(None, None)
            self.head.next = self.tail
            self.tail.prev = self.head
            self.size = 0

        def add(self, node: Node):
            """添加节点到头部
            头部是最常使用的节点

            Args:
                node (Node): node
            """
            node.prev = self.head
            node.next = self.head.next
            self.head.next.prev = node
            # 注意顺序
            self.head.next = node
            self.datas[node.key] = node
            self.size += 1

        def remove_node(self, node: Node):
            node.prev.next = node.next
            node.next.prev = node.prev
            self.datas.pop(node.key)
            self.size -= 1

        def remove_end(self):
            """头部是最常使用的节点，那么尾部就是最少使用的节点
            删除尾部节点就删除了最少使用的节点
            """
            if self.tail.prev == self.head:
                return
            node = self.tail.prev
            self.remove_node(node)


    class LRUCache:

        def __init__(self, capacity: int):
            self.capacity = capacity
            self.hash_linked_list = HashLinkedList()

        def get(self, key: int) -> int:
            if key in self.hash_linked_list.datas:
                # 提一下使用次数
                self.make_recently(key)
                return self.hash_linked_list.datas.get(key).val
            return -1

        def put(self, key: int, value: int) -> None:
            if key in self.hash_linked_list.datas:
                # 变更顺序还要提升使用次数，其实就是删除之前的，重新加
                self.hash_linked_list.remove_node(self.hash_linked_list.datas[key])
                self.hash_linked_list.add(Node(key, value))
            else:
                if self.capacity == self.hash_linked_list.size:
                    self.hash_linked_list.remove_end()
                self.hash_linked_list.add(Node(key, value))

        def make_recently(self, key: int):
            """提升次数
            其实就是删除再添加

            Args:
                key (int): [description]
            """
            node = self.hash_linked_list.datas.get(key)
            self.hash_linked_list.remove_node(node)
            self.hash_linked_list.add(node)


    # Your LRUCache object will be instantiated and called as such:
    # obj = LRUCache(capacity)
    # param_1 = obj.get(key)
    # obj.put(key,value)


二叉树
=========

1. leetcode 102 二叉树的层序遍历 https://leetcode-cn.com/problems/binary-tree-level-order-traversal/

.. code::

    class Solution:
        def levelOrder(self, root: TreeNode) -> List[List[int]]:
            if root is None:
                return []
            lst = []
            queue = [root]
            while queue:
                tmp = []
                for _ in range(len(queue)):
                    node = queue.pop(0)
                    tmp.append(node.val)
                    if node.left:
                        queue.append(node.left)
                    if node.right:
                        queue.append(node.right)
                lst.append(tmp)
            return lst

2. leetcode 105 从前序和中序遍历构造二叉树 https://leetcode-cn.com/problems/construct-binary-tree-from-preorder-and-inorder-traversal/

.. code::

    class Solution:
        def buildTree(self, preorder: List[int], inorder: List[int]) -> TreeNode:
            if not preorder:
                return None

            def function(inorder):
                node = TreeNode(val=preorder.pop(0))
                node_index = inorder.index(node.val)
                left, right = inorder[:node_index], inorder[node_index+1:]
                node.left = function(left) if left else None
                node.right = function(right) if right else None
                return node
            return function(inorder)

3. leetcode 98 验证二叉搜索树 https://leetcode-cn.com/problems/validate-binary-search-tree/

.. code::

    class Solution:

        def is_valid_BST(self, root: TreeNode, min_node: TreeNode, max_node: TreeNode):
            if root is None:
                return True
            if min_node is not None and root.val <= min_node.val:
                return False
            if max_node is not None and root.val >= max_node.val:
                return False
            return self.is_valid_BST(root.left, min_node, root) and self.is_valid_BST(root.right, root, max_node)

        def isValidBST(self, root: TreeNode) -> bool:
            return self.is_valid_BST(root, None, None)



