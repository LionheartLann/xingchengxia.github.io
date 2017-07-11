---
title: Best Time to Buy and Sell Stock
date: 2017-07-11 10:40:25
tags:
---
# Best Time to Buy and Sell Stock(121)
#LeetCode
# Problem
Say you have an array for which the ith element is the price of a given stock on day i.

If you were only permitted to complete at most one transaction (ie, buy one and sell one share of the stock), design an algorithm to find the maximum profit.

# Example
> Input: [7, 1, 5, 3, 6, 4]  Output: 5  
> max. difference = 6-1 = 5 (not 7-1 = 6, as selling price needs to be larger than buying price)  
>   
> Input: [7, 6, 4, 3, 1]  Output: 0  
> In this case, no transaction is done, i.e. max profit = 0.  

# My Solution
Basically, assign two pointers which point to head and head->next, the left one p1 aims at the smaller price, while the right one p2 moves forward to find the higher price. if p1 > p2, move both pointers forward. By comparing the profit each move (when p1 < p2), we can find the maximum profit.
* Time: O(n)
* Space: O(1)
## Note
* At the aspect of C, `int* p2 = p1+1;`  moves the pointer to the next element of the array. The 1 is in fact the `sizeof(int)`.  
```
int maxProfit(int* prices, int pricesSize) {
    int i = 0;
    int* p1 = prices;
    int* p2 = p1+1;
    int max = 0;
    int profit = 0;
    
    if(pricesSize<2){
        return 0;
    }
    while(p2 && i < pricesSize -1){
        if(*p2 > *p1){
          profit = *p2 - *p1;
          if(max < profit){
            max = profit;
          }   
        }else{
          p1 = p2; 
        }   
        p2 = p2+1;
        i++;
    }
    return max;
}
```

# Solution II
This solution is the implement of editorial solution, which maintains one pointer to update the profit  by comparing current price to low_price.
```
int maxProfit(int* prices, int pricesSize) {
    int max = 0;
    int profit = 0;
    int low_price = prices[0];
    if(pricesSize<2){
        return 0;
    }
    for(int i = 0; i<= pricesSize -1; i++){
        if(*prices < low_price){
            low_price = *prices;
        }else{
            profit = *prices - low_price;
            if(max<profit){
                max=profit;
            }
        }
        prices = prices + 1;
    }
    return max;
}
```

[Best Time to Buy and Sell Stock - LeetCode](https://leetcode.com/problems/best-time-to-buy-and-sell-stock/#/description)
