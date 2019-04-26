---
layout: post
title : "동일한 조건으로 update시에 처음 실행만 반영되고 그 다음부터는 반영이 안되는 현상"
subtitle : "2019-04-26-Mysql-Part-Update"
date: 2019-04-26 15:18:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [java, mysql]
---

동일한 조건으로 update시에 처음 실행만 반영되고 그 다음부터는 반영이 안되는 현상
=========

작업 중 다량의 데이터를 Update 쿼리를 실행할 과정이 필요하였습니다.
한번의 많은 양의 데이터를 Update 한다면 DB의 커넥션이 끊어진다거나. 메모리 이슈가 생겨
증분으로 업데이트 시도하였습니다. 코드는 아래와 같습니다.

[method]
```
    int changeCntSum = 0;    // 업데이트된 데이터의 총 개수
    while (true) {
        int changCnt = 0;    // 업데이트된 데이터 개수
        changeCnt = updateMapper.updateData(...매개변수..);
        changeCntSum += changeCnt;

        // 업데이트할 데이터의 개수가 없을 때 while문 break
        if (changeCnt == 0) {
            break;
        }
    }
```

[query]
```
    UPDATE test_table
    SET
        mod_user_id = #{userId},
        status = status | 2
    WHERE mall_id = #{mallId}
    AND brand = #{brand}
    LIMIT 10000
```

하지만 위와 같이 작성하여 실행하면 changeCnt 값이 첫 update 시만 limit 만큼 변경개수를 리턴하고 
그 다음부터 계속 0값만 리턴하고있는 문제가 발생합니다.

> 원인은 쿼리캐시 문제였습니다. 동일한 조건으로 계속 업데이트를 치니 쿼리캐시에 걸려 다른 데이터 반영이 안되는 것이였습니다. 그래서 증분업데이트를 size와 offset값을 기준으로 Looping하여 Update하는 방법으로 변경하였고 아래 코드를 참고하시면 됩니다.

[method]
```
    int page = 1;
    int size = 10000;
    int changeCntSum = 0;    // 업데이트된 데이터의 총 개수
    while (true) {
        int changCnt = 0;    // 업데이트된 데이터 개수
        Integer offset = (page - 1) * size;

        changeCnt = updateMapper.updateData(...매개변수..); // size, offset 추가
        changeCntSum += changeCnt;

        // 업데이트할 데이터의 개수가 없을 때 while문 break
        if (changeCnt == 0) {
            break;
        }

        page++;
    }
```

[query]
```
    UPDATE test_table AS upd_t
    INNER JOIN (
        SELECT product_id
        FROM test_table
        WHERE mall_id = #{mallId}
        AND brand = #{brand}
        LIMIT #{size} OFFSET #{offset}
    ) AS join_t ON upd_t.product_id = join_t.product_id
    SET
        upd_t.mod_user_id = #{userId},
        upd_t.status = status | 2
    WHERE upd_t.mall_id = #{mallId}
    AND upd_t.brand = #{brand}
```

