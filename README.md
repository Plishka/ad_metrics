# ad_metrics

## **Facebook and Google Ad Campaigns Analysis**

Combining data from two tables (Facebook and Google ad campaigns) in the PostgreSQL database, the analysis aims to:
- Extract the campaign name (`utm_campaign`) from the `url_parameters` string.
- Decode Cyrillic characters in `utm_campaign`.
- Calculate the following metrics for the campaigns: CTR (Click-Through Rate), CPC (Cost Per Click), CPM (Cost Per Thousand Impressions), ROMI (Return on Marketing Investment).
- Determine the differences for `CPM`, `CTR`, and `ROMI` in the current month compared to the previous month in percentage terms.
- Build an interactive Dashboard using Looker Studio to display the results.

### **Advanced Functions and Approaches Used**
- **CTEs (Common Table Expressions)**: To structure the query and make it more readable.
- **COALESCE Function**: To handle missing values by replacing them with `0`.
- **Substring and Case Functions**: To extract and handle the `utm_campaign` from `url_parameters`.
- **Custom Function**: `decode_cyrillic_url` to decode Cyrillic characters.
- **Aggregation and Grouping**: To calculate the metrics by month.
- **Self-Join**: To shift the metrics by one month for comparison.
- **Conditional Logic**: To avoid division by zero and handle edge cases in percentage change calculations.

### Dashboard

Click the image below to view the interactive Dashboard on Looker Studio website.

