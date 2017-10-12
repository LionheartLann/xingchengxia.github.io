---
title: KMeans Clustering
date: 2017-08-22 18:52:41
tags:
---
# K-means聚类
#lannister/machinelearning

k-means算法以k为参数,把n个对象分成k个簇,使簇内具有较高的相似度,而簇间的相似度较低。其处理过程如下:
1.随机选择k个点作为初始的聚类中心; 
2.对于剩下的点,根据其与聚类中心的距离,将其归入最近的簇 
3.对每个簇,计算所有点的均值作为新的聚类中心 
4.重复2、3直到聚类中心不再发生改变

Clustering is to find the relations/connections between data without labels.
K-means is one of the most widely used algorithms.

K-means for non-separated clusters(T-shirt sizing)


* Find closest centroids
```
K = size(centroids, 1);
distance = zeros(K, 1); % to store and return the min distance
idx = zeros(size(X,1), 1);

for i = 1:size(X, 1)
  for k = 1:K
    distance(k) = sqrt(sum((X(i,:)-centroids(k,:)).^2));
  end
  [mini, index] = min(distance);
   idx(i) = index;
end
```

* Compute Means
```
[m n] = size(X);
centroids = zeros(K, n);

for k=1:K
  logic = idx==k;
  centroids(k,:) = 1/sum(logic)*sum(X.*logic); 
  % sum(logic) is the number of examples assigned to kth centroid
end
```

* Randomly initialize cluster centroids
```
centroids = zeros(K, size(X, 2));
randidx = randperm(size(X, 1));
% Take the first K examples as centroids
centroids = X(randidx(1:K), :);
```

* K-Means Clustering on Pixels
```
% Run K-Means
for i=1:max_iters
    
    % Output progress
    fprintf('K-Means iteration %d/%d...\n', i, max_iters);
    if exist('OCTAVE_VERSION')
        fflush(stdout);
    end
    
    % For each example in X, assign it to the closest centroid
    idx = findClosestCentroids(X, centroids);
        
    % Given the memberships, compute new centroids
    centroids = computeCentroids(X, idx, K);
end
```


