# Famous-Painting-Analysis-
This project made by using postgres SQL query . This Tool help me to get important insights from the large dataset.

select * from product_size
select * from canvas_size
select * from work
select * from museum_hours
select * from museum
select * from artist
select * from image_link
select * from subject


1.Fetch all the paintings which are not displayed on any museums?
   select * from work where museum_id is null

2.Are there museums without any paintings?
  select * from museum m
  where not exists (select museum_id from work w
		where w.museum_id = m.museum_id)
 
3.Are there museums without any paintings?
  select count(work_id) from product_size p
  where p.sale_price > p.regular_price
  
4. Identify the paintings whose asking price is less than 50% of its regular price
  select count(work_id) from product_size p
  where p.sale_price < (p.regular_price*0.5)  
  
5. Which canva size costs the most?
  select cs.label as canva, ps.sale_price
	from (select *
		  , rank() over(order by sale_price desc) as rnk 
		  from product_size) ps
	join canvas_size cs on cs.size_id::text=ps.size_id
	where ps.rnk=1;					 

6. Delete duplicate records from work, product_size, subject and image_link tables
  delete from work 
	where work_id  in (select work_id from work
	group by work_id having count(*)>1)

  delete from product_size
	where work_id  in (select work_id from product_size
	group by work_id ,size_id having count(*)>1)

  delete from image_link
	where work_id  in (select work_id from image_link
	group by work_id ,url having count(*)>1)

  delete from subject
	where work_id  in (select work_id from subject
	group by work_id ,subject having count(*)>1)

7.Identify the museums with invalid city information in the given dataset?
    select * from museum 
	where city ~ '^[0-9]'
	
8. Museum_Hours table has 1 invalid entry. Identify it and remove it.
   delete from museum_hours 
	where museum_id not in (select min(museum_id)
    from museum_hours
	group by museum_id, day)
9.Fetch the top 10 most famous painting subject?
  select * from 
  (select s.subject,count(w.work_id) as num , rank()over(order by count(w.work_id) desc) as rnk
  from work w
  join subject s on s.work_id = w.work_id
  group by s.subject) as a
  where a.rnk <=10

10. Identify the museums which are open on both Sunday and Monday. Display
museum name, city.
   select distinct m.name as museum_name,m.city from museum_hours mh
   join museum m on m.museum_id = mh.museum_id
   where day in ('Sunday','Monday')

11. How many museums are open every single day?
  select count(1) from
  (select museum_id,count(day)as single_day_count from museum_hours
   group by museum_id having count(day) = 7) x
   
12. Which are the top 5 most popular museum? (Popularity is defined based on most
no of paintings in a museum)?
select * from  
(select m.name,count(w.work_id) ,rank() over(order by count(w.work_id) desc) as rank
from museum m
join work w on w.museum_id = m.museum_id
group by m.name) z
where rank<=5

13.Who are the top 5 most popular artist? (Popularity is defined based on most no of
paintings done by an artist).
  select * from  
(select a.full_name,count(w.work_id) ,rank() over(order by count(w.work_id) desc) as rank
from artist a
join work w on w.artist_id = a.artist_id
group by a.full_name) z
where rank<=5

14. Display the 3 least popular canva sizes?
   
select *
	from (
		select cs.size_id,cs.label,count(w.work_id) as no_of_paintings
		, dense_rank() over(order by count(w.work_id) ) as ranking
		from work w
		join product_size ps on ps.work_id=w.work_id
		join canvas_size cs on cs.size_id::text = ps.size_id
		group by cs.size_id,cs.label) x
	where x.ranking<=3;
	
15. Which museum is open for the longest during a day. Dispay museum name, state
and hours open and which day?
  select museum_name,state ,day, open, close, duration
	from (	select m.name as museum_name, m.state, day, open, close
			, to_timestamp(open,'HH:MI AM') 
			, to_timestamp(close,'HH:MI PM') 
			, to_timestamp(close,'HH:MI PM') - to_timestamp(open,'HH:MI AM') as duration
			, rank() over (order by (to_timestamp(close,'HH:MI PM') - to_timestamp(open,'HH:MI AM')) desc) as rnk
			from museum_hours mh
		 	join museum m on m.museum_id=mh.museum_id) x
	where x.rnk=1;
	
