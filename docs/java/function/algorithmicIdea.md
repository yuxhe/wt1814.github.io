

<!-- TOC -->

- [1. 算法](#1-算法)
    - [1.1. 递归](#11-递归)
        - [1.1.1. 递归题型](#111-递归题型)
    - [1.2. 回溯算法](#12-回溯算法)
    - [1.3. 动态规划](#13-动态规划)
        - [1.3.1. 动态规划题型](#131-动态规划题型)
    - [1.4. 缓存LRU算法](#14-缓存lru算法)
        - [1.4.1. 基于哈希链表的LRU实现](#141-基于哈希链表的lru实现)

<!-- /TOC -->

# 1. 算法  
<!-- 
一文图解分治算法和思想 
https://mp.weixin.qq.com/s/kRBAwF5xAV54AqdvmMLxAg
一文学会动态规划解题技巧
https://mp.weixin.qq.com/s?__biz=MzI5MTU1MzM3MQ==&mid=2247483932&idx=1&sn=d9cd9d5a5ebf5f31e23f11c82b6465f1&scene=21#wechat_redirect
一文学会递归解题
https://mp.weixin.qq.com/s?__biz=MzI5MTU1MzM3MQ==&mid=2247483813&idx=1&sn=423c8804cd708b8892763a41cfcc8886&scene=21#wechat_redirect


拜托，别再问我贪心算法了！ 
https://mp.weixin.qq.com/s?__biz=MzI5MTU1MzM3MQ==&mid=2247483945&idx=1&sn=6f5b0d8c0ac60f40068986738d038a4d&scene=21#wechat_redirect
你说你会位运算，那你用位运算来解下八皇后问题吧 
https://mp.weixin.qq.com/s?__biz=MzI5MTU1MzM3MQ==&mid=2247483994&idx=1&sn=2baad696e0f74986195c3bcc7db3816e&scene=21#wechat_redirect


一文学会排列组合 
https://mp.weixin.qq.com/s?__biz=MzI5MTU1MzM3MQ==&mid=2247483857&idx=1&sn=c4fbb9d55a656aac55c4976c48879c45&scene=21#wechat_redirect

-->
<!-- 

2. 算法的基本方法：

1.递归法：
树的遍历，递归，循环
程序调用自身的编程技巧称为递归（recursion）。一个过程或函数在其定义或说明中有直接或间接调用自身的一种方法，它通常把一个大型复杂的问题层层转化为一个与原问题相似的规模较小的问题来求解，递归策略只需少量的程序就可描述出解题过程所需要的多次重复计算，大大地减少了程序的代码量。递归的能力在于用有限的语句来定义对象的无限集合。一般来说，递归需要有边界条件、递归前进段和递归返回段。当边界条件不满足时，递归前进；当边界条件满足时，递归返回。
注意：
(1) 递归就是在过程或函数里调用自身;
(2) 在使用递归策略时，必须有一个明确的递归结束条件，称为递归出口。
https://blog.csdn.net/qfikh/article/details/51463081

2.分治法：
分治算法的基本思想是将一个规模为N的问题分解为K个规模较小的子问题，这些子问题相互独立且与原问题性质相同。求出子问题的解，就可得到原问题的解。
https://blog.csdn.net/qfikh/article/details/51946134

分治法是把一个复杂的问题分成两个或更多的相同或相似的子问题，再把子问题分成更小的子问题……直到最后子问题可以简单的直接求解，原问题的解即子问题的解的合并。
分治法所能解决的问题一般具有以下几个特征：
(1)该问题的规模缩小到一定的程度就可以容易地解决；
(2)该问题可以分解为若干个规模较小的相同问题，即该问题具有最优子结构性质；
(3)利用该问题分解出的子问题的解可以合并为该问题的解；
(4)该问题所分解出的各个子问题是相互独立的，即子问题之间不包含公共的子子问题。
分治算法的一般步骤：
（1）分解，将要解决的问题划分成若干规模较小的同类问题；
（2）求解，当子问题划分得足够小时，用较简单的方法解决；
（3）合并，按原问题的要求，将子问题的解逐层合并构成原问题的解。
3.贪心算法：
贪心算法是一种对某些求最优解问题的更简单、更迅速的设计技术。
用贪心法设计算法的特点是一步一步地进行，常以当前情况为基础根据某个优化测度作最优选择，而不考虑各种可能的整体情况，它省去了为找最优解要穷尽所有可能而必须耗费的大量时间，它采用自顶向下,以迭代的方法做出相继的贪心选择,每做一次贪心选择就将所求问题简化为一个规模更小的子问题, 通过每一步贪心选择,可得到问题的一个最优解，虽然每一步上都要保证能获得局部最优解，但由此产生的全局解有时不一定是最优的，所以贪婪法不要回溯。
贪婪算法是一种改进了的分级处理方法，其核心是根据题意选取一种量度标准，然后将这多个输入排成这种量度标准所要求的顺序，按这种顺序一次输入一个量，如果这个输入和当前已构成在这种量度意义下的部分最佳解加在一起不能产生一个可行解，则不把此输入加到这部分解中。这种能够得到某种量度意义下最优解的分级处理方法称为贪婪算法。
对于一个给定的问题，往往可能有好几种量度标准。初看起来，这些量度标准似乎都是可取的，但实际上，用其中的大多数量度标准作贪婪处理所得到该量度意义下的最优解并不是问题的最优解，而是次优解。因此，选择能产生问题最优解的最优量度标准是使用贪婪算法的核心。
一般情况下，要选出最优量度标准并不是一件容易的事，但对某问题能选择出最优量度标准后，用贪婪算法求解则特别有效。
https://blog.csdn.net/qfikh/article/details/51959226

4.回溯法：
回溯法（探索与回溯法）是一种选优搜索法，按选优条件向前搜索，以达到目标。但当探索到某一步时，发现原先选择并不优或达不到目标，就退回一步重新选择，这种走不通就退回再走的技术为回溯法，而满足回溯条件的某个状态的点称为“回溯点”。
其基本思想是，在包含问题的所有解的解空间树中，按照深度优先搜索的策略，从根结点出发深度探索解空间树。当探索到某一结点时，要先判断该结点是否包含问题的解，如果包含，就从该结点出发继续探索下去，如果该结点不包含问题的解，则逐层向其祖先结点回溯。（其实回溯法就是对隐式图的深度优先搜索算法）。若用回溯法求问题的所有解时，要回溯到根，且根结点的所有可行的子树都要已被搜索遍才结束。而若使用回溯法求任一个解时，只要搜索到问题的一个解就可以结束。
https://blog.csdn.net/qfikh/article/details/51960331
5.分支界限法：
分枝界限法是一个用途十分广泛的算法，运用这种算法的技巧性很强，不同类型的问题解法也各不相同。
分支定界法的基本思想是对有约束条件的最优化问题的所有可行解（数目有限）空间进行搜索。该算法在具体执行时，把全部可行的解空间不断分割为越来越小的子集（称为分支），并为每个子集内的解的值计算一个下界或上界（称为定界）。在每次分支后，对凡是界限超出已知可行解值那些子集不再做进一步分支，这样，解的许多子集（即搜索树上的许多结点）就可以不予考虑了，从而缩小了搜索范围。这一过程一直进行到找出可行解为止，该可行解的值不大于任何子集的界限。因此这种算法一般可以求得最优解。
与贪心算法一样，这种方法也是用来为组合优化问题设计求解算法的，所不同的是它在问题的整个可能解空间搜索，所设计出来的算法虽其时间复杂度比贪婪算法高，但它的优点是与穷举法类似，都能保证求出问题的最佳解，而且这种方法不是盲目的穷举搜索，而是在搜索过程中通过限界，可以中途停止对某些不可能得到最优解的子空间进一步搜索（类似于人工智能中的剪枝），故它比穷举法效率更高。
https://blog.csdn.net/qfikh/article/details/51966332

递推法：
递推是序列计算机中的一种常用算法。它是按照一定的规律来计算序列中的每个项，通常是通过计算机前面的一些项来得出序列中的指定项的值。其思想是把一个复杂的庞大的计算过程转化为简单过程的多次重复，该算法利用了计算机速度快和不知疲倦的机器特点。
穷举法：
穷举法，或称为暴力破解法，其基本思路是：对于要解决的问题，列举出它的所有可能的情况，逐个判断有哪些是符合问题所要求的条件，从而得到问题的解。它也常用于对于密码的破译，即将密码进行逐个推算直到找出真正的密码为止。例如一个已知是四位并且全部由数字组成的密码，其可能共有10000种组合，因此最多尝试10000次就能找到正确的密码。理论上利用这种方法可以破解任何一种密码，问题只在于如何缩短试误时间。因此有些人运用计算机来增加效率，有些人辅以字典来缩小密码组合的范围。
动态规划法：
动态规划是一种在数学和计算机科学中使用的，用于求解包含重叠子问题的最优化问题的方法。其基本思想是，将原问题分解为相似的子问题，在求解的过程中通过子问题的解求出原问题的解。动态规划的思想是多种算法的基础，被广泛应用于计算机科学和工程领域。
动态规划程序设计是对解最优化问题的一种途径、一种方法，而不是一种特殊算法。不象前面所述的那些搜索或数值计算那样，具有一个标准的数学表达式和明确清晰的解题方法。动态规划程序设计往往是针对一种最优化问题，由于各种问题的性质不同，确定最优解的条件也互不相同，因而动态规划的设计方法对不同的问题，有各具特色的解题方法，而不存在一种万能的动态规划算法，可以解决各类最优化问题。因此读者在学习时，除了要对基本概念和方法正确理解外，必须具体问题具体分析处理，以丰富的想象力去建立模型，用创造性的技巧去求解。
https://blog.csdn.net/qfikh/article/details/51954901
迭代法：
迭代法也称辗转法，是一种不断用变量的旧值递推新值的过程，跟迭代法相对应的是直接法（或者称为一次解法），即一次性解决问题。迭代法又分为精确迭代和近似迭代。“二分法”和“牛顿迭代法”属于近似迭代法。迭代算法是用计算机解决问题的一种基本方法。它利用计算机运算速度快、适合做重复性操作的特点，让计算机对一组指令（或一定步骤）进行重复执行，在每次执行这组指令（或这些步骤）时，都从变量的原值推出它的一个新值。
-->

## 1.1. 递归  

![image](https://gitee.com/wt1814/pic-host/raw/master/images/java/function/function-5.png)  

&emsp; 递归就是指函数直接或间接的调用自己。递归函数也可以使用栈加循环来实现。  

&emsp; **递归的特点（满足递归的条件、递归解题的关键）：**  
1. 一个问题的解可以分解为几个子问题的解；
2. 这个问题与分解之后的子问题，除了数据规模不同，求解思路完全一样；
3. 存在递归终止条件，即必须有一个明确的递归结束条件，称之为递归出口。

&emsp; **如何写递归代码？**  
1. 找到如何将大问题分解为小问题的规律
2. 通过规律写出递推公式
3. 通过递归公式的临界点推敲出终止条件
4. 将递推公式和终止条件翻译成代码

### 1.1.1. 递归题型  
......

## 1.2. 回溯算法  
......
<!-- 
 揭秘回溯算法 
 https://mp.weixin.qq.com/s/yH6cLfOBjMJdbprdo3c4mg

-->

## 1.3. 动态规划  
<!-- 
 不会动态规划，如何做出这道动态规划题？ 
 https://mp.weixin.qq.com/s/2SWKifZJ3Gf1s5L2xBDJtg

  这才是真正的状态压缩动态规划好不好！！！ 
  https://mp.weixin.qq.com/s/H2V3D0DMPbT8hQW9Cq6LjQ
-->

&emsp; **动态规划与分治：**  
&emsp; 分治策略：将原问题分解为若干个规模较小但类似于原问题的子问题（Divide），递归的求解这些子问题（Conquer），然后再合并这些子问题的解来建立原问题的解。  
&emsp; 因为在求解大问题时，需要递归的求小问题，因此一般用递归的方法实现，即自顶向下。  
&emsp; 动态规划：动态规划其实和分治策略是类似的，也是将一个原问题分解为若干个规模较小的子问题，递归的求解这些子问题，然后合并子问题的解得到原问题的解。  
&emsp; <font color = "red">动态规划和分治策略的区别在于这些子问题会有重叠，一个子问题在求解后，可能会再次求解，可以将这些子问题的解存储起来，当下次再次求解这个子问题时，直接取过来用。</font>  

&emsp; **从递归到动态规划：**  
&emsp; 递归采用自顶向下的运算，比如：f(n) 是f(n-1)与f(n-2)相加，f(n-1) 是f(n-2)与f(n-3)相加。  
&emsp; 如果反过来，采取自底向上，用迭代的方式进行推导，则是动态规划。  

&emsp; **详解动态规划：**  
&emsp; 动态规划求解最值问题。动态规划实质是求最优解，不过很多题目是简化版，只要求返回最大值/最小值。最优解问题是指问题通常有多个可行解，需要从这些可行解中找到一个最优的可行解。  
&emsp; 动态规划中包含三个重要的概念：最优子结构（ f(10) =f(9)+f(8) ）、边界（ f(1) 与 f(2) ）、状态转移公式（ f(n) =f(n-1)+f(n-2) ）。  

### 1.3.1. 动态规划题型  
<!-- 
最长公共子串
https://mp.weixin.qq.com/s/0Mhe1NAZJIewbVy6A0HE4Q

-->

---


## 1.4. 缓存LRU算法  
&emsp; LRU是Least Recently Used的缩写，即最近最少使用。选择最近最久未使用的数据删除。  
&emsp; LRU Cache具备的操作：  
1. put(key, val) 方法插入新的或更新已有键值对，如果缓存已满的话，要删除那个最久没用过的键值对以腾出位置插入。  
2. get(key) 方法获取 key 对应的 val，如果 key 不存在则返回 -1。  

### 1.4.1. 基于哈希链表的LRU实现    
&emsp; **<font color = "red">LRU缓存算法的核心数据结构就是哈希链表，双向链表和哈希表的结合体。哈希表查找快，但是数据无固定顺序；链表有顺序之分，插入删除快，但是查找慢。</font>**  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/java/function/function-4.png)  

&emsp; LRU算法实现：  
&emsp; 方式一：  

```java
import java.util.LinkedHashMap;
import java.util.Map;

public class LRUCache3<K, V> {
    /**
     * 最大缓存大小
     */
    private int cacheSize;
    private LinkedHashMap<K, V> cacheMap;

    public LRUCache3(int cacheSize) {
        this.cacheSize = cacheSize;
        cacheMap = new LinkedHashMap(16, 0.75F, true) {
            @Override
            protected boolean removeEldestEntry(Map.Entry eldest) {
                if (cacheSize + 1 == cacheMap.size()) {
                    return true;
                } else {
                    return false;
                }
            }
        };
    }

    public void put(K key, V value) {
        cacheMap.put(key, value);
    }

    public V get(K key) {
        return cacheMap.get(key);
    }
}
```  

&emsp; 手写LRU算法    
&emsp; 基于LinkedHashMap实现一个简单版本的LRU算法。  

```java
class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int CACHE_SIZE;
    /**
     * @param cacheSize 缓存大小
     */
    // true表示让linkedHashMap按照访问顺序来进行排序，最近访问的放在头部，最老访问的放在尾部。
    public LRUCache(int cacheSize) {
        super((int) Math.ceil(cacheSize / 0.75) + 1, 0.75f, true);
        CACHE_SIZE = cacheSize;
    }

    @Override
    // 当map中的数据量大于指定的缓存个数的时候，就自动删除最老的数据。
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > CACHE_SIZE;
    }
}
```

```java
public class LRUCache<k, v> {
    //容量
    private int capacity;
    //当前有多少节点的统计
    private int count;
    //缓存节点
    private Map<k, node> nodeMap;
    private Node head;
    private Node tail;

    public LRUCache(int capacity) {
        if (capacity < 1) {
            throw new IllegalArgumentException(String.valueOf(capacity));
        }
        this.capacity = capacity;
        this.nodeMap = new HashMap<>();
        //初始化头节点和尾节点，利用哨兵模式减少判断头结点和尾节点为空的代码
        Node headNode = new Node(null, null);
        Node tailNode = new Node(null, null);
        headNode.next = tailNode;
        tailNode.pre = headNode;
        this.head = headNode;
        this.tail = tailNode;
    }

    public void put(k key, v value) {
        Node node = nodeMap.get(key);
        if (node == null) {
            if (count >= capacity) {
                //先移除一个节点
                removeNode();
            }
            node = new Node<>(key, value);
            //添加节点
            addNode(node);
        } else {
            //移动节点到头节点
            moveNodeToHead(node);
        }
    }

    public Node get(k key) {
        Node node = nodeMap.get(key);
        if (node != null) {
            moveNodeToHead(node);
        }
        return node;
    }

    private void removeNode() {
        Node node = tail.pre;
        //从链表里面移除
        removeFromList(node);
        nodeMap.remove(node.key);
        count--;
    }

    private void removeFromList(Node node) {
        Node pre = node.pre;
        Node next = node.next;

        pre.next = next;
        next.pre = pre;

        node.next = null;
        node.pre = null;
    }

    private void addNode(Node node) {
        //添加节点到头部
        addToHead(node);
        nodeMap.put(node.key, node);
        count++
    }

    private void addToHead(Node node) {
        Node next = head.next;
        next.pre = node;
        node.next = next;
        node.pre = head;
        head.next = node;
    }

    public void moveNodeToHead(Node node) {
        //从链表里面移除
        removeFromList(node);
        //添加节点到头部
        addToHead(node);
    }

    class Node<k, v> {
        k key;
        v value;
        Node pre;
        Node next;

        public Node(k key, v value) {
            this.key = key;
            this.value = value;
        }
    }
}
```