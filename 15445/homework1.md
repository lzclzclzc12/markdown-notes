## 犯错记录

- ```
  select name , 
  case   -- 条件判断语句
    when died is not null
    then died - born
    else 2022 - born
  end as age
  from people
  where born >= 1900
  order by age desc , name  -- 这里必须是按age排序，而不是按照born排序，因为查询结果里没有born
  limit 20;
  ```

- ```
  -- 要求统计每10年的平均数据，那么就是先把年份搞成统一样式，比如1993和1995都搞成1990，然后按照年份group by
  select 
  case 
    when premiered is not null
    then cast(premiered / 10 * 10 as text) || 's'
  end as decade,
  round(avg(rating) , 2) as avg_ratings,
  max(rating) as top_ratings,
  min(rating) as min_rating,
  count(*) as num_releases
  from titles, ratings
  where premiered is not null and titles.title_id = ratings.title_id
  group by decade
  order by avg_ratings desc, decade;
  ```

- ```
  -- 直接将4个表连起来的话，很费时间，所以先让crew 和 people表连接，并且搞一个只有一列的视图
  WITH cruise_crew AS (
    select crew.title_id as title_id
    from crew , people
    where people.name like '%Cruise%' and people.born = 1962 and crew.person_id = people.person_id
  )
  select primary_title , votes
  from ratings , titles , cruise_crew
  where titles.title_id = ratings.title_id and titles.title_id = cruise_crew.title_id
  order by votes desc
  limit 10;
  ```

- 