# SQL_MASTER 5주차 정규과제

📌SQL MASTER 정규과제는 매주 정해진 분량의 『*데이터 분석을 위한 SQL 레시피*』 를 읽고 학습하는 것입니다. 이번 주는 아래의 **SQL_MASTER_5th_TIL**에 나열된 분량을 읽고 공부하시면 됩니다.

아래 실습을 수행하며 학습 내용을 직접 적용해보세요. 단순히 결과를 재현하는 것이 아니라, SQL을 직접 작성하는 과정에서 개념을 스스로 정리하는 것이 중요합니다.

필요한 경우 교재와 추가 자료를 참고하여 이해를 보완하시기 바랍니다.

## SQL_MASTER_5th_TIL

### 5장 사용자를 파악하기 위한 데이터 추출
#### 2. 시계열에 따른 사용자 전체의 상태 변화 찾기
#### 3. 시계열에 따른 사용자의 개별적인 행동 분석하기 


## Study Schedule

| 주차  | 공부 범위     | 완료 여부 |
| ----- | ------------- | --------- |
| 1주차 | p.20~50    | ✅         |
| 2주차 | p.52~136   | ✅         |
| 3주차 | p.138~184  | ✅         |
| 4주차 | p.186~232 | ✅         |
| 5주차 | p.233~321 | ✅         |
| 6주차 | p.324~406 | 🍽️         |
| 7주차 | p.408~464 | 🍽️         |

<br>

<!-- 여기까진 그대로 둬 주세요-->


# 실습 

## 0. 실습 규칙

1. 샘플 데이터 생성 코드는 **07_SQL_MASTER_Template/src** 경로에 장별로 정리되어 있습니다.
2. 아래 목차에 맞춰 해당 코드를 실행하여 샘플 데이터를 생성한 후, 각 장에서 요구하는 쿼리를 직접 작성해보시기 바랍니다.
3. 작성한 쿼리의 **실행 결과 화면도 함께 제출**해 주세요.
4. 단순히 교재의 예시 코드를 그대로 작성하는 것이 아니라, **제시된 로직을 충분히 이해한 뒤 교재를 보지 않고 스스로 쿼리를 구성**해보는 것을 권장합니다.
5. 교재 예시는 PostgreSQL, Hive, BigQuery 등 다양한 DBMS 기준으로 제시되어 있기 때문에, **MySQL이 아닌 다른 SQL 환경을 사용하여 실습을 진행해도 무방합니다.**
6. 다만, 사용 중인 DBMS에 맞는 문법으로 적절히 변환하여 작성하시기 바랍니다.

## 2. 시계열에 따른 사용자 전체의 상태 변화 찾기

### 2-1 등록 수의 추이와 경향 보기

<!-- 이 부분을 지우고 새롭게 배운 내용을 자유롭게 정리해주세요. -->

```sql
WITH daily_register AS (
    SELECT
        DATE(register_date) AS register_day,
        COUNT(*) AS register_user_count
    FROM mst_users
    GROUP BY DATE(register_date)
),
register_trend AS (
    SELECT
        register_day,
        register_user_count,
        LAG(register_user_count) OVER (ORDER BY register_day) AS prev_day_count,
        AVG(register_user_count) OVER (
            ORDER BY register_day
            ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
        ) AS moving_avg_7d
    FROM daily_register
)
SELECT
    register_day,
    register_user_count,
    prev_day_count,
    register_user_count - prev_day_count AS diff_from_prev_day,
    moving_avg_7d
FROM register_trend
ORDER BY register_day;
```

![](image/0406_01.png)

### 2-2 지속률과 정착률 산출하기

<!-- 이 부분을 지우고 새롭게 배운 내용을 자유롭게 정리해주세요. -->

