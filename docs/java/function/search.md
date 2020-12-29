
<!-- TOC -->

- [查找算法](#查找算法)
    - [1. 二分查找](#1-二分查找)
        - [1.1. 二分查找模版](#11-二分查找模版)
        - [1.2. 二分查找实现](#12-二分查找实现)
        - [1.3. 二分查找题型](#13-二分查找题型)

<!-- /TOC -->

<!-- 
三款经典的查找算法
https://mp.weixin.qq.com/s/3RvYUaAL8xAQQvT88WAJ7g

在N个乱序数字中查找第k大的数字
https://blog.csdn.net/u010412301/article/details/67704530

寻找第K大数的方法
https://blog.csdn.net/csl13/article/details/6056522
-->
<!-- 
你真的会写二分检索吗？ 
https://mp.weixin.qq.com/s/zlNDNwsTV5GvPzlGjfB0sQ
一网打尽！二分查找解题模版与题型全面解析 
https://mp.weixin.qq.com/s?__biz=MzIwNTc4NTEwOQ==&mid=2247487194&idx=1&sn=bd094c2953137469a51bc700319286a2&chksm=972adfa0a05d56b625e9a40bf6d7cb5baf32b34e9bcca861ba3e3ba5b7e1a39670126e62bfe8&mpshare=1&scene=1&srcid=&sharer_sharetime=1568041137749&sharer_shareid=b256218ead787d58e0b58614a973d00d&key=dee829c9aae7a0c07b1a6df0769ad2ff9adcf98eff1f993fbfcb5665f85eb64c3f3ad155311c62a3935f7e826bfa9a18edb4368cb626bc3386a5188fca1f22bcaf8c26344b6956a89644391713eb0616&ascene=1&uin=MTE1MTYxNzY2MQ%3D%3D&devicetype=Windows+10&version=62060844&lang=zh_CN&pass_ticket=5e25q4PxFBEBE22tP%2FFCoORgWWOx%2FBQjku90ubbS9N5KcxzzEydoolU%2BArDDM%2FKQ
-->

# 查找算法  
&emsp; 顺序查找、二分查找、分块查找、哈希表查找  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/java/function/function-32.png)  

## 1. 二分查找  
&emsp; <font color = "lime">要在有序数组中进行查找，最好的解决方法是使用二分查找算法。</font>  

### 1.1. 二分查找模版   

```java
int start = 0, end = nums.length - 1;
while (start + 1 < end) {
    int mid = start + (end - start) / 2;
    
    if (...) {
        start = mid;
    } else {
        end = mid;
    }
}
```
&emsp; 上面模版当中的start + (end - start) / 2，这里不直接写成 (end + start) / 2的目的是防止计算越界，举个例子，假如end = 2^31 - 1, start = 100，如果是后一种写法的话就会在计算end + start的时候越界，导致结果不正确。  

### 1.2. 二分查找实现  
&emsp; 二分查找可以有递归和非递归两种方式来实现。  

```java
/**
 * 非递归实现，while循环
 * @param array
 * @param target
 * @return
 */
public static int binarySearch(int []array,int target){

    //查找范围起点
    int start=0;
    //查找范围终点
    int end=array.length-1;
    //查找范围中位数
    int mid;

    while(start<=end){
        //mid=(start+end)/2 有可能溢出
        mid=start+(end-start)/2;
        if(array[mid]==target){
            return mid;
        }else if(array[mid]<target){
            start=mid+1;
        }else{
            end=mid-1;
        }
    }
    return -1;
}

/**
 * 递归实现
 * @param array
 * @param start
 * @param end
 * @param target
 * @return
 */
public static int recursionBinarySearch(int[] array,int start,int end,int target){
    if (start <= end) {
        int mid=start+(end-start)/2;
        if (target == array[mid]) {
            return mid;
        } else if (target < array[mid]) { //比关键字大则关键字在左区域
            return recursionBinarySearch(array, start, mid - 1, target);
        } else { //比关键字小则关键字在右区域
            return recursionBinarySearch(array, mid + 1, end, target);
        }
    } else {
        return -1;
    }
}

public static void main(String[] args) {
    int[] array = new int[1000];
    for(int i=0; i<1000;i++){
        array[i] = i;
    }
    System.out.println(binarySearch(array, 173));
    System.out.println(recursionBinarySearch(array, 0,array.length,173));
}
```  

### 1.3. 二分查找题型  

1. 找出第一个与key相等的元素
2. 找出最后一个与key相等的元素
3. 查找第一个等于或者大于Key的元素
4. 查找第一个大于key的元素
5. 查找最后一个等于或者小于key的元素
6. 查找最后一个小于key的元素  

