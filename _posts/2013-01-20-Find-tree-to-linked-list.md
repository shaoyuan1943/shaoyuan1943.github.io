---
layout: post
title: "关于二叉查找树转换成双向排序链表"
date:2013-01-20 00:00:00
categories: Win32
excerpt: 关于二叉查找树转换成双向排序链表
---

* content
{:toc}

关于树的话，就不变多说了，想必大家都清楚。关于树最具价值的应该是二叉查找树和红黑树，这里
先只学习了下二叉查找树，红黑树的学习在下一节。
二叉查找树是二叉树里面的一种，红黑树也是(我的理解就是二叉查找树的加强版)。二叉查找树是一种数据结构,它支持动态操作
包括查找、插入、删除等。
二叉查找树的基本执行时间与树的高度成正比，之所以会正比下面会给出原因的。对于一个含有n个节点的完全二叉树来说，最坏情况下时间复杂度是O(ln)，但是如果树是一个含有n各节点的链式结构的话，最坏情况下时间复杂度是O(n)。上面说到，二叉查找树的时间复杂度与树的高度成正比，探究此原因的话，请看下面关于二叉查找树的几个性质。

一、关于左右子树  
1、设x为二叉树中的一个节点，在这个节点上，如果存在左子树，则其左子树每一个节点的值只会小于或者等于x  
2、设x为二叉树中的一个节点，在这个节点上，如果存在右子树，则其右子树每一个节点的值只会大于或者等于x  
也就是说在二叉查找树种，根节点的左子树肯定是小于或者等于根节点的，根节点的右子树肯定是大于或者等于跟节点的。这样就给我们查找某一节点提供了一个很好的思路。当要search某一个节点时，可以直接与树的根节点比较，如果是小于那么该节点肯定是在左子树中，下面的事情就是一直递归左子树。同理可推理出当查找节点大于根节点的情况。

二、树的遍历：  
1、中序遍历：某一树的根在左子树与右子树之间输出  
2、前序遍历：某一树的根在左右子树之前输出  
3、后续遍历：某一树的根在左右子树之后输出  
二叉查找树的遍历实际上也就是中序遍历，中序遍历就是排序遍历。实例请看本文下面对微软面试的一道相关题的学习。

三、二叉查找树的前驱和后继  
1、某一个节点x的后继就是大于x的关键字中最小的那个节点，前驱就是小于x的关键字中最大的那个节点。当该节点有左子树时，只要遍历左子树找到左子树中最大的一个节点即可。如无左子树，则指针不停的向上移动(设x是当前节点)，```node *y = x->parent```，如此遍历直到x是y的右子树为止。因为如果是左子树，父节点只能位于该节点的后边，因此不可能是前驱节点。  
2、查找后继节点，如果该节点有右子树，则在其有子树中查找最小的那个节点即为后继节点。如果节点没有右子树。与查找前驱类似，指针不停的向上移动(设x为当前节点，即```node *y = x->parent```),直到x是y的左子树为止，因为如果是右子树，父节点只能位于该节点的前边，不可能是后继节点。

那么现在大概就知道为什么二叉查找树的时间复杂度与树的高度有关。因为二叉查找树在进行遍历的时候最坏情况下遍历到左子树或者右子树的终点，当度越大，终点也就越远，遍历所需的时间就越长。
二叉树的实现：

    typedef struct Node  
    {  
        int KeyValue;  
        struct Node* p_left_child_node;  
        struct Node* p_right_child_node;  
        struct Node* p_father_node;  
    }Node,*PNode;  

二叉查找树的构建：  

    void create_BST_tree(PNode* root,int key_array[],int node_number)  
    {   
        //逐个结点插入二叉树中    
        for(int i = 0;i < node_number; i++)    
            insert_node(root,key_array[i]);  
    }  

插入节点：  

    //有可能会改变节点指针，所以用二级指针，当然也可以指针引用   
    void insert_node(PNode* p_root,int KeyValue) 
    {  
        PNode p_node = (PNode)malloc(sizeof(Node));  
        p_node->KeyValue = KeyValue;  
        p_node->p_left_child_node = NULL;  
        p_node->p_right_child_node = NULL;  
        p_node->p_father_node = NULL;  
      
        if((*p_root) == NULL)  
        {     
            *p_root = p_node;  
            return ;  
        }  
        if((*p_root)->p_left_child_node == NULL && (*p_root)->KeyValue > KeyValue)  
        {     
            p_node->p_father_node = (*p_root);  
            (*p_root)->p_left_child_node = p_node;  
            return ;  
        }  
        if((*p_root)->p_right_child_node == NULL && (*p_root)->KeyValue < KeyValue)  
        {  
            p_node->p_father_node = (*p_root);  
            (*p_root)->p_right_child_node = p_node;  
            return ;  
        }  
        //递归方式  
        if((*p_root)->KeyValue > KeyValue)  
            insert_node(&(*p_root)->p_left_child_node,KeyValue);  
        else  
            insert_node(&(*p_root)->p_right_child_node,KeyValue);  
        return ;  
    }  

查找某一个节点：  

    PNode search(PNode p_root,int KeyValue)  
    {  
        if(p_root->KeyValue > KeyValue)  
            return search(p_root->p_left_child_node,KeyValue);  
        else if(p_root->KeyValue < KeyValue)  
            return search(p_root->p_right_child_node,KeyValue);  
        return p_root;    
    }  