```sql
WITH action_with_register AS (
    SELECT
        u.user_id,
        DATE(u.register_date) AS register_day,
        DATE(a.stamp) AS action_day,
        DATEDIFF(DATE(a.stamp), DATE(u.register_date)) AS diff_day
    FROM mst_users u
    LEFT JOIN action_log a
        ON u.user_id = a.user_id
),
user_retention_flag AS (
    SELECT
        user_id,
        register_day,
        MAX(CASE WHEN diff_day = 1 THEN 1 ELSE 0 END) AS repeat_1d,
        MAX(CASE WHEN diff_day BETWEEN 7 AND 13 THEN 1 ELSE 0 END) AS retention_7d,
        MAX(CASE WHEN diff_day BETWEEN 14 AND 20 THEN 1 ELSE 0 END) AS retention_14d,
        MAX(CASE WHEN diff_day BETWEEN 21 AND 27 THEN 1 ELSE 0 END) AS retention_28d
    FROM action_with_register
    GROUP BY user_id, register_day
)
SELECT
    register_day,
    COUNT(*) AS registered_users,
    AVG(repeat_1d) * 100 AS repeat_1d_rate,
    AVG(retention_7d) * 100 AS retention_7d_rate,
    AVG(retention_14d) * 100 AS retention_14d_rate,
    AVG(retention_28d) * 100 AS retention_28d_rate
FROM user_retention_flag
GROUP BY register_day
ORDER BY register_day;
```

![](image/0406_02.png)

### 2-3 지속과 정착에 영향을 주는 액션 집계하기 

<!-- 이 부분을 지우고 새롭게 배운 내용을 자유롭게 정리해주세요. -->

```sql
WITH action_with_register AS (
    SELECT
        u.user_id,
        DATE(u.register_date) AS register_day,
        a.action,
        DATE(a.stamp) AS action_day,
        DATEDIFF(DATE(a.stamp), DATE(u.register_date)) AS diff_day
    FROM mst_users u
    LEFT JOIN action_log a
        ON u.user_id = a.user_id
),
user_action_flag AS (
    SELECT
        user_id,
        register_day,
        action,
        MAX(CASE WHEN diff_day BETWEEN 0 AND 6 THEN 1 ELSE 0 END) AS did_action_in_7d,
        MAX(CASE WHEN diff_day BETWEEN 21 AND 27 THEN 1 ELSE 0 END) AS retained_28d
    FROM action_with_register
    GROUP BY user_id, register_day, action
)
SELECT
    action,
    COUNT(*) AS users,
    AVG(did_action_in_7d) * 100 AS action_usage_rate,
    AVG(CASE WHEN did_action_in_7d = 1 THEN retained_28d END) * 100 AS retained_if_action,
    AVG(CASE WHEN did_action_in_7d = 0 THEN retained_28d END) * 100 AS retained_if_no_action
FROM user_action_flag
GROUP BY action
ORDER BY action;
```

![](image/0406_03.png)
 
### 2-4 액션 수에 따른 정착률 집계하기 

<!-- 이 부분을 지우고 새롭게 배운 내용을 자유롭게 정리해주세요. -->

```sql
WITH action_with_register AS (
    SELECT
        u.user_id,
        a.action,
        DATEDIFF(DATE(a.stamp), DATE(u.register_date)) AS diff_day
    FROM mst_users u
    LEFT JOIN action_log a
        ON u.user_id = a.user_id
),
action_count AS (
    SELECT
        user_id,
        action,
        SUM(CASE WHEN diff_day BETWEEN 0 AND 6 THEN 1 ELSE 0 END) AS action_cnt_7d,
        MAX(CASE WHEN diff_day BETWEEN 21 AND 27 THEN 1 ELSE 0 END) AS retained_28d
    FROM action_with_register
    GROUP BY user_id, action
),
bucketed AS (
    SELECT
        user_id,
        action,
        retained_28d,
        CASE
            WHEN action_cnt_7d = 0 THEN '0'
            WHEN action_cnt_7d BETWEEN 1 AND 5 THEN '1 ~ 5'
            WHEN action_cnt_7d BETWEEN 6 AND 10 THEN '6 ~ 10'
            ELSE '11+'
        END AS action_count_range
    FROM action_count
)
SELECT
    action,
    action_count_range,
    COUNT(*) AS users,
    AVG(retained_28d) * 100 AS retention_28d_rate
FROM bucketed
GROUP BY action, action_count_range
ORDER BY action, action_count_range;
```

![](image/0406_04.png)

### 2-5 사용 일수에 따른 정착률 집계하기 

<!-- 이 부분을 지우고 새롭게 배운 내용을 자유롭게 정리해주세요. -->

