---
layout: archive
title:  "链表的相关算法"
date: 2017-06-13 10:00
categories: algorithm
tag: linklist algorithm
---

### 关于一些链表的常见的面试题备忘
#### 先预定义下链表节点的结构
```
struct LinkNode {
	int val;
	struct LinkNode *next;
	LinkNode() : val(0), next(NULL) {
	}
}
```
#### 单向链表反转
*给的头节点，让单向链表反转,并返回反转后的头节点*
```
LinkNode *InversedLink(const LinkNode *head) {
	assert(head != NULL);
	LinkNode *pnext, *pcurrent, *phead;
	phead = NULL;
	pcurrent = head;
	pnext = head->next;
	while (pcurrent) {
		pcurrent->next = phead;
		phead = pcurrent;
		pcurrent = pnext;
		pnext = pnext ? pnext->next : pnext;
	}
	return phead;
}
```

#### 寻找单向链表的中间节点或倒数第K个节点
```
LinkNode *FindMidleNode(const LinkNode *head) {
	assert(head != NULL);
	LinkNode *pfront = head;
	LinkNode *plast = head;
	while (pfront && pfront->next) {
		// 快指针每次走两个单位，慢指针每次走一步
		plast = plast->next;
		pfront = pfront->next->next;
	}
	return plast;

}
LinkNode *FindKNode(const LinkNode *head, int k) {
	assert(head != NULL);
	LinkNode *pfront = head;
	LinkNode *plast = head;
	while (k--) { //快指针先走K步
		pfront = pfront->next;
	}
	while (pfront) {//两个指针同步走
		pfront = pfront->next;
		plast = plast->next;
	}
	return plast;
}
```
#### 判断单项链表是否有环，并找出环的入口节点

*思路：
	假设该链表前单链部分的长度为a, 环的长度为b, 现在使用一个二倍速的指针p2和一个
单倍速的指针p1开始都指向头节点，遍历循环下去，如果存在环，那么这两者一定会在环内相遇。
	当存在环的时候，在p1最先到达环的时候，走过的距离是a，p2一定在环中，走过的距离是2a,
也是a+nb+k(n为任意的自然数，k为自然数并且0<=k<b), 故 a = nb+k。此时p1和p2继续走，
因为相对的距离为k，则要经过b-k个单位才能相遇，相遇的时候，p1走过 a+(b-k), 我们结合下
之前的结论：a=nb+k, 可知，再经过a个单位就是能走到头节点(a+(b-k) + nb+k = a+(n+1)b)，
故代码如下：*
```
LinkNode *FindLoopNode(const LinkNode* head) {
	assert(head != NULL);
	LinkNode *pfront = head;
	LinkNode *plast = head;
	while (pfront && pfront->next) {
		if (pfront == plast) {
			break;
		}
		pfront = pfront->next->next;
		plast = plast->next;
	}
	if (!pfront) {
		return NULL;//无环
	}
	plast = head;
	while (plast == pfront) {
		plast = plast->next;
		pfront = pfront->next;
	}
	return plast;
}
```
#### 判断两个单项链表是否有交叉点，并找交叉的第一个节点
*思路：
	两个链表是否有交叉节点，如果有交叉节点，因为是交叉节点，所以一定是Y形交叉
而不是X形交叉。Y形交叉的话，这两个链表的尾节点一点相等。
	如何找到第一个交叉点呢，一种是在距离第一个节点相距距离相同的位置开始遍历，
这样的话，这确定距离相同点的位置的时候得计算两个链表的长度。另一种方法是利用，
a+b+ K = b+a+ K(a为一链表不相交部分，b为另一链表不相交部分，K为公共重合的长度)
实现的代码如下：*
```
LinkNode *FindFirstPub(const LinkNode *head1, const LinkNode *head2) {
	assert(head1 != NULL && head2 != NULL);
	LinkNode *p1 = head1, *p2 = head2;
	while (p1 != p2) {
		p1 = p1 ? p1->next : head2;
		p2 = p2 ? p2->next : head1;
	}
	return p1;
}
```
 注：同样的思路可以拓展到寻找树中两个节点的最近公共节点
