# Домашнее задание к занятию 12.5. «Индексы» - Ковбаса Анна

### Задание 1

Напишите запрос к учебной базе данных, который вернёт процентное отношение общего размера всех индексов к общему размеру всех таблиц.
```sql
SELECT  SUM(INDEX_LENGTH) / SUM(DATA_LENGTH) * 100 AS 'index/data ratio'
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'sakila';
```
![1-1](https://github.com/kovbasaad/12-5-homework/blob/main/img/1-1.JPG)

### Задание 2

Выполните explain analyze следующего запроса:
```sql
select distinct concat(c.last_name, ' ', c.first_name), sum(p.amount) over (partition by c.customer_id, f.title)
from payment p, rental r, customer c, inventory i, film f
where date(p.payment_date) = '2005-07-30' and p.payment_date = r.rental_date and r.customer_id = c.customer_id and i.inventory_id = r.inventory_id
```
```sql
-> Limit: 200 row(s)  (cost=0.00..0.00 rows=0) (actual time=16796.622..16796.658 rows=200 loops=1)
    -> Table scan on <temporary>  (cost=2.50..2.50 rows=0) (actual time=16796.620..16796.647 rows=200 loops=1)
        -> Temporary table with deduplication  (cost=0.00..0.00 rows=0) (actual time=16796.618..16796.618 rows=391 loops=1)
            -> Window aggregate with buffering: sum(payment.amount) OVER (PARTITION BY c.customer_id,f.title )   (actual time=7037.371..16073.893 rows=642000 loops=1)
                -> Sort: c.customer_id, f.title  (actual time=7037.330..7245.002 rows=642000 loops=1)
                    -> Stream results  (cost=22211566.90 rows=16700349) (actual time=0.513..4659.977 rows=642000 loops=1)
                        -> Nested loop inner join  (cost=22211566.90 rows=16700349) (actual time=0.506..3895.722 rows=642000 loops=1)
                            -> Nested loop inner join  (cost=20537356.88 rows=16700349) (actual time=0.503..3550.929 rows=642000 loops=1)
                                -> Nested loop inner join  (cost=18863146.85 rows=16700349) (actual time=0.496..3174.371 rows=642000 loops=1)
                                    -> Inner hash join (no condition)  (cost=1608783.80 rows=16086000) (actual time=0.480..130.467 rows=634000 loops=1)
                                        -> Filter: (cast(p.payment_date as date) = '2005-07-30')  (cost=1.68 rows=16086) (actual time=0.037..21.878 rows=634 loops=1)
                                            -> Table scan on p  (cost=1.68 rows=16086) (actual time=0.026..9.609 rows=16044 loops=1)
                                        -> Hash
                                            -> Covering index scan on f using idx_title  (cost=112.00 rows=1000) (actual time=0.045..0.366 rows=1000 loops=1)
                                    -> Covering index lookup on r using rental_date (rental_date=p.payment_date)  (cost=0.97 rows=1) (actual time=0.003..0.004 rows=1 loops=634000)
                                -> Single-row index lookup on c using PRIMARY (customer_id=r.customer_id)  (cost=0.00 rows=1) (actual time=0.000..0.000 rows=1 loops=642000)
                            -> Single-row covering index lookup on i using PRIMARY (inventory_id=r.inventory_id)  (cost=0.00 rows=1) (actual time=0.000..0.000 rows=1 loops=642000)
```

- перечислите узкие места:
  поиск выполняется по всем данным многих таблиц, обращение к таблицам, данные из которых не используются в запросе
- оптимизируйте запрос: внесите корректировки по использованию операторов, при необходимости добавьте индексы.

```sql
select distinct concat(c.last_name, ' ', c.first_name), sum(p.amount) over (partition by c.customer_id)
from payment p 
join rental r on r.rental_id = p.rental_id 
join customer c on c.customer_id = p.customer_id 
group by c.customer_id, p.amount, p.payment_date
having date(p.payment_date) = '2005-07-30';
```

```sql
-> Limit: 200 row(s)  (cost=0.00..0.00 rows=0) (actual time=500.656..500.692 rows=200 loops=1)
    -> Table scan on <temporary>  (cost=2.50..2.50 rows=0) (actual time=500.654..500.682 rows=200 loops=1)
        -> Temporary table with deduplication  (cost=0.00..0.00 rows=0) (actual time=500.652..500.652 rows=391 loops=1)
            -> Window aggregate with buffering: sum(payment.amount) OVER (PARTITION BY c.customer_id )   (actual time=487.193..500.169 rows=634 loops=1)
                -> Filter: (cast(p.payment_date as date) = '2005-07-30')  (actual time=487.121..497.863 rows=634 loops=1)
                    -> Sort: c.customer_id  (actual time=487.107..496.357 rows=16043 loops=1)
                        -> Table scan on <temporary>  (cost=14908.50..15116.75 rows=16462) (actual time=468.847..470.971 rows=16043 loops=1)
                            -> Temporary table with deduplication  (cost=14908.49..14908.49 rows=16462) (actual time=468.844..468.844 rows=16043 loops=1)
                                -> Nested loop inner join  (cost=13262.29 rows=16462) (actual time=0.080..384.484 rows=16044 loops=1)
                                    -> Nested loop inner join  (cost=7500.59 rows=16462) (actual time=0.071..243.558 rows=16044 loops=1)
                                        -> Covering index scan on r using idx_fk_staff_id  (cost=1738.90 rows=16419) (actual time=0.043..15.446 rows=16044 loops=1)
                                        -> Index lookup on p using fk_payment_rental (rental_id=r.rental_id)  (cost=0.25 rows=1) (actual time=0.009..0.013 rows=1 loops=16044)
                                    -> Single-row index lookup on c using PRIMARY (customer_id=p.customer_id)  (cost=0.25 rows=1) (actual time=0.009..0.009 rows=1 loops=16044)
```