```sql
WITH action_days AS (
    SELECT
        u.user_id,
        DATE(u.register_date) AS register_day,
        DATE(a.stamp) AS action_day,
        DATEDIFF(DATE(a.stamp), DATE(u.register_date)) AS diff_day
    FROM mst_users u
    LEFT JOIN action_log a
        ON u.user_id = a.user_id
),
usage_summary AS (
    SELECT
        user_id,
        COUNT(DISTINCT CASE WHEN diff_day BETWEEN 1 AND 7 THEN action_day END) AS used_days_in_7d,
        MAX(CASE WHEN diff_day BETWEEN 21 AND 27 THEN 1 ELSE 0 END) AS retained_28d
    FROM action_days
    GROUP BY user_id
)
SELECT
    used_days_in_7d,
    COUNT(*) AS users,
    AVG(retained_28d) * 100 AS retention_28d_rate
FROM usage_summary
GROUP BY used_days_in_7d
ORDER BY used_days_in_7d;
```

![](image/0406_05.png)

### 2-6 사용자의 잔존율 집계하기 

<!-- 이 부분을 지우고 새롭게 배운 내용을 자유롭게 정리해주세요. -->

```sql
WITH action_with_register AS (
    SELECT
        u.user_id,
        DATE(u.register_date) AS register_day,
        DATEDIFF(DATE(a.stamp), DATE(u.register_date)) AS diff_day
    FROM mst_users u
    LEFT JOIN action_log a
        ON u.user_id = a.user_id
),
retention_base AS (
    SELECT
        register_day,
        user_id,
        diff_day
    FROM action_with_register
    WHERE diff_day BETWEEN 0 AND 30
),
cohort_size AS (
    SELECT
        register_day,
        COUNT(DISTINCT user_id) AS cohort_users
    FROM retention_base
    WHERE diff_day = 0
    GROUP BY register_day
),
retention_calc AS (
    SELECT
        register_day,
        diff_day,
        COUNT(DISTINCT user_id) AS active_users
    FROM retention_base
    GROUP BY register_day, diff_day
)
SELECT
    r.register_day,
    r.diff_day,
    c.cohort_users,
    r.active_users,
    ROUND(r.active_users / c.cohort_users * 100, 2) AS retention_rate
FROM retention_calc r
JOIN cohort_size c
    ON r.register_day = c.register_day
ORDER BY r.register_day, r.diff_day;
```

![](image/0406_06.png)

### 2-7 방문 빈도를 기반으로 사용자 속성을 정의하고 집계하기

<!-- 이 부분을 지우고 새롭게 배운 내용을 자유롭게 정리해주세요. -->

```sql
WITH visit_summary AS (
    SELECT
        user_id,
        COUNT(DISTINCT DATE(stamp)) AS visit_days,
        COUNT(*) AS total_actions
    FROM action_log
    GROUP BY user_id
),
user_segment AS (
    SELECT
        user_id,
        visit_days,
        total_actions,
        CASE
            WHEN visit_days >= 20 THEN 'heavy_user'
            WHEN visit_days >= 10 THEN 'middle_user'
            WHEN visit_days >= 3 THEN 'light_user'
            ELSE 'new_or_low_user'
        END AS user_type
    FROM visit_summary
)
SELECT
    user_type,
    COUNT(*) AS users,
    AVG(visit_days) AS avg_visit_days,
    AVG(total_actions) AS avg_total_actions
FROM user_segment
GROUP BY user_type
ORDER BY users DESC;
```

![](image/0406_07.png)

### 2-8 방문 종류를 기반으로 성장지수 집계하기 

<!-- 이 부분을 지우고 새롭게 배운 내용을 자유롭게 정리해주세요. -->

```sql
WITH action_summary AS (
    SELECT
        user_id,
        SUM(CASE WHEN action = 'view' THEN 1 ELSE 0 END) AS view_cnt,
        SUM(CASE WHEN action = 'comment' THEN 1 ELSE 0 END) AS comment_cnt,
        SUM(CASE WHEN action = 'follow' THEN 1 ELSE 0 END) AS follow_cnt,
        SUM(CASE WHEN action = 'purchase' THEN 1 ELSE 0 END) AS purchase_cnt
    FROM action_log
    GROUP BY user_id
),
growth_index AS (
    SELECT
        user_id,
        view_cnt,
        comment_cnt,
        follow_cnt,
        purchase_cnt,
        (view_cnt * 1)
        + (comment_cnt * 3)
        + (follow_cnt * 2)
        + (purchase_cnt * 5) AS growth_score
    FROM action_summary
)
SELECT
    user_id,
    view_cnt,
    comment_cnt,
    follow_cnt,
    purchase_cnt,
    growth_score
FROM growth_index
ORDER BY growth_score DESC;
```

![](image/0406_08.png)

