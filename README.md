## A. 資料庫

### 1. 請說明SQL與No-SQL的差異，並說明兩者的運⽤場景:
    SQL最大特性在於ACID，不管在甚麼場合取值，都是最完整的，其效能就仰賴cluster架構與redis如何搭配。

    No-SQL最大的特性在於，資料可以快速的丟向前台，效能上比SQL還要快，而它的ACID特性在於資料最後的樣子才符合。

    場景分為三種:

    　　一.使用SQL的話，主要就要求資料一致性並且對資料要求非常嚴景的狀況，像是:會計系統、金融相關、貨倉管理這類。

    　　二.使用NoSQL的話，就會是在需要及時處裡的資料，像是:BigDate、IOT、網頁遊戲、購物網站、留言板這類的。

    　　三.就是同時取其特長分開使用，就以社交平台為例:帳戶、朋友關係會使用SQL作為儲存資料的依據；評論、點讚、發布動態的資料就會以No-SQL來儲存資料。


### 2. 請說明何謂資料庫正規化:
    正規化的目的在於使資料庫的Table能夠越單純越簡潔越好，其帶來的好處可以使的資料庫在查詢或存儲時能夠有更好的操作效率，讓資料庫更好去做管理，同時也減少資料發生異常使資料正確可靠性更高。

    正規化是一個過程，透過1NF->2NF->3NF來實現:

    　　1NF : 一欄位只能有單一值、消除重複的欄位、決定PK

    　　2NF : 消除部分相關的欄位，並另外創建屬於他們的Table並用FK關聯(部分相依)

    　　3NF : 將與PK不相關的欄位再次獨立一張Table並用FK關聯(遞移相依)

    進階正規化 BCNF->4NF->5NF

    　　BCNF: 滿足3NF的情況下，對非PK欄位賦予(CK)候選鍵

    　　4NF : 滿足BCNF的情況下，消除多值依賴

    　　5NF : 滿足4NF的情況下，將關聯度過高的資料，拆分到新的Table，避免數據不一致

### 3. 請對以下資料進⾏正規化(⾄第三正規化)，請⾃⾏建⽴並將資料寫⼊SQLite3資料庫(檔案隨Django程式碼⼀同繳交)，資料庫將做為後續題⽬使⽤:

## B. Django
根據以上資料，透過Django框架進⾏開發(作業系統指定使⽤Ubuntu 20.04)，並提供
說明⽂件及原始碼。


## C. 伺服器部屬
請使⽤AWS免費⽅案或GCP前三個⽉免費300美元額度啟動⼀台虛擬機，將Django專
案搭配Apache 2網⾴伺服器進⾏部屬:
    我使用GCP，以下是我的設定:
![螢幕擷取畫面 2024-06-13 212504](https://github.com/DokuroTW/roar/assets/100449940/43b14340-d224-4016-94cf-cfc8f4bbe35a)

Computer Engine > VM執行個體 > 建立執行個體 選擇使用ubuntu20.04 與設定防火牆 http、https

![螢幕擷取畫面 2024-06-13 212615](https://github.com/DokuroTW/roar/assets/100449940/6c54b67c-a8af-4d3f-a92d-65b349d21ac3)

虛擬私有雲網路 > 防火牆政策 > 建立防火牆政策 > 設定 tcp:8000 與 udp:8000

![螢幕擷取畫面 2024-06-13 212649](https://github.com/DokuroTW/roar/assets/100449940/291c7883-7ff4-497e-86e1-b5c7115040ec)

Computer Engine > VM執行個體 > 選擇新建的執行個體 > 選擇新建立的防火牆政策 並確認 http與https通行

遠端SSH我使用Xshell與Xftp:

![螢幕擷取畫面 2024-06-13 213600](https://github.com/DokuroTW/roar/assets/100449940/a4e0a90e-d56d-4a8e-b349-ef30277758bc)

工具 > 使用者金鑰管理 > 產生 生成 publickey(RSA)

![螢幕擷取畫面 2024-06-13 213152](https://github.com/DokuroTW/roar/assets/100449940/c67a97af-ceb1-4feb-9636-25a03c5fca3e)

Computer Engine > 中繼資料 >安全殼層金鑰 > 將公鑰值填入

```
accound : djangotest
password : test123
```

設定完成後 Xshell 與 xftp 皆可 SSH

![螢幕擷取畫面 2024-06-13 213446](https://github.com/DokuroTW/roar/assets/100449940/acc09662-57ec-489a-b96d-2baa9e5bf153)

![螢幕擷取畫面 2024-06-13 213510](https://github.com/DokuroTW/roar/assets/100449940/5bbe6ccd-1c68-4c5c-a236-cda6ced2576d)

設定 apache2 的 config 在 site-available中

![螢幕擷取畫面 2024-06-17 212322](https://github.com/DokuroTW/roar/assets/100449940/713333b7-f029-4b61-8843-2d6bf07ba7a6)

```
<VirtualHost *:80>
    ServerAdmin webmaster@localhost
    ServerName 34.125.227.86

    ProxyPreserveHost On
    ProxyPass / http://localhost:8000/
    ProxyPassReverse / http://localhost:8000/

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>

```

## D. 網域設定
### 1. 請了解如何在Cloudflare中代管網域，將做為⼆⾯中⼝試題⽬之⼀。

    過去在公司使用的是中華電信的DNS服務，來對ip轉成domain的。

![螢幕擷取畫面 2024-06-13 150106](https://github.com/DokuroTW/roar/assets/100449940/22b4b6d1-94ff-4044-8979-4a10ee009f06)