[![Revenue Metrics Dashboard](https://github.com/Plishka/ad_metrics/blob/main/Ad_campaigns.png?raw=true)](https://lookerstudio.google.com/reporting/53f08051-f147-4132-88cd-cf997aeb9770/page/rfDeD)


The results of the analysis are visualized using an interactive dashboard built with Looker Studio. The dashboard displays:
- Monthly metrics (CTR, CPC, CPM, ROMI) for each campaign.
- The percentage change in `CPM`, `CTR`, and `ROMI` compared to the previous month.

---

### **Steps**

1. **Create Temporary Function**
   - **Function**: `decode_cyrillic_url`
   - **Purpose**: Decodes Cyrillic characters in the URL parameters to make the campaign names readable.
   - **Code**:
     ```sql
     create or replace function public.decode_cyrillic_url(url_encoded_text text)
     returns text
     language plpgsql
     as $function$
     declare
     decoded_text TEXT := upper(url_encoded_text);
     begin
       decoded_text := REPLACE(decoded_text, '%D0%87', 'ї'); -- ї
       decoded_text := REPLACE(decoded_text, '%D0%B0', 'а'); -- а
       decoded_text := REPLACE(decoded_text, '%D0%B1', 'б'); -- б
       decoded_text := REPLACE(decoded_text, '%D0%B2', 'в'); -- в
       decoded_text := REPLACE(decoded_text, '%D0%B3', 'г'); -- г
       decoded_text := REPLACE(decoded_text, '%D0%B4', 'д'); -- д
       decoded_text := REPLACE(decoded_text, '%D0%B5', 'е'); -- е
       decoded_text := REPLACE(decoded_text, '%D0%B6', 'ж'); -- ж
       decoded_text := REPLACE(decoded_text, '%D0%B7', 'з'); -- з
       decoded_text := REPLACE(decoded_text, '%D0%B8', 'и'); -- и
       decoded_text := REPLACE(decoded_text, '%D0%B9', 'й'); -- й
       decoded_text := REPLACE(decoded_text, '%D0%BA', 'к'); -- к
       decoded_text := REPLACE(decoded_text, '%D0%BB', 'л'); -- л
       decoded_text := REPLACE(decoded_text, '%D0%BC', 'м'); -- м
       decoded_text := REPLACE(decoded_text, '%D0%BD', 'н'); -- н
       decoded_text := REPLACE(decoded_text, '%D0%BE', 'о'); -- о
       decoded_text := REPLACE(decoded_text, '%D0%BF', 'п'); -- п
       decoded_text := REPLACE(decoded_text, '%D1%80', 'р'); -- р
       decoded_text := REPLACE(decoded_text, '%D1%81', 'с'); -- с
       decoded_text := REPLACE(decoded_text, '%D1%82', 'т'); -- т
       decoded_text := REPLACE(decoded_text, '%D1%83', 'у'); -- у
       decoded_text := REPLACE(decoded_text, '%D1%84', 'ф'); -- ф
       decoded_text := REPLACE(decoded_text, '%D1%85', 'х'); -- х
       decoded_text := REPLACE(decoded_text, '%D1%86', 'ц'); -- ц
       decoded_text := REPLACE(decoded_text, '%D1%87', 'ч'); -- ч
       decoded_text := REPLACE(decoded_text, '%D1%88', 'ш'); -- ш
       decoded_text := REPLACE(decoded_text, '%D1%89', 'щ'); -- щ
       decoded_text := REPLACE(decoded_text, '%D1%8C', 'ь'); -- ь
       decoded_text := REPLACE(decoded_text, '%D1%8D', 'э'); -- э
       decoded_text := REPLACE(decoded_text, '%D1%8E', 'ю'); -- ю
       decoded_text := REPLACE(decoded_text, '%D1%8F', 'я'); -- я
     return lower(decoded_text);
     end;
     $function$;
     ```

2. **Combine Data from Facebook and Google Ad Campaigns**
   - **Common Table Expression (CTE) #1**: `fgc`
   - **Purpose**: Unite Facebook and Google campaigns from two tables and fill in missing values with `0` using the `COALESCE` function.
   - **Code**:
     ```sql
     with fgc as (
         select
             ad_date,
             url_parameters,
             coalesce(spend, 0) as spend,
             coalesce(impressions, 0) as impressions,
             coalesce(reach, 0) as reach,
             coalesce(clicks, 0) as clicks,
             coalesce(leads, 0) as leads,
             coalesce(value, 0) as value
         from facebook_ads_basic_daily
         union
         select
             ad_date,
             url_parameters,
             coalesce(spend, 0) as spend,
             coalesce(impressions, 0) as impressions,
             coalesce(reach, 0) as reach,
             coalesce(clicks, 0) as clicks,
             coalesce(leads, 0) as leads,
             coalesce(value, 0) as value
         from google_ads_basic_daily
         order by 1
     ),
     ```

3. **Calculate Metrics**
   - **Common Table Expression (CTE) #2**: `metrics`
   - **Purpose**: Calculate metrics such as CTR, CPC, CPM, and ROMI, grouped by month.
   - **Code**:
     ```sql
     metrics as (
         select
             date(date_trunc('month', ad_date)) as add_month,
             case lower(substring(url_parameters, 'utm_campaign=([^&#$]+)'))
                 when 'nan' then null
                 else lower(substring(url_parameters, 'utm_campaign=([^&#$]+)')) end as utm_campaign,
             sum(spend) as spend,
             sum(impressions) as impressions,
             sum(clicks) as clicks,
             sum(value) as value,
             avg(case impressions
                 when 0 then null
                 else round(clicks/impressions::numeric,2) end) as ctr,
             avg(case clicks
                 when 0 then null
                 else round(spend/clicks::numeric,2) end) as cpc,
             avg(case clicks
                 when 0 then null
                 else round(spend/clicks::numeric,2)*1000 end) as cpm,
             avg(case spend
                 when 0 then null
                 else round((value-spend)/spend::numeric,2) end) as romi
         from fgc
         group by 1,2
     ),
     ```

4. **Shift Metrics by One Month**
   - **Common Table Expression (CTE) #3**: `shifted`
   - **Purpose**: Self-join the metrics table with a one-month shift to calculate the previous month's metrics.
   - **Code**:
     ```sql
     shifted as (
         select
             m.add_month,
             case
                 when m.utm_campaign like '%\%%' then decode_cyrillic_url(m.utm_campaign)
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
         left join metrics as m3 on m.add_month = m3.add_month + interval '1 month'
             and m.utm_campaign = m3.utm_campaign
         where m.utm_campaign is not null
     )
     ```

5. **Final Query to Calculate Monthly Differences**
   - **Purpose**: Calculate the differences in `CPM`, `CTR`, and `ROMI` for the current month compared to the previous month in percentage terms, and convert values from cents to dollars where applicable.
   - **Code**:
     ```sql
     select
         add_month,
         utm_campaign,
         spend/100 as spend,
         impressions,
         clicks,
         value/100 as value,
         round(cpc/100, 2) as cpc,
         round(cpm/100, 2) as cpm,
         round(ctr, 2) as ctr,
         round(romi, 2) as romi,
         case
             when cpm > 0 then round(cpm::numeric / prev_cpm - 1, 2) * 100
             when cpm = 0 and prev_cpm > 0 then -100
         end as "cpm_diff_%",
         case
             when prev_ctr > 0 then round(ctr::numeric / prev_ctr - 1, 2) * 100
             when ctr = 0 and prev_ctr > 0 then -100
         end as "ctr_diff_%",
         case
             when prev_romi > 0 then round(romi::numeric / prev_romi - 1, 2) * 100
             when romi = 0 and prev_romi > 0 then -100
         end as "romi_diff_%"
     from shifted
     ```

