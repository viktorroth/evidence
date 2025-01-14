# Home

## Context

```summary
    select 
        count(*) as total_calls,
        countif(timestamp_diff(current_timestamp(), created_date, day) <7) as calls_in_the_last_7_days,
        countif(timestamp_diff(current_timestamp(), created_date, day) <365) as calls_in_the_last_365_days,
        min(created_date) as earliest_call_date, 
        max(created_date) as latest_call_date,
        "string1" as string1,
    from `bigquery-public-data.austin_311.311_service_requests` 
    limit 1 
```

```rolling_average_daily_calls
    select *, 
        avg(number_of_complaints) over(order by date rows between 7 preceding and current row) as seven_day_average
    from ${daily_complaints}
    limit 50
```

<Chart data={data.rolling_average_daily_calls} x=date title={`Last ${data.rolling_average_daily_calls.length} days of call volume`}>
    <Bar y=number_of_complaints/> 
    <Line y=seven_day_average/>
</Chart>




Austin 311 has fielded <Value data={data.summary}/> calls since <Value data={data.summary} column=earliest_call_date/> and <Value data={data.summary} column=calls_in_the_last_365_days/> calls over the last 365 days.

<LineChart data={data.daily_complaints} x='date' y='number_of_complaints' units="calls to Austin 311 per day"/>

```daily_complaints
    select 
        extract(date from created_date) as date, 
        count(*) as number_of_complaints 
    from `bigquery-public-data.austin_311.311_service_requests` 
    group by 1 
    order by 1 desc
    limit 365
```

```monthly_complaints
    select date_trunc(date, month) as month,
    sum(number_of_complaints) as complaints
    from ${daily_complaints}
    group by month
    order by month asc
```

```annual_complaints
    select date_trunc(month, year) as year,
    sum(complaints) as complaints
    from ${ monthly_complaints }
    group by year
```


<LineChart data={data.monthly_complaints}/>

Call data is updated every few days -- the most recent update was on <Value data={data.summary} column=latest_call_date/>. 

## Top {data.top_complaint.length} Call Categories 
The following {data.top_complaint.length} categories have generated the most calls since 2014:

```top_complaint 
    with total as (
        select 
            count(*) as complaints 
        from `bigquery-public-data.austin_311.311_service_requests` 
    ), 
    top_complaints as (
        select 
            complaint_description as description, 
            count(*) as number_of_complaints
        from `bigquery-public-data.austin_311.311_service_requests` 
        group by 1
        order by 2 desc 
        limit 4
    )

    select *,
    number_of_complaints/total.complaints as complaints_pct  
    from top_complaints 
    left join total on true
```

<ol>
{#each data.top_complaint as complaint}

<li> {complaint.description}: <Value value={complaint.number_of_complaints}/> calls (<Value value={complaint.complaints_pct} fmt="pct"/> of all calls)</li>

{/each}
</ol>

```top_categories_weekly
with     

top_complaints as (
    select 
        complaint_description as description, 
        count(*) as number_of_complaints
    from `bigquery-public-data.austin_311.311_service_requests` 
    group by 1
    order by 2 desc 
    limit 16
)

select 
    date_trunc(created_date, week) as date, 
    complaint_description as description,
    count(*) as number_of_complaints 
from `bigquery-public-data.austin_311.311_service_requests` 
where complaint_description in (select description from  top_complaints)
and extract(year from created_date) >= extract(year from current_date()) - 2
group by 1,2
```

<LineChart title="Weekly Call Volume, by Category" data={data.top_categories_weekly} x=date y=number_of_complaints series=description/>

``` ytd_volume 
with annual_call_volume as (
    select 
        extract(year from created_date) as year,
        count(*) as vol
    from `bigquery-public-data.austin_311.311_service_requests`
    where extract(dayofyear from created_date) <= extract(dayofyear from current_date())
    and extract(year from created_date) >= extract(year from current_date()) - 3 
    group by 1 
)

select 
    safe_divide(vol, lag(vol) over(order by year)) - 1 as growth_pct,
    *   
from annual_call_volume
order by year desc

```

<LineChart data={data.daily_vol_yoy} x=day_of_year y=cum_vol series=year 
yAxisTitle="cumulative calls" 
xAxisTitle="day of year"/>


```daily_vol
 select 
        extract(year from created_date) as year,
        extract(dayofyear from created_date) as day_of_year,
        count(*) as vol
    from `bigquery-public-data.austin_311.311_service_requests`
    where extract(year from created_date) >= extract(year from current_date()) - 2 
    group by 1,2
```


```daily_vol_yoy
    select 
        *, 
        sum(vol) over(partition by year order by day_of_year) as cum_vol
    from ${daily_vol}
```

```daily_volume_yoy
with daily_vol as (
        select 
        extract(year from created_date) as year,
        extract(dayofyear from created_date) as day_of_year,
        count(*) as vol
    from `bigquery-public-data.austin_311.311_service_requests`
    where extract(year from created_date) >= extract(year from current_date()) - 2 
    group by 1,2)

select 
    *, 
    sum(vol) over(partition by year order by day_of_year) as cum_vol
from daily_vol

```

# Recent Call Volume Spikes  
The following [volume spikes](spikes) may warrant further investigation.

```daily_complaints_by_category
        select 
            complaint_description as description,
            extract(date from created_date) as date, 
            count(*) as number_of_complaints 
        from `bigquery-public-data.austin_311.311_service_requests` 
        group by 1,2 
```



```spikes
    with daily_complaints_by_category as ${daily_complaints_by_category}, 
    rolling_metrics as (
        select *,
            stddev_pop(number_of_complaints) over(partition by description order by date rows between 365 preceding and 1 preceding) as rolling_stddev_daily_complaints,
            avg(number_of_complaints) over(partition by description order by date rows between 365 preceding and 1 preceding) as rolling_avg_daily_complaints
        from daily_complaints_by_category 
    ), 
    spikes as (
        select *,
        from rolling_metrics
        where 
        number_of_complaints > rolling_avg_daily_complaints + 5*rolling_stddev_daily_complaints
        and number_of_complaints > 50
        order by date desc
    )

    select * ,
    row_number() over() as spike_id,
    from spikes
    limit 3
```


{#each data.spikes as spike}

## {spike.description}
[{spike.number_of_complaints} calls on <Value value={spike.date} fmt="date"/> &rarr;](/spikes/{spike.spike_id}) 

<LineChart data={data.daily_complaints_by_category.filter(d => d.description === spike.description)} x="date" y="number_of_complaints" units={spike.description + " Calls"}/>

{/each}
