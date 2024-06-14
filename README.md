# ad_metrics
**Facebook and Google Ad Campaigns Analysis**
**Task**. Combining data from two tables (Facebook and Google add campaigns) in the PostgreSQL database, identify:
  - Campaign name (utm_campaign) from url_parameters string
  - Calculate the next metrics for the campaigns: CTR, CPC, CPM, ROMI
  - Find differences for ‘CPM, CTR, ROMI’ in the current month compared to the previous one in percentage terms


with fgc as ( -- CTE #1 United Facebook and Google campaigns from two tables. Fill in missing values with 0 using COALESCE function
      select
          ad_date,
          url_parameters,
          coalesce (spend, 0) as spend,
          coalesce (impressions, 0) as impressions,
          coalesce (reach, 0) as reach,
          coalesce (clicks, 0) as clicks,
          coalesce (leads, 0) as leads,
          coalesce (value, 0) as value
        from facebook_ads_basic_daily
    union
       select
          ad_date,
          url_parameters,
          coalesce (spend, 0) as spend,
          coalesce (impressions, 0) as impressions,
          coalesce (reach, 0) as reach,
          coalesce (clicks, 0) as clicks,
          coalesce (leads, 0) as leads,
          coalesce (value, 0) as value
       from google_ads_basic_daily
       order by 1
),
metrics as (     -- CTE #2 Metrics calculation CTR, CPC, CPM, ROMI
      select -- query to add CTR, CPC, CPM, ROMI grouped by month
          date(date_trunc ('month', ad_date)) as add_month, -- sorting data by month
          case lower(substring(url_parameters, 'utm_campaign=([^&#$]+)')) -- getting campaign name from URL using regexp
                when 'nan' then null
                else lower(substring(url_parameters, 'utm_campaign=([^&#$]+)')) end as utm_campaign,
          sum(spend) as spend,
          sum(impressions) as impressions,
          sum (clicks) as clicks,
          sum(value) as value,
          avg(case impressions -- escape division by zero error using 'case' function
                when 0 then null
                else round(clicks/impressions::numeric,2) end) as CTR,
          avg(case clicks
                when 0 then null
                else round(spend/clicks::numeric,2) end) as CPC,
          avg(case clicks
                when 0 then null
                else round(spend/clicks::numeric,2)*1000 end) as CPM,
          avg(case spend
                when 0 then null
                else round((value-spend)/spend::numeric,2) end) as ROMI
      from fgc
      group by 1,2
      order by 1
),
shifted as ( -- CTE #3 with self-joined CTE shifted by 1 month
      select
          m.add_month,
          case
              when m.utm_campaign like '%\%%' then decode_cyrillic_url(m.utm_campaign) -- decoding cyrillic campaigns
              else m.utm_campaign end as utm_campaign,
          m3.utm_campaign as utm_campaign_prev,
          m.spend,
          m.impressions,
          m.clicks,
          m.value,
          m.cpc,
          m.cpm,
          m.ctr,
          m.romi,
          m3.cpm as prev_cpm,
          m3.ctr as prev_ctr,
          m3.romi as prev_romi
      from metrics m
      left join metrics as m3 on m.add_month=m3.add_month + interval '1 month' 
          and m.utm_campaign=m3.utm_campaign -- self-join with 1 month shift to calculate the previous month metrics
      where m.utm_campaign is not null
)
select
      add_month,
      utm_campaign,
      spend/100 as spend, -- converting cents into dollars
      impressions,
      clicks,
      value/100 as value, -- converting cents into dollars
      round(cpc/100,2) as cpc, -- converting cents into dollars
      round(cpm/100,2) as cpm, -- converting cents into dollars
      round(ctr,2) as ctr,
      round(romi,2) as romi,
      case
          when cpm >0 then round(cpm::numeric /prev_cpm-1, 2)*100
          when cpm=0 and prev_cpm>0 then -100
          end as "cpm_diff_%",
      case
          when prev_ctr >0 then round(ctr::numeric /prev_ctr-1, 2)*100
          when ctr=0 and prev_ctr>0 then -100
          end as "ctr_diff_%",
      case
          when prev_romi >0 then round(romi::numeric /prev_romi-1, 2)*100
          when romi=0 and prev_romi>0 then -100
          end as "romi_diff_%"
from shifted
