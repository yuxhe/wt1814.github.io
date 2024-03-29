

<!-- TOC -->

- [1. Trie](#1-trie)
    - [1.1. Trie介绍](#11-trie介绍)
    - [1.2. Trie实现](#12-trie实现)
    - [1.3. Trie应用](#13-trie应用)
    - [1.4. 使用Trie的建议](#14-使用trie的建议)

<!-- /TOC -->

# 1. Trie 
<!-- 
「算法与数据结构」Trie树之美 
https://juejin.im/post/6888451657504391181

Trie 树（前缀树）
https://mp.weixin.qq.com/s?__biz=MzI5MTU1MzM3MQ==&mid=2247484257&idx=1&sn=ef0104a011707ad1ab35bb4b8991ad79&scene=21#wechat_redirect

-->
## 1.1. Trie介绍
<!-- 
&emsp; 又称单词查找树，Trie树，是一种树形结构，是一种哈希树的变种。<font color = "lime">典型应用是用于统计，排序和保存大量的字符串（但不仅限于字符串），所以经常被搜索引擎系统用于文本词频统计。</font>它的优点是：利用字符串的公共前缀来减少查询时间，最大限度地减少无谓的字符串比较，查询效率比哈希树高。  

Trie 树，又称前缀树，字典树，或单词查找树，是一种树形结构，也是哈希表的变种，它是一种专门处理字段串匹配的数据结构，用来解决在一组字符串集合中快速查找某个字符串的问题，主要被搜索引擎用来做文本词频的统计。
-->
&emsp; 在计算机科学中，trie，又称前缀树或字典树，是一种有序树，用于保存关联数组，其中的键通常是字符串。与二叉查找树不同，键不是直接保存在节点中，而是由节点在树中的位置决定。一个节点的所有子孙都有相同的前缀，也就是这个节点对应的字符串，而根节点对应空字符串。一般情况下，不是所有的节点都有对应的值，只有叶子节点和部分内部节点所对应的键才有相关的值。  

    Trie有很多变种，如后缀树，Radix Tree/Trie，PATRICIA tree，以及bitwise版本的crit-bit tree。当然很多名字的意义其实有交叉。   

&emsp; Trie结构图(红色节点表示一个单词的结尾)：  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/java/function/function-13.png)  
<!-- 
&emsp; 1、可以看到， trie 树每一层的节点数是 26^i 级别的。所以为了节省空间，还可以 用动态链表，或者用数组来模拟动态。而空间的花费，不会超过单词数× 单词长度。  
&emsp; 2、实现： 对每个结点开一个字母集大小的数组， 每个结点挂一个链表， 使用左儿子右 兄弟表示法记录这棵树；  
&emsp; 3、对于中文的字典树，每个节点的子节点用一个哈希表存储，这样就不用浪费太 大的空 间， 而且查询速度上可以保留哈希的复杂度 O(1)。  
-->
&emsp; Trie 的核心思想是空间换时间， 利用字符串的公共前缀来降低查询时间的开销以达到提高效率的目的。它有 3 个基本性质：   
1. 根节点不包含字符， 除根节点外每一个节点都只包含一个字符。  
2. 从根节点到某一节点， 路径上经过的字符连接起来， 为该节点对应的字符串。  
3. 每个节点的所有子节点包含的字符都不相同。  

&emsp; 优点：  
&emsp; 可以最大限度地减少无谓的字符串比较，故可以用于词频统计和大量字符串排序。  
&emsp; 跟哈希表比较：  
1. 最坏情况时间复杂度比hash表好  
2. 没有冲突，除非一个key对应多个值（除key外的其他信息）  
3. 自带排序功能（类似Radix Sort），中序遍历trie可以得到排序。  

&emsp; 缺点：
1. 虽然不同单词共享前缀，但其实trie是一个以空间换时间的算法。其每一个字符都可能包含至多字符集大小数目的指针（不包含卫星数据）。  
&emsp;每个结点的子树的根节点的组织方式有几种。1>如果默认包含所有字符集，则查找速度快但浪费空间（特别是靠近树底部叶子）。2>如果用链接法(如左儿子右兄弟)，则节省空间但查找需顺序（部分）遍历链表。3>alphabet reduction: 减少字符宽度以减少字母集个数。4>对字符集使用bitmap，再配合链接法。  
2. 如果数据存储在外部存储器等较慢位置，Trie会较hash速度慢（hash访问O(1)次外存，Trie访问O(树高)）。  
3. 长的浮点数等会让链变得很长。可用bitwise trie改进。  

## 1.2. Trie实现 
&emsp; Trie 树是一颗多叉树，那么多叉树该怎么表示呢，假设字符串都是由 26 个小写字母组成，则显然 Trie 树应该是一颗 26 叉树，每个节点包含 26 个子节点，如下  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/java/function/function-39.png)  
&emsp; 上图可以看出，26 个子节点我们可以用大小为 26 的数组表示，所以 Trie 树表示如下  

```java
/**
 * 26 个字母
 */
static final int ALPHABET_SIZE = 26;

/**
 * Trie 树的节点表示
 */
static class TriedNode {
    /**
     * 根节点专用，存储 "/"
     */
    public char val;

    /**
     * 以此结点字符为终止字符的字符串的个数
     */
    public int frequency;

    /**
     * 节点指向的子节点
     */
    TriedNode[] children = new TriedNode[ALPHABET_SIZE];

    public TriedNode(char val) {
        this.val = val;
    }
}

/**
 * Trie 树
 */
static class TrieTree {
    private TriedNode root = new TriedNode('/'); // 根节点
}
```
&emsp; Trie 树的定义有了，现在来看下 Trie 树的两个主要操作  

1. 根据一组字符串构造 Trie 树
2. 在 Trie 树中查找字符串是否存在

&emsp; 先来看如何根据一组字符串构造 Trie 树，首先如何根据一个单词来构造 Trie 树呢，假设以单词 「and」 为例来看下 Trie 树的表现形式  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/java/function/function-40.png)  
&emsp; 注：图中的数字表示数组的元素位置  
&emsp; 可以看到构建 Trie 树的主要步骤如下  

1. 构建根节点，此时根节点存有一个元素大小为 26 的数组
2. 遍历字符串「and」
3. 遍历第一个字符 a 时，将上述数组的第一个元素赋值为一个 TriedNode 实例（假设其名为 A）
4. 当遍历第二个字符 n 时，将 A 结点 TriedNode 数组下标为 n-a = 13 (a 的 ascii 为 97，n 的 ascii 码为 110) 的元素赋值为一个 TriedNode 实例（假设其名为 N）
5. 同理，当遍历第三个字符 d 时，将 N 结点 TriedNode 数组的第 4 个元素（下标为 3）赋值为一个 TriedNode 实例（假设其名为 D），同时也将其结点的 frequency 加一，代表以此字符为终止字符的字符串多了一个。  

&emsp; 由以上分析不难写出根据字符串构建 Trie 树的代码，如下  

```java
/**
 * Trie 树
 */
static class TrieTree {
    private TriedNode root = new TriedNode('/'); // 根节点

    /**
     * 以 String 为条件构建 Trie 树
     * @param s
     */
    public void insertString(String s) {
        TriedNode p = root;
        for (int i = 0; i < s.length(); i++){
            char c = s.charAt(i);
            int index  = c-'a';
            if (p.children[index] == null) {
                p.children[index] = new TriedNode(c);
            }
            p = p.children[index];
            //Process char
        }
        p.frequency++;
    }
}
```
&emsp; Trie 树构造好了，再在 Trie 树中查找某字符串是否存在就简单很多了，遍历字符串，查看每个字符在相应层级的数组位置的元素是否为空即可，如果是，说明不存在，如果不是，则继续遍历字符查找，直到遍历完成，代码如下  

```java
/**
 * 查找字符串是否在原字符串集合中
 * @param s
 * @return boolean
 */
public boolean findStr(String s) {
    TriedNode p = root;
    for (int i = 0; i < s.length(); i++){
        // 当前被遍历的字符
        char c = s.charAt(i);
        int index  = c-'a';
        if (p.children[index] == null) {
            // 如果字符对应位置的数组元素为空，说明肯定不存在此字符，终止之后的字符遍历
            return false;
        }
        // 如果存在，则继续往后遍历字符串
        p = p.children[index];
    }
    return true;
}
```
&emsp; 由于在节点中也用 frequency 保存了单词数，所以如果在 Trie 树中最终发现字符串存在，也可以随便查找出此字符串的个数。  

## 1.3. Trie应用  
<!-- 
https://www.cnblogs.com/justinh/p/7716421.html
https://mp.weixin.qq.com/s?__biz=MzI5MTU1MzM3MQ==&mid=2247484257&idx=1&sn=ef0104a011707ad1ab35bb4b8991ad79&scene=21#wechat_redirect
-->
1. 串的快速检索  
&emsp; 给出N个单词组成的熟词表，以及一篇全用小写英文书写的文章，请按最早出现的顺序写出所有不在熟词表中的生词。  
&emsp; 在这道题中，可以用数组枚举，用哈希，用字典树，先把熟词建一棵树，然后读入文章进行比较，这种方法效率是比较高的。  
2. “串”排序  
&emsp; 给定N个互不相同的仅由一个单词构成的英文名，让将它们按字典序从小到大输出。  
&emsp; 用字典树进行排序，采用数组的方式创建字典树，这棵树的每个结点的所有儿子很显然地按照其字母大小排序。对这棵树进行先序遍历即可。  
3. 最长公共前缀  
&emsp; 对所有串建立字典树，对于两个串的最长公共前缀的长度即它们所在的结点的公共祖先个数，于是，问题就转化为当时公共祖先问题。  

&emsp; 代码实现  

```java
packagecom.suning.search.test.tree.trie;
 
public class Trie
{
    private int SIZE=26;
    private TrieNode root;//字典树的根
 
    Trie() //初始化字典树
    {
        root=new TrieNode();
    }
 
    private class TrieNode //字典树节点
    {
        private int num;//有多少单词通过这个节点,即由根至该节点组成的字符串模式出现的次数
        private TrieNode[]  son;//所有的儿子节点
        private boolean isEnd;//是不是最后一个节点
        private char val;//节点的值
        private boolean haveSon;
 
        TrieNode()
        {
            num=1;
            son=new TrieNode[SIZE];
            isEnd=false;
            haveSon=false;
        }
    }
 
    //建立字典树
    public void insert(String str) //在字典树中插入一个单词
    {
        if(str==null||str.length()==0)
        {
            return;
        }
        TrieNode node=root;
        char[]letters=str.toCharArray();
        for(int i=0,len=str.length(); i<len; i++)
        {
            int pos=letters[i]-'a';
            if(node.son[pos]==null)
            {
                node.haveSon = true;
                node.son[pos]=newTrieNode();
                node.son[pos].val=letters[i];
            }
            else
            {
                node.son[pos].num++;
            }
            node=node.son[pos];
        }
        node.isEnd=true;
    }
 
    //计算单词前缀的数量
    public int countPrefix(Stringprefix)
    {
        if(prefix==null||prefix.length()==0)
        {
            return-1;
        }
        TrieNode node=root;
        char[]letters=prefix.toCharArray();
        for(inti=0,len=prefix.length(); i<len; i++)
        {
            int pos=letters[i]-'a';
            if(node.son[pos]==null)
            {
                return 0;
            }
            else
            {
                node=node.son[pos];
            }
        }
        return node.num;
    }
    //打印指定前缀的单词
    public String hasPrefix(String prefix)
    {
        if (prefix == null || prefix.length() == 0)
        {
            return null;
        }
        TrieNode node = root;
        char[] letters = prefix.toCharArray();
        for (int i = 0, len = prefix.length(); i < len; i++)
        {
            int pos = letters[i] - 'a';
            if (node.son[pos] == null)
            {
                return null;
            }
            else
            {
                node = node.son[pos];
            }
        }
        preTraverse(node, prefix);
        return null;
    }
    // 遍历经过此节点的单词.
    public void preTraverse(TrieNode node, String prefix)
    {
        if (node.haveSon)
        {
                     for (TrieNode child : node.son)
            {
                if (child!=null)
                {
                    preTraverse(child, prefix+child.val);
                }
            }
            return;
        }
        System.out.println(prefix);
    }
 
 
    //在字典树中查找一个完全匹配的单词.
    public boolean has(Stringstr)
    {
        if(str==null||str.length()==0)
        {
            return false;
        }
        TrieNode node=root;
        char[]letters=str.toCharArray();
        for(inti=0,len=str.length(); i<len; i++)
        {
            intpos=letters[i]-'a';
            if(node.son[pos]!=null)
            {
                node=node.son[pos];
            }
            else
            {
                return false;
            }
        }
        return node.isEnd;
    }
 
    //前序遍历字典树.
    public void preTraverse(TrieNodenode)
    {
        if(node!=null)
        {
            System.out.print(node.val+"-");
                        for(TrieNodechild:node.son)
            {
                preTraverse(child);
            }
        }
    }
 
    public TrieNode getRoot()
    {
        return this.root;
    }
 
    public static void main(String[]args)
    {
        Trietree=newTrie();
        String[]strs= {"banana","band","bee","absolute","acm",};
        String[]prefix= {"ba","b","band","abc",};
                for(Stringstr:strs)
        {
            tree.insert(str);
        }
        System.out.println(tree.has("abc"));
        tree.preTraverse(tree.getRoot());
        System.out.println();
                //tree.printAllWords();
                for(Stringpre:prefix)
        {
            int num=tree.countPrefix(pre);
            System.out.println(pre+""+num);
        }
    }
}
```

## 1.4. 使用Trie的建议  
&emsp; 从前面的介绍中可以看到使用 Trie 树确实在能在快速查找字符串与词频统计上发挥重要作用，但天下没有免费的午餐，如果字符集比较大的话，用 Trie 树可能会造成空间的浪费，以上文中构建的 Trie 树为例  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/java/function/function-41.png)  
&emsp; 每个结点维护一个 26 个元素大小的数组，共有 4 个数组，也就是分配了 26 x 4 = 104  个元素的空间，但实际上只有三个元素空间（a，n，d）被分配了，浪费了 101 个空间，空间利率率很低，所以一般更适用于字符串前缀重复比较多的情况，当然也可以考虑对 Trie 树进行如下缩点优化，能节省一些空间  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/java/function/function-42.png)  
&emsp; 当然这么优化后也增加了代码的编码难度，所以要视情况而定。  
&emsp; 另外如果用 Trie 树的话，一般需要我们自己编码，对工程师的编码能力要求较高，所以是否用 Trie 树一般建议如下：  

1. 如果是字符串的精确匹配查找，一般建议使用散列表或红黑树来解决，毕竟很多语言的类库都有现成的，不需要自己实现，拿来即用
2. 如果需要进行前缀匹配查找，则用 Trie 树更合适一些
