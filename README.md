## 概念
git push到git server後，git server若設置webhook，會發http post通知CICD server。  
CICD server是web server，在local端架設通常會有非公開網域的問題:  
1. 使用公開的git server(github)，無法呼叫local端的domain
2. 自架git server，在cicd server轉址到git server auth授權時，如果使用的是docker，由於docker container網路機制，
   使用127.0.0.1是無法成功的

> 可以使用ngrok將local端的服務掛載到public domain，非常適合開發測試階段


## Gitea + Drone
參考:
https://jiaming0708.github.io/2021/03/22/gitea-drone-docker/

1. 打開localhost:3000設定完gitea，其中要設定auth，允許從drone訪問時顯示授權存取
2. 訪問localhost:8000時，會導到gitea(localhost:3000)授權頁。

如同網頁所提到的，出現
> Login Failed. Post "http://localhost:3000/login/oauth/access_token": dial tcp 127.0.0.1:3000: connect: connection refused

需要安裝 ngrok
設定好後，再次訪問drone-server
授權頁出現授權失敗

![Unregistered_redirect](https://user-images.githubusercontent.com/24542187/193612764-64cc1405-e12a-443c-af6a-f1caad44120d.jpg)


Unregistered Redirect URI
這是因為重新導向URI的問題

既然已經導到gitea server，表示導過來沒問題，
是倒回去drone server出錯，
需檢視gitea所設定的 OAuth2 導回drone server的url是否有錯
注意重新導向 drone-server的 protocol須與DRONE_SERVER_PROTO一致

### 測試
成功進去後，開始安裝 drone runner
照參考網站上面的修改設定檔成功後，


可以開始測試上傳
這邊參考網站跳過一些步驟，直接倒了push一個commit
在認證成功後可以在drone頁面看到已經多了一個repository
點進去頁面後可以啟用該repo
啟用後的會發現在gitea建立了一個webhook


然後就可以開始測試=>push一個commit


push完後觀察ACTIVITY FEED會發現一直處於在Pending狀態
1. 根據此篇有個錯誤是因為gitea上傳到localhost:3000沒有改成ngrok的domain
   將gitea container的ROOT_URL 改成ngrok的domain之後
   依然是處於pending狀態

2. 發現drone-runner的log一直跳出
```
level=error msg="cannot ping the remote server" error="Temporary Redirect"
```
將`DRONE_RPC_PROTO`改成https之後就沒有跳出該訊息
但push commit依然還是處於pending狀態

3. 最後發現是m1問題，將 __.drone.yml__ 加上
```yml
platform:
  arch: arm64
```

終於順利看到drone.yml定義的test任務執行


--------
重新安裝完後有時候會出現錯誤400的狀況，猜測是因為用ngrok
或者因為ngrok是使用https的關係