寻找二叉树中最大节点： 
 
    PNode search_MAX(PNode p_root)  
    {  
        if(p_root == NULL)  
            return NULL;  
        if(p_root->p_right_child_node == NULL)  
            return p_root;  
        else  
            return search_MAX(p_root->p_right_child_node);  
    }  

寻找二叉树中最小节点：

    PNode search_MIN(PNode p_root)  
    {  
        if(p_root == NULL)  
            return NULL;  
        if(p_root->p_left_child_node == NULL)  
            return p_root;  
        else  
            return search_MIN(p_root->p_left_child_node);  
    }  

查找某一节点的前驱节点：

    PNode search_precursor_node(PNode p_node)  
    {  
        if(p_node == NULL)  
            return NULL;  
        //如果说该节点有左子树，则在左子树中一直找，直到找到最大的节点就返回  
        if(p_node->p_left_child_node)  
            return search_MAX(p_node->p_left_child_node);  
        else  
        {  
            PNode y = p_node->p_father_node;  
            //如果是没有左子树  
            if(p_node->p_father_node == NULL)  
                return NULL;  
            else  
            {  
                while(y != NULL && p_node == y->p_left_child_node)  
                {  
                    p_node = y;  
                    y = y->p_father_node;  
                }  
            }  
            return y;  
        }  
    }  

查找某一节点后继节点：  

    PNode search_succeed_node(PNode p_node)  
    {  
        if (p_node == NULL)  
        {  
            return NULL;  
        }  
        if (p_node->p_right_child_node)  
        {  
            search_MIN(p_node->p_right_child_node);  
        }  
        else  
        {  
            PNode y = p_node->p_father_node;  
            //如果是没有左子树  
            if(p_node->p_father_node == NULL)  
                return NULL;  
            else  
            {  
                while(y != NULL && p_node == y->p_right_child_node)  
                {  
                    p_node = y;  
                    y = y->p_father_node;  
                }  
            }  
            return y;     
        }  
    }  

测试：  

    int _tmain(int argc, _TCHAR* argv[])  
    {  
        PNode root=NULL;    
        int nodeArray[11]={15,6,18,3,7,17,20,2,4,13,9};    
        create_BST_tree(&root,nodeArray,11);   
        cout << search(root,20)->KeyValue << endl;  
        cout << search_MAX(root)->KeyValue << endl;  
        cout << search_MIN(root)->KeyValue << endl;  
        cout << search_precursor_node(search(root,18))->KeyValue << endl;  
        cout << search_succeed_node(search(root,20))->KeyValue << endl;  
        return 0;  
    }  

代码缺点：
每次insert的时候都是要malloc新空间，有可能会造成内存泄露。
下面看学习一道微软关于二叉查找树的算法面试题。
将下面的二叉查找树转换成排序双向链表：  
![alt text](/img/2013-01-20-1.jpg)  
我们可以大致推理出排序之后的双向链表：4=6=8=10=12=14=16
从上面可以看到，实际上这个排序是与二叉树的中序遍历一样的，也就是说对二叉查找树的中序遍历就是一个排序过程。由二叉树性质可知，左子树肯定都是比根节点小，右子树都是比根节点，那么对于转换成双向链表我们可以分成两部分，从左到右。左边是链表的前半部分，右边是链表的后半部分。图解如下：  
![alt text](/img/2013-01-20-2.jpg)  
（此图转自网络，只为说明一下转换方法)

根据上面的图解，方法就出来了。可以采用递归的方式，以左子树中最小节点的上一个节点进行递归，完成 1=2 ，然后以最小节点的上一个节点完成2=3，此操作完成后有1=2=3。

    typedef struct TNode   
    {   
        int value;   
        TNode* lchild;   
        TNode* rchild;   
    }TNode,*BTree;   
      
    //二叉树转换为双向链表   
    TNode* TreeToList(BTree tree,TNode* &lastNode)   
    {   
        TNode* head;   
        //若树为空，返回空   
        if (tree == NULL)    
        {   
            lastNode = NULL;   
            return NULL;   
        }   
        //若无左子树，则该根节点为链表的头结点   
        if (tree->lchild==NULL)   
        {   
            head = tree;   
        }   
        //若有左子树，递归调用转换函数将左子树转换为双向链表   
        //左子树转换所得链表的头结点是整个树的头结点   
        //左子树链表的尾结点与根节点相邻   
        else   
        {   
            head = TreeToList(tree->lchild,lastNode);   
            tree->lchild = lastNode;   
            tree->lchild = lastNode;   
    		if( (*lastNode) != nullptr )  
    		{  
    			lastNode->rchild = tree;  
    		}  
        }   
        //若无右子树，则该根节点为链表的尾结点   
        if (tree->rchild==NULL)   
        {   
            lastNode = tree;   
      
        }   
        //若有右子树，递归调用转换函数将左子树转换为双向链表   
        //右子树转换所得链表的尾结点是整个树的尾结点   
        //右子树链表的头结点与根节点相邻   
        else   
        {   
            tree->rchild = TreeToList(tree->rchild,lastNode);   
            tree->rchild->lchild = tree;   
        }   
        return head;   
    }