16. Which museum has the most no of most popular painting style?
 with pop_style as 
			(select style
			,rank() over(order by count(work_id) desc) as rnk
			from work
			group by style),
		cte as
			(select w.museum_id,m.name as museum_name ,ps.style, count(work_id) as no_of_paintings
			,rank() over(order by count(work_id) desc) as rnk
			from work w
			join museum m on m.museum_id=w.museum_id
			join pop_style ps on ps.style = w.style
			where w.museum_id is not null
			and ps.rnk=1
			group by w.museum_id, m.name,ps.style)
	select museum_name,style,no_of_paintings
	from cte 
	where rnk=1;
	
17.Identify the artists whose paintings are displayed in multiple countries.
   with cte as
   (select w.work_id as paintings,m.country ,a.full_name as artist from work w
	join museum m on m.museum_id = w.museum_id
	join artist a on a.artist_id = w.artist_id 
	 )
 select artist,count(country)  as multi_country
 from cte
 group by artist
 having count(country)>1
 order by multi_country desc
 
18. Display the country and the city with most no of museums. Output 2 seperate
columns to mention the city and country. If there are multiple value, seperate them
with comma?
   with cte_country as 
			(select country, count(museum_id)
			, rank() over(order by count(museum_id) desc) as rnk
			from museum
			group by country),
		cte_city as
			(select city, count(museum_id)
			, rank() over(order by count(museum_id) desc) as rnk
			from museum
			group by city)
	select string_agg(distinct c.country,', '), string_agg(cy.city,', ')
	from cte_country c
	cross join cte_city cy
	where c.rnk = 1 and cy.rnk = 1;
	
19. Identify the artist and the museum where the most expensive and least expensive
painting is placed. Display the artist name, sale_price, painting name, museum
name, museum city and canvas label?
   
 with cte as
(select * ,rank()over(order by sale_price desc) rank_desc, 
 rank()over(order by sale_price asc) rank_asc
  from product_size)
  select w.name as painting,cte.sale_price,a.full_name as artist,m.name,m.city,cv.label from cte
  join work w on w.work_id=cte.work_id
	join museum m on m.museum_id=w.museum_id
	join artist a on a.artist_id=w.artist_id
	join canvas_size cv on cv.size_id = cte.size_id::NUMERIC
	where rank_desc=1 or rank_asc=1;
	
20. Which country has the 5th highest no of paintings?
with cte as 
		(select m.country, count(work_id) as no_of_Paintings
		, rank() over(order by count(work_id) desc) as rnk
		from work w
		join museum m on m.museum_id=w.museum_id
		group by m.country)
	select country, no_of_Paintings
	from cte 
	where rnk=5;
	
21. Which are the 3 most popular and 3 least popular painting styles?
    with cte as 
		(select style, count(work_id) as cnt
		, rank() over(order by count(work_id) desc) rnk
		, count(work_id) over() as no_of_records
		from work
		where style is not null
		group by style)
	select style
	, case when rnk <=3 then 'Most Popular' else 'Least Popular' end as remarks 
	from cte
	where rnk <=3 or rnk > no_of_records - 3
  
 22. Which artist has the most no of Portraits paintings outside USA?. Display artist
name, no of paintings and the artist nationality.
   select full_name as artist_name, nationality, no_of_paintings
	from (
		select a.full_name, a.nationality
		,count(w.work_id) as no_of_paintings
		,rank() over(order by count(w.work_id) desc) as rnk
		from work w
		join artist a on a.artist_id=w.artist_id
		join subject s on s.work_id=w.work_id
		join museum m on m.museum_id=w.museum_id
		where s.subject='Portraits'
		and m.country != 'USA'
		group by a.full_name, a.nationality) x
	where rnk=1;	
 
  
  
  
  
  
  
  
  
  





