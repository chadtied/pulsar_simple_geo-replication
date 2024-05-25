# 簡易實作pulsar geo-repliction概念





![image](https://github.com/chadtied/pulsar_simple_geo-replication/assets/96424234/c88d2a3b-4582-41d7-9ade-8fd0bf2673b8)





概念說明: 以上圖說明 當producer傳一份資料給Topic T1的時候，clusterA和clusterB會個複製一份資料，當其中一個cluster出問題導致資料遺失或受損時，就可以取另一cluster的複製檔。






### 實作內容:



## 1. docker建立cluster

透過docker建立clusterA、clusterB，還有各cluster的bookie、zookeeper、broker；下圖有三個broker，目前實作只有兩個，分別屬於clusterA、clusterB
![image](https://github.com/chadtied/pulsar_simple_geo-replication/assets/96424234/010f888a-8cd6-4552-9690-3033ecf86e8f)

在當前目錄下建置docker-compose.yaml，並執行

linux
```sudo docker-compose up```
windows
```docker-compose up```
Mac/ios
```有錢人店店```

查看目前docker狀況
```sudo docker ps```


## 2. Cluster設置

先透過clusterA向clusterB建立連線
```bin/pulsar-admin --admin-url http://localhost:8080 cluster create cluster-b --broker-url pulsar://broker-edge1:6650 --url http://broker-edge1:8080```
反之透過clusterB向clusterA建立連線
```bin/pulsar-admin --admin-url http://localhost:8081 clusters create cluster-a --broker-url pulsar://broker-main:6650 --url http://broker-main:8080```

接下來，在這兩個cluster下建立共用namespace
```bin/pulsar-admin --admin-url http://localhost:8080 tenants create edge1 --allowed-clusters cluster-a,cluster-b
bin/pulsar-admin --admin-url http://localhost:8081 tenants create edge1 --allowed-clusters cluster-a,cluster-b
bin/pulsar-admin --admin-url http://localhost:8080 namespaces create edge1/replicated --clusters cluster-a,cluster-b
bin/pulsar-admin --admin-url http://localhost:8081 namespaces create edge1/replicated --clusters cluster-a,cluster-b```


最後一步，我們要在namespace內建立topic，也就是producer、consumer訂閱，並處理資料的地方
```bin/pulsar-admin --admin-url http://localhost:8080 topics create edge1/replicated/events```



## 3. 放置監聽器

我們透過放置監聽器，確定producer、consumer的資料蒐集狀況

新開分頁，建立clusterA /edge1/replicated/events的監聽器
```bin/pulsar-client --url http://localhost:8080 --listener-name external consume --subscription-name "sub-a" persistent://edge1/replicated/events -n 0```
新開分頁，建立clusterB /edge1/replicated/events的監聽器
```bin/pulsar-client --url http://localhost:8081 --listener-name external consume --subscription-name "sub-b" persistent://edge1/replicated/events -n 0```


## 4. 觀測結果

向```clusterA > event``` topic 傳入訊息
```bin/pulsar-client --url http://localhost:8080 --listener-name external produce persistent://edge1/replicated/events --messages "Winter is so cute" ```
會發現clusterA、clusterB的listener都收到了有意義的訊息

反之，向```clusterB > event``` topic 傳入訊息
```bin/pulsar-client --url http://localhost:8081 --listener-name external produce persistent://edge1/replicated/events --messages "Winter is so cute" ```
也會得到相同結果

那如果使用```disable-replication```傳呢 ?
```bin/pulsar-client --url http://localhost:8081 --listener-name external produce --disable-replication persistent://edge1/replicated/events --messages "However IU is the cutest"```
會發覺只有傳入的cluster得到訊息而已，此處是clusterB

## 5. 停用並卸除docker

linux:
```sudo docker stop $(sudo docker ps -aq)```
```sudo docker rm $(sudo docker ps -aq)```
windows
```docker stop $(docker ps -aq)```
```docker rm $(docker ps -aq)```

