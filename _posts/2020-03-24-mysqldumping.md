---
title: "우분투 환경crontab으로 Mysql DB 백업하기"
date: 2020-03-24 17:23:28 -0400
categories: cron Mysql dump
---
shell file

mysql_backup.sh

위치
/home/ubuntu/

```

DATE_YYYYMMDDHHMMSS=`date '+%Y%m%d%H%M%S'`

dailysql=$DATE_YYYYMMDDHHMMSS'.sql'
password='[password]'

echo "mysql dailysql dump start.."

mysqldump -h [host address] -u '[username]' -p$password [DATABASE]> ./$dailysql 
#따로 경로를 지정하지 않아 /home/ubuntu/ 에 저장됨

echo 'dumpfile : '$dailysql

echo "mysql dailysql dump stop.."

```

crontab 에 스케줄러 추가하기
crontab은 사용자에 종속 (sudo를 할 경우 root 사용자 전용 crontab)
```
crontab -e
```
에디터 하단에 다음과 같이 추가 (매일 12시 백업진행)
```
0 12 * * * /home/ubuntu/mysql_backup.sh
```

참조

https://12bme.tistory.com/29

https://www.leafcats.com/94
