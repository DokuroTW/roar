## 除了B-c，ABCD其他問題都已經回答

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

腳本`InsertData.py` 位置在 羅爾科技測題\專案原始碼\MPInformation\InsertData.py

```
import os
import django
import requests
from datetime import datetime
from django.utils import timezone

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "MPInformation.settings")
django.setup()

from MusicApp.models import Event, Category, Unit, EventUnit, ShowInfo

def FetchData(url):
    response = requests.get(url)
    response.raise_for_status()
    return response.json()


def ParseDatetime(date_string):
    if date_string:
        dt = datetime.strptime(date_string, '%Y/%m/%d %H:%M:%S')
        return timezone.make_aware(dt, timezone.get_current_timezone())
    return None


def InsertData(data):
    for record in data:
        category, _ = Category.objects.get_or_create(name=record['category'])

        event, created = Event.objects.update_or_create(
            uid=record['UID'],
            defaults={
                'version': record['version'],
                'title': record['title'],
                'category': category,
                'show_unit': record.get('showUnit'),
                'description': record.get('descriptionFilterHtml'),
                'discount_info': record.get('discountInfo'),
                'image_url': record.get('imageUrl'),
                'web_sales': record.get('webSales'),
                'source_web_promote': record.get('sourceWebPromote'),
                'comment': record.get('comment'),
                'edit_modify_date': ParseDatetime(record.get('editModifyDate')),
                'source_web_name': record['sourceWebName'],
                'start_date': timezone.make_aware(datetime.strptime(record['startDate'], '%Y/%m/%d'),
                                                  timezone.get_current_timezone()),
                'end_date': timezone.make_aware(datetime.strptime(record['endDate'], '%Y/%m/%d'),
                                                timezone.get_current_timezone()),
                'hit_rate': record['hitRate']
            }
        )

        if not created:
            EventUnit.objects.filter(event=event).delete()
            ShowInfo.objects.filter(event=event).delete()

        units = ['masterUnit', 'subUnit', 'supportUnit', 'otherUnit']
        for unit_type in units:
            for unit_name in record[unit_type]:
                unit, _ = Unit.objects.get_or_create(name=unit_name)
                EventUnit.objects.create(event=event, unit=unit, role=unit_type)

        for show in record['showInfo']:
            ShowInfo.objects.create(
                event=event,
                time=ParseDatetime(show['time']),
                location=show['location'],
                location_name=show['locationName'],
                on_sales=show['onSales'] == 'Y',
                latitude=float(show['latitude']) if show['latitude'] else None,
                longitude=float(show['longitude']) if show['longitude'] else None,
                price=show['price'],
                end_time=ParseDatetime(show['endTime'])
            )


if __name__ == '__main__':
    url = 'https://cloud.culture.tw/frontsite/trans/SearchShowAction.do?method=doFindTypeJ&category=1'
    data = FetchData(url)
    InsertData(data)

```
前置作業: 必須先將model.py的欄位給創建好，並同步到SQLite中
```
python3 manage.py makemigrations
python3 manage.py migrate
```
我透過DB工具 `DBeaver` 來查看資料確認匯入

![image](https://github.com/DokuroTW/roar/assets/100449940/d3cea7b8-3731-4d2f-b341-caaf57a1a728)



## B. Django
根據以上資料，透過Django框架進⾏開發(作業系統指定使⽤Ubuntu 20.04)，並提供
說明⽂件及原始碼。


#### 專案位置: 羅爾科技測題\專案原始碼\MPInformation

這是我的頁面，只顯示Event中，標題與內容這兩個欄位，在登入前只能夠使用搜尋功能；登入後可以使用搜尋、編輯及刪除的功能。

```
測試帳號:djangotest
測試密碼:test123
```

## 未登入畫面

![image](https://github.com/DokuroTW/roar/assets/100449940/db17f618-8e23-4634-8ee0-a0fae67cfae7)

## 登入畫面

![image](https://github.com/DokuroTW/roar/assets/100449940/3401c36a-d6f4-43c3-b710-369cd7e9b836)



### c. ⾃動更新資料
撰寫程式⾃動於每⽇01:00:00(UTC+8)從政府資料開放平臺抓取資料並與更新資料庫，爬取紀錄儲存於MongoDB中，紀錄內容不限，⽤以檢視是否了解如何透過Python操作MongoDB。
MongoDB請參考官⽅操作⼿冊，安裝於本地端。

```
本人不會使用MongoDB，會再花時間去學習
```


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
accound　:　djangotest
password　:　test123
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

反向代理會使用nginx並上CA，使其變為SSL，以下是我的nginx.conf的設定檔:
```
server {
    listen 80;
    server_name 34.125.227.86;

    # Redirect HTTP to HTTPS
    return 301 https://$host$request_uri;

    server_tokens off;

    # Add HSTS header
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
    add_header X-Frame-Options "DENY" always; 
    add_header Cache-Control "no-cache, no-store, must-revalidate";
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block";
    add_header Referrer-Policy "same-origin";

    access_log  /var/log/nginx/euptop-access.log;
    error_log   /var/log/nginx/euptop-error.log;

    location / {
        proxy_pass http://127.0.0.1:8069;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Host $http_host;
        proxy_set_header X-Forwarded-Host $host;
	proxy_cookie_path / "/; secure; HttpOnly; SameSite=Strict";
	proxy_hide_header X-Frame-Options;
	proxy_hide_header X-Runtime;
	proxy_hide_header X-powered-by;
	autoindex off;
        proxy_redirect off;
    }
}

# HTTPS server block
server {
    listen 443 ssl http2;
    server_name 34.125.227.86;

    server_tokens off;

    ssl_certificate /etc/letsencrypt/live/euptop.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/euptop.com/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers off;
    ssl_session_timeout 10m;
    ssl_session_cache shared:SSL:10m;
    ssl_session_tickets off;
    ssl_stapling on;
    ssl_stapling_verify on;

    resolver 8.8.8.8 8.8.4.4 valid=300s;
    resolver_timeout 5s;

    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
    add_header X-Frame-Options "DENY" always; 
    add_header Cache-Control "no-cache, no-store, must-revalidate";
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block";
    add_header Referrer-Policy "same-origin";

    access_log  /var/log/nginx/euptop-ssl-access.log;
    error_log   /var/log/nginx/euptop-ssl-error.log;

    location / {
        proxy_pass http://127.0.0.1:8069;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Host $http_host;
        proxy_set_header X-Forwarded-Host $host;
	proxy_cookie_path / "/; secure; HttpOnly; SameSite=Strict";
	proxy_hide_header X-Frame-Options;
	proxy_hide_header X-Runtime;
	proxy_hide_header X-powered-by;
	autoindex off;
        proxy_redirect off;
    }
}
```

但目前了解下來，cloudflare可以做到更多，包含domain轉至給其代管、代管後享有DDOS基本防護(透過加入hash在封包中)，跟過去我所接觸的incapsula解決方式不同(透過引導流量)，也支援雲端WAF或是CDN，總之只要是與proxy相關的，皆可以在他們代管下實現，同時SSL也是免費使用(目前瞭解下來並不用定時更換)。

並起可以透過UI進行操作:

![螢幕擷取畫面 2024-06-17 213432](https://github.com/DokuroTW/roar/assets/100449940/5cdb2ab7-cdc8-4888-a5cd-76f78e78af6e)

