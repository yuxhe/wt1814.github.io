
<!-- TOC -->

- [1. 查找算法](#1-查找算法)
    - [1.1. 二分查找](#11-二分查找)
        - [1.1.1. 二分查找模版](#111-二分查找模版)
        - [1.1.2. 二分查找实现](#112-二分查找实现)
        - [1.1.3. 二分查找题型](#113-二分查找题型)

<!-- /TOC -->


<!-- 


-->

<!-- 
三款经典的查找算法
https://mp.weixin.qq.com/s/3RvYUaAL8xAQQvT88WAJ7g

在N个乱序数字中查找第k大的数字
https://blog.csdn.net/u010412301/article/details/67704530
寻找第K大数的方法
https://blog.csdn.net/csl13/article/details/6056522
-->

# 1. 查找算法  
&emsp; 顺序查找、二分查找、分块查找、哈希表查找  
![image](https://gitee.com/wt1814/pic-host/raw/master/images/java/function/function-32.png)  

## 1.1. 二分查找  
&emsp; 要在 **<font color = "lime">有序数组</font>** 中进行查找，最好的解决方法是使用二分查找算法。  

### 1.1.1. 二分查找模版   

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

### 1.1.2. 二分查找实现  
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

### 1.1.3. 二分查找题型  
&emsp; 很多的时候，应用二分查找的地方都不是直接的查找和key相等的元素，而是使用下面提到的二分查找的各个变种。  

1. 找出第一个与key相等的元素  

```java
int searchFirstEqual(int *arr, int n, int key)
{
    int left = 0, right = n-1;
    while(left<=right) {
        int mid = (left+right)/2;
        if(arr[mid] >= key) right = mid - 1;
        else if(arr[mid] < key) left = mid + 1;
    }
    if( left < n && arr[left] == key) return left;
    return -1;
}
```
2. 找出最后一个与key相等的元素  

```java
int searchLastEqual(int *arr, int n, int key)
{
   int left = 0, right = n-1;
   while(left<=right) {
       int mid = (left+right)/2;
       if(arr[mid] > key) right = mid - 1;
       else if(arr[mid] <= key) left = mid + 1;
   }
   if( right>=0 && arr[right] == key) return right;
    return -1;
}
```
3. 查找第一个等于或者大于Key的元素  

```java
int searchFirstEqualOrLarger(int *arr, int n, int key)
{
   int left=0, right=n-1;
   while(left<=right) {
       int mid = (left+right)/2;
       if(arr[mid] >= key) right = mid-1;
       else if (arr[mid] < key) left = mid+1;
   }
   return left;
}
```
4. 查找第一个大于key的元素  

```java
int searchFirstLarger(int *arr, int n, int key)
{
   int left=0, right=n-1;
   while(left<=right) {
       int mid = (left+right)/2;
       if(arr[mid] > key) right = mid-1;
       else if (arr[mid] <= key) left = mid+1;
   }
   return left;
}
```
5. 查找最后一个等于或者小于key的元素  

```java
int searchLastEqualOrSmaller(int *arr, int n, int key)
{
   int left=0, right=n-1;
   while(left<=right) {
       int m = (left+right)/2;
       if(arr[m] > key) right = m-1;
       else if (arr[m] <= key) left = m+1;
   }
   return right;
}
```
6. 查找最后一个小于key的元素    

```java
int searchLastSmaller(int *arr, int n, int key)
{
   int left=0, right=n-1;
   while(left<=right) {
       int mid = (left+right)/2;
       if(arr[mid] >= key) right = mid-1;
       else if (arr[mid] < key) left = mid+1;
   }
   return right;
}
```

&emsp; 下面是一个测试的例子：  

```java
int main(void)
{
   int arr[17] = {1,
                  2, 2, 5, 5, 5,
                  5, 5, 5, 5, 5,
                  5, 5, 6, 6, 7};
   printf("First Equal           : %2d \n", searchFirstEqual(arr, 16, 5));
   printf("Last Equal            : %2d \n", searchLastEqual(arr, 16, 5));
   printf("First Equal or Larger : %2d \n", searchFirstEqualOrLarger(arr, 16, 5));
    printf("First Larger          : %2d \n", searchFirstLarger(arr, 16, 5));
    printf("Last Equal or Smaller : %2d \n", searchLastEqualOrSmaller(arr, 16, 5));
    printf("Last Smaller          : %2d \n", searchLastSmaller(arr, 16, 5));
    system("pause");
    return 0;
}
```
&emsp; 最后输出结果是：  

```java
First Equal           :  3
Last Equal            : 12
First Equal or Larger :  3
First Larger          : 13
Last Equal or Smaller : 12
Last Smaller          :  2
```
