---
title: "Redis vs Memcached: 選哪一個？"
date: 2024-06-07
categories:
  - Cache
tags:
  - Cache
---

如果你正在考慮 server-side cache 解決方案，可能聽說過 [Redis](https://redis.io/) 或 [Memcached](https://www.memcached.org/)。

## Memcached 和 Redis 有什麼不同？

Memcached 和 Redis 都將資料儲存在記憶體中以提高應用程式速度，但它們滿足不同的需求。

- Memcached 提供了基本的 key-value 形式儲存，並且提供了多線程 (multi-threaded) 來處理緩存數據。
- Redis 支援更多的數據類型和功能，像是 backup、replication、encryption、multi-database、Pub/Sub 以上功能。
- 每單位儲存大小 Redis 最大可以儲存 512 MB，則 Memcached 最大儲存只有 1 MB，如果應用需要儲存超過 1 MB 的數據，Redis 會是更好的選擇。

## 不同的記憶體管理方案

#### Redis
資料並非全都存儲在內存中。當內存使用超過上限時，Redis 會將不常用的值存到硬碟。Redis使用 malloc/free進行內存管理，儘管簡單易用，但可能導致內存碎片。為了提高性能，Redis 還可以配置I/O線程以並行處理讀寫請求。

Redis單線程是指獲取 (socket 讀)、解析、執行、內容返回 (socket 寫) 等都由一個順序串行的主線程處理，這個主線程就是我們說的單線程。

雖然 Redis 被稱為單線程，因為核心邏輯確實是由一個線程處理的，但實際上 Redis 可以在同一個 Process 下使用多個 I/O 多線程來處理網絡數據的讀寫和協議解析，從而減輕主線程的負擔，提高並發性能。


#### Memcached
使用 Slab Allocation 機制進行內存管理，將內存劃分為不同大小的 chunk，並將相同大小的 chunk 分組到 Slab Class 中，從而減少內存碎片， Slab Allocation 處理大部分內存請求，而其他少量內存請求則使用 malloc/free。

每個 Slab Class 中的 chunk 大小可以通過設置 Growth Factor 來控制，假設圖中的 Growth Factor 為1.25。如果第一組中的 chunk 大小為 88 位元組，則第二組中的 chunk 大小將為 112 位元組。其餘塊遵循相同的規則。

![Slab Class](https://miro.medium.com/v2/resize:fit:640/format:webp/0*HigiL27s2I7W_lEN.png)

其最大的缺陷是可能造成空間浪費。由於系統會為每個 chunk 分配特定長度的記憶體空間，因此較長的資料可能無法充分利用該空間。如圖所示，當我們將 100 位元組的資料快取到 128 位元組的 chunk 中時，未使用的 28 位元組就被浪費了。

![Chunk](https://miro.medium.com/v2/resize:fit:640/format:webp/0*geF42vL6L0Qxv563.png)

## Redis 支援資料持久化

Redis 資料持久化主要有兩種不同的方式：

1. RDB snapshot：這是一個在特定時間點對整個數據集的快照，存儲在磁碟上的一個文件中，並在指定的間隔時間執行。這樣，在啟動時可以恢復數據集。

2. AOF log：這是一個僅追加文件（Append Only File）日誌，記錄了 Redis 服務器上執行的所有寫入命令。這個文件也存儲在磁碟上，因此通過按順序重新執行所有命令，可以在啟動時恢復數據集。

這些文件由子進程處理，這是決定使用哪種持久化方式的一個關鍵因素。

如果 Redis 中存儲的數據集過大，RDB 文件的生成將需要一些時間，這會影響響應時間。然而，與 AOF 日誌相比，它在啟動時的加載速度更快。

如果完全無法接受數據丟失，AOF 日誌會更好，因為它可以在每次命令後更新。由於它是僅追加文件，因此不會有數據損壞問題。然而，它的大小可能比 Snapshot 大得多。

相比之下，Memcached 不支持持久化。這代表，當服務器重新啟動時，所有存儲在 Memcached 中的數據都會遺失。這使得 Memcached 更適合用於簡單且高速的緩存，而不是用作數據存儲。

## Memcached 和 Redis 哪個更好?

如果你需要一個簡單高效的緩存解決方案，Memcached 可能是更好的選擇。如果你需要更多功能和更高的數據持久性，Redis 將是更好的選擇。了解兩者的特點和應用場景，可以幫助你在合適的場景中發揮它們的最大效能。

## 參考資料

[Memcached vs Redis: which one to choose?](https://www.imaginarycloud.com/blog/redis-vs-memcached/)
[Redis vs. Memcached: In-Memory Data Storage Systems](https://alibaba-cloud.medium.com/redis-vs-memcached-in-memory-data-storage-systems-3395279b0941)
[Hotspot-Aware Hybrid Memory Management for In-Memory Key-Value Stores](https://ieeexplore.ieee.org/abstract/document/8859283)
[Redis vs Memcached 比較](https://medium.com/jerrynotes/redis-vs-memcached-%E6%AF%94%E8%BC%83-15d2ba829da7)