### 2-9 지표 개선 방법 익히기 

<!-- 이 부분을 지우고 새롭게 배운 내용을 자유롭게 정리해주세요. -->

```sql
WITH register_users AS (
    SELECT
        user_id,
        DATE(register_date) AS register_day
    FROM mst_users
),
action_summary AS (
    SELECT
        r.user_id,
        r.register_day,
        COUNT(a.stamp) AS total_actions,
        COUNT(DISTINCT DATE(a.stamp)) AS active_days,
        MAX(CASE WHEN DATEDIFF(DATE(a.stamp), r.register_day) = 1 THEN 1 ELSE 0 END) AS repeat_1d,
        MAX(CASE WHEN DATEDIFF(DATE(a.stamp), r.register_day) BETWEEN 21 AND 27 THEN 1 ELSE 0 END) AS retention_28d
    FROM register_users r
    LEFT JOIN action_log a
        ON r.user_id = a.user_id
    GROUP BY r.user_id, r.register_day
)
SELECT
    register_day,
    COUNT(*) AS registered_users,
    AVG(total_actions) AS avg_total_actions,
    AVG(active_days) AS avg_active_days,
    AVG(repeat_1d) * 100 AS repeat_1d_rate,
    AVG(retention_28d) * 100 AS retention_28d_rate
FROM action_summary
GROUP BY register_day
ORDER BY register_day;
```

![](image/0406_09.png)


## 3. 시계열에 따른 사용자의 개별적인 행동 분석하기 

### 3-1 사용자의 액션 간격 집계하기

<!-- 이 부분을 지우고 새롭게 배운 내용을 자유롭게 정리해주세요. -->

```sql
WITH ordered_actions AS (
    SELECT
        user_id,
        stamp,
        LAG(stamp) OVER (
            PARTITION BY user_id
            ORDER BY stamp
        ) AS prev_stamp
    FROM action_log
)
SELECT
    user_id,
    stamp,
    prev_stamp,
    TIMESTAMPDIFF(HOUR, prev_stamp, stamp) AS diff_hours
FROM ordered_actions
WHERE prev_stamp IS NOT NULL
ORDER BY user_id, stamp;
```

![](image/0406_10.png)

### 3-2 카트 추가 후에 구매했는지 파악하기 

<!-- 이 부분을 지우고 새롭게 배운 내용을 자유롭게 정리해주세요. -->

```sql
WITH cart_action AS (
    SELECT
        user_id,
        product_id,
        MIN(stamp) AS cart_time
    FROM action_log
    WHERE action = 'add_cart'
    GROUP BY user_id, product_id
),
buy_action AS (
    SELECT
        user_id,
        product_id,
        MIN(stamp) AS buy_time
    FROM action_log
    WHERE action = 'purchase'
    GROUP BY user_id, product_id
)
SELECT
    c.user_id,
    c.product_id,
    c.cart_time,
    b.buy_time,
    CASE
        WHEN b.buy_time IS NOT NULL
             AND b.buy_time >= c.cart_time THEN 1
        ELSE 0
    END AS purchased_after_cart
FROM cart_action c
LEFT JOIN buy_action b
    ON c.user_id = b.user_id
   AND c.product_id = b.product_id
ORDER BY c.user_id, c.product_id;
```

![](image/0406_10.png)

### 3-3 등록으로부터의 매출을 날짜별로 집계하기 

<!-- 이 부분을 지우고 새롭게 배운 내용을 자유롭게 정리해주세요. -->

```sql
WITH cart_action AS (
    SELECT
        user_id,
        product_id,
        MIN(stamp) AS cart_time
    FROM action_log
    WHERE action = 'add_cart'
    GROUP BY user_id, product_id
),
buy_action AS (
    SELECT
        user_id,
        product_id,
        MIN(stamp) AS buy_time
    FROM action_log
    WHERE action = 'purchase'
    GROUP BY user_id, product_id
)
SELECT
    c.user_id,
    c.product_id,
    c.cart_time,
    b.buy_time,
    CASE
        WHEN b.buy_time IS NOT NULL
             AND b.buy_time >= c.cart_time THEN 1
        ELSE 0
    END AS purchased_after_cart
FROM cart_action c
LEFT JOIN buy_action b
    ON c.user_id = b.user_id
   AND c.product_id = b.product_id
ORDER BY c.user_id, c.product_id;
```

![](image/0406_10.png)



### 🎉 수고하셨습니다.